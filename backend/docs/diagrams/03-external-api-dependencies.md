# 외부 API 의존성 맵

이 다이어그램은 프로젝트가 의존하는 모든 외부 API와 그 복원력 메커니즘을 보여줍니다.

```mermaid
graph TB
    subgraph "Application Layer"
        RecService[RecommendationService]
    end

    subgraph "Adapters with Resilience Patterns"

        subgraph "LocationRecommenderAdapter"
            LocAdapter[LocationRecommenderAdapter]
            CB1[CircuitBreaker:<br/>geminiBreaker]
            CB2[CircuitBreaker:<br/>geminiRetryableBreaker]
            Retry1[Spring Retry<br/>maxAttempts: 2]
            FallbackLogic[Fallback Logic]
            Recover[Recovery Method]
        end

        subgraph "PlaceRecommenderAdapter (Sync)"
            PlaceAdapter[PlaceRecommenderAdapter<br/>---<br/>검색 반경: 800m<br/>도보 시간 계산]
        end

        subgraph "PlaceRecommenderAsyncAdapter"
            AsyncAdapter[PlaceRecommenderAsyncAdapter<br/>---<br/>Parallel: Flux/Mono<br/>retry(2)]
        end

        subgraph "KakaoPlaceFinderAdapter"
            FinderAdapter[KakaoPlaceFinderAdapter]
        end

        subgraph "SubwayMapLoaderAdapter"
            LoaderAdapter[SubwayMapLoaderAdapter<br/>---<br/>CSV + API 조합]
        end
    end

    subgraph "External APIs"

        subgraph "Kakao API"
            KakaoMap[Kakao Map API<br/>---<br/>장소 검색<br/>좌표 변환<br/>이미지 검색]
            KakaoRest[RestClient<br/>Connection: 3s<br/>Read: 5s]
            KakaoWeb[WebClient<br/>Timeout: 5s]
        end

        subgraph "AI APIs"
            Gemini[Google Gemini API<br/>---<br/>지역 추천 (Primary)<br/>AI 기반 의사결정<br/>설명/이유 생성]
            Perplexity[Perplexity API<br/>---<br/>지역 추천 (Fallback)<br/>Gemini 장애 시 대체<br/>Timeout: 20s]
        end

        subgraph "Public Data"
            OpenAPI[Open API<br/>공공 데이터<br/>---<br/>지하철 노선 정보<br/>최단 시간 경로<br/>최소 환승 경로]
        end
    end

    subgraph "Error Types"
        External[ExternalApiException<br/>---<br/>재시도 불가능<br/>INVALID_API_KEY<br/>EXCEEDED_QUOTA<br/>INVALID_RESPONSE]

        Retryable[RetryableApiException<br/>---<br/>재시도 가능<br/>SERVER_UNAVAILABLE<br/>TIMEOUT<br/>NETWORK_ERROR]
    end

    %% Service to Adapters
    RecService --> LocAdapter
    RecService --> PlaceAdapter
    RecService --> AsyncAdapter
    RecService --> FinderAdapter

    %% LocationRecommenderAdapter Flow
    LocAdapter --> Retry1
    Retry1 --> CB1
    CB1 --> CB2
    CB2 --> Gemini
    CB2 -.CircuitOpen/Exception.-> FallbackLogic
    FallbackLogic --> Perplexity
    Retry1 -.Max Attempts Exceeded.-> Recover
    Recover --> Perplexity

    %% PlaceRecommenderAdapter
    PlaceAdapter --> KakaoRest
    KakaoRest --> KakaoMap

    %% AsyncPlaceRecommenderAdapter
    AsyncAdapter --> KakaoWeb
    KakaoWeb --> KakaoMap

    %% KakaoPlaceFinderAdapter
    FinderAdapter --> KakaoRest

    %% SubwayMapLoaderAdapter
    LoaderAdapter --> OpenAPI

    %% Error handling
    Gemini -.throws.-> Retryable
    Gemini -.throws.-> External
    Perplexity -.throws.-> External
    KakaoMap -.throws.-> External
    OpenAPI -.throws.-> Retryable

    style RecService fill:#fff4e1
    style LocAdapter fill:#e1f5ff
    style PlaceAdapter fill:#e1f5ff
    style AsyncAdapter fill:#e1f5ff
    style FinderAdapter fill:#e1f5ff
    style LoaderAdapter fill:#e1f5ff

    style CB1 fill:#ffebee
    style CB2 fill:#ffebee
    style Retry1 fill:#fff3e0
    style FallbackLogic fill:#e8f5e9
    style Recover fill:#e8f5e9

    style Gemini fill:#e3f2fd
    style Perplexity fill:#f3e5f5
    style KakaoMap fill:#fff9c4
    style OpenAPI fill:#e0f2f1

    style External fill:#ffcdd2
    style Retryable fill:#ffe0b2
```

## 외부 API별 상세 정보

### 1. Google Gemini API

**사용 목적**: AI 기반 지역 추천
- 출발지와 요구사항을 기반으로 최적의 만남 장소 추천
- 추천 이유와 설명 생성
- 구조화된 JSON 응답 (place, description, reason)

**복원력 메커니즘**:
```java
@Retryable(retryFor = RetryableApiException.class, maxAttempts = 2)
public RecommendedLocationsResponse recommendLocations(...) {
    Supplier<RecommendedLocationsResponse> geminiCall =
        () -> geminiClient.generateResponse(...);

    return Decorators.ofSupplier(geminiCall)
        .withCircuitBreaker(geminiBreaker)           // 1차 방어
        .withCircuitBreaker(geminiRetryableBreaker)  // 2차 방어
        .withFallback(                                // Fallback
            List.of(ExternalApiException.class, CallNotPermittedException.class),
            throwable -> perplexityClient.generateResponse(...)
        )
        .decorate()
        .get();
}

@Recover  // 최종 실패 시
public RecommendedLocationsResponse recoverRecommendedLocations(...) {
    return perplexityClient.generateResponse(...);
}
```

**실패 시나리오**:
1. **정상**: Gemini → 성공 ✓
2. **Retryable 오류**: Gemini 실패 → 재시도(1회) → 재시도(2회) → Recover → Perplexity
3. **CircuitBreaker Open**: Gemini 호출 차단 → Fallback → Perplexity
4. **External 오류**: Gemini 실패 → 즉시 Fallback → Perplexity

**에러 타입**:
- `RetryableApiException.GEMINI_API_SERVER_UNAVAILABLE` → 재시도
- `ExternalApiException.INVALID_GEMINI_API_KEY` → 재시도 안함, 즉시 Fallback
- `ExternalApiException.EXCEEDED_GEMINI_API_TOKEN_QUOTA` → 재시도 안함

---

### 2. Perplexity API

**사용 목적**: Gemini의 Fallback AI
- Gemini 실패 시 대체 AI 모델
- 동일한 지역 추천 기능
- 더 긴 타임아웃 (20초)

**설정**:
```java
@Bean
public WebClient perplexityWebClient() {
    return WebClient.builder()
        .baseUrl("https://api.perplexity.ai")
        .clientConnector(new ReactorClientHttpConnector(httpClient(20)))  // 20초 timeout
        .defaultHeader("Authorization", "Bearer " + perplexityApiKey)
        .build();
}
```

**특징**:
- CircuitBreaker 없음 (최후의 방어선)
- 실패 시 예외를 그대로 throw

---

### 3. Kakao Map API

**사용 목적**:
1. **장소 검색** (PlaceRecommender)
   - 특정 좌표 주변 카페, 식당 등 검색
   - 카테고리별 검색 지원
   - 반경 800m 내 검색

2. **좌표 변환** (PlaceFinder)
   - 장소명 → 위경도 좌표

3. **이미지 검색**
   - 장소 대표 이미지 URL 제공

**동기 버전** (RestClient):
```java
@Bean
public RestClient kakaoRestClient() {
    return RestClient.builder()
        .baseUrl("https://dapi.kakao.com/v2")
        .requestFactory(simpleClientHttpRequestFactory())  // 3초 연결, 5초 읽기
        .build();
}
```
- 사용: PlaceRecommenderAdapter, KakaoPlaceFinderAdapter
- 타임아웃: 연결 3초, 읽기 5초

**비동기 버전** (WebClient):
```java
@Bean
public WebClient kakaoWebClient() {
    return WebClient.builder()
        .baseUrl("https://dapi.kakao.com/v2")
        .clientConnector(new ReactorClientHttpConnector(httpClient(5)))  // 5초 timeout
        .build();
}
```
- 사용: PlaceRecommenderAsyncAdapter
- Reactor 기반 비동기 처리
- retry(2) 적용
- 병렬 처리: Place × Condition × Keyword

**API 엔드포인트**:
- `/v2/local/search/keyword.json` - 키워드 검색
- `/v2/search/image` - 이미지 검색

**에러 처리**:
- 빈 응답 검증
- ExternalApiException.INVALID_KAKAO_MAP_API_RESPONSE

---

### 4. Open API (공공 데이터)

**사용 목적**: 지하철 노선 정보 로드
- 역 간 경로 정보
- 소요 시간 정보
- CSV 파일 + API 조합

**API 엔드포인트**:
- 최단 시간 경로 조회
- 최소 환승 경로 조회

**데이터 소스**:
1. CSV 파일: `route-for-station-map.csv`
2. Open API: 경로 상세 정보

**에러 처리**:
- `RetryableApiException.OPEN_API_SERVER_UNAVAILABLE`

---

## API 호출 빈도 및 비용

### Gemini API
- **호출 시점**: 추천 요청마다 1회
- **비용 영향**: 높음 (AI 모델)
- **최적화**: CircuitBreaker로 연쇄 호출 방지

### Kakao Map API (비동기)
- **호출 시점**: 추천 요청마다 N회 (Place × Condition × Keyword)
- **병렬 처리**: 최대 수십 개 동시 호출
- **비용 영향**: 높음 (호출 수 많음)
- **최적화**:
  - 비동기 병렬 처리로 속도 향상
  - retry(2)로 실패율 감소

### Perplexity API
- **호출 시점**: Gemini 실패 시에만
- **비용 영향**: 낮음 (Fallback)

### Open API
- **호출 시점**: 서버 초기화 시 1회 (데이터 로드)
- **비용 영향**: 없음 (공공 데이터)

---

## 복원력 패턴 요약

| 패턴 | 적용 위치 | 목적 |
|------|----------|------|
| **CircuitBreaker** | LocationRecommenderAdapter (Gemini) | 연쇄 장애 방지 |
| **Fallback** | LocationRecommenderAdapter | Gemini → Perplexity 전환 |
| **Retry** | LocationRecommenderAdapter, AsyncAdapter | 일시적 오류 재시도 |
| **Timeout** | 모든 HTTP 클라이언트 | 무한 대기 방지 |
| **Parallel** | AsyncPlaceRecommenderAdapter | 응답 속도 향상 |

---

## 에러 분류 전략

### ExternalApiException (재시도 불가)
- API 키 오류
- 쿼터 초과
- 잘못된 응답 형식
- **대응**: 즉시 Fallback 또는 실패 처리

### RetryableApiException (재시도 가능)
- 서버 일시 장애
- 타임아웃
- 네트워크 오류
- **대응**: Spring Retry로 자동 재시도

---

## 의존성 격리 효과

Port-Adapter 패턴 덕분에:

✅ **API 교체 용이**
```java
// Gemini → GPT-4로 교체
@Component
public class GptLocationRecommenderAdapter implements LocationRecommender {
    // LocationRecommender 인터페이스만 구현하면 됨
}
```

✅ **테스트 간소화**
```java
// Mock Adapter로 교체
public class MockLocationRecommender implements LocationRecommender {
    public RecommendedLocationsResponse recommendLocations(...) {
        return mockResponse;
    }
}
```

✅ **외부 API 장애 격리**
- Gemini 장애 → Perplexity로 자동 전환
- Kakao API 장애 → 빈 결과 반환 (서비스는 계속 동작)

✅ **복원력 메커니즘 적용**
- Adapter 레벨에서 CircuitBreaker, Retry, Fallback 적용
- Application Service는 복원력 로직을 몰라도 됨