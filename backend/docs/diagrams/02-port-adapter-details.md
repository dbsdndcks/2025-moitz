# Port-Adapter 상세 구조

이 다이어그램은 모든 Port 인터페이스와 Adapter 구현체의 관계를 상세히 보여줍니다.

```mermaid
graph LR
    subgraph "Application Services"
        RecService[RecommendationService]
        SubwayService[SubwayStationService]
        VoteService[VoteService]
    end

    subgraph "Ports (Interfaces)"
        direction TB

        subgraph "Route & Path Ports"
            RouteFinder["&lt;&lt;interface&gt;&gt;<br/>RouteFinder<br/>---<br/>+ findRoutes(...): List&lt;Route&gt;<br/>+ findCourses(...): List&lt;Course&gt;"]
        end

        subgraph "Place Recommendation Ports"
            PlaceRec["&lt;&lt;interface&gt;&gt;<br/>PlaceRecommender<br/>---<br/>+ recommendPlaces(...): Map"]
            AsyncPlaceRec["&lt;&lt;interface&gt;&gt;<br/>AsyncPlaceRecommender<br/>---<br/>+ recommendPlacesAsync(...): Mono&lt;Map&gt;"]
        end

        subgraph "Location Recommendation Port"
            LocRec["&lt;&lt;interface&gt;&gt;<br/>LocationRecommender<br/>---<br/>+ recommendLocations(...): Response"]
        end

        subgraph "Place Search Port"
            PlaceFinder["&lt;&lt;interface&gt;&gt;<br/>PlaceFinder<br/>---<br/>+ findPlaceByName(name): Place<br/>+ findPlacesByNames(...): List&lt;Place&gt;"]
        end

        subgraph "Data Loading Port"
            MapLoader["&lt;&lt;interface&gt;&gt;<br/>SubwayMapLoader<br/>---<br/>+ loadRawRoutes(): List&lt;RawRouteInfo&gt;"]
        end
    end

    subgraph "Adapters (Implementations)"
        direction TB

        subgraph "Application Layer Adapters"
            RouteAdapter["SubwayRouteFinderAdapter<br/>---<br/>사용: SubwayStationService<br/>사용: SubwayEdgeService<br/>알고리즘: Dijkstra"]
        end

        subgraph "Infrastructure Layer Adapters"
            PlaceAdapter["PlaceRecommenderAdapter<br/>---<br/>API: Kakao Map (동기)<br/>검색 반경: 800m<br/>도보 시간 계산"]

            AsyncPlaceAdapter["PlaceRecommenderAsyncAdapter<br/>---<br/>API: Kakao Map (비동기)<br/>병렬 처리: Flux/Mono<br/>Retry: 2회"]

            LocAdapter["LocationRecommenderAdapter<br/>---<br/>Primary: Google Gemini<br/>Fallback: Perplexity<br/>CircuitBreaker: 2개<br/>Retry: 2회 (RetryableApiException)"]

            FinderAdapter["KakaoPlaceFinderAdapter<br/>---<br/>API: Kakao Map<br/>장소명 → 좌표 변환"]

            LoaderAdapter["SubwayMapLoaderAdapter<br/>---<br/>API: Open API (공공)<br/>CSV 파일 로드"]
        end
    end

    subgraph "External Dependencies"
        direction LR

        KakaoSync[Kakao Map API<br/>동기 RestClient]
        KakaoAsync[Kakao Map API<br/>비동기 WebClient]
        Gemini[Google Gemini API<br/>AI 지역 추천]
        Perplexity[Perplexity API<br/>AI 백업]
        OpenAPI[Open API<br/>지하철 데이터]
    end

    %% Services to Ports
    RecService --> RouteFinder
    RecService --> PlaceRec
    RecService --> AsyncPlaceRec
    RecService --> LocRec
    RecService --> PlaceFinder

    SubwayService --> MapLoader

    %% Ports to Adapters (implements)
    RouteFinder -.implements.-> RouteAdapter
    PlaceRec -.implements.-> PlaceAdapter
    AsyncPlaceRec -.implements.-> AsyncPlaceAdapter
    LocRec -.implements.-> LocAdapter
    PlaceFinder -.implements.-> FinderAdapter
    MapLoader -.implements.-> LoaderAdapter

    %% Adapters to External APIs
    RouteAdapter --> SubwayService
    PlaceAdapter --> KakaoSync
    AsyncPlaceAdapter --> KakaoAsync
    LocAdapter --> Gemini
    LocAdapter -.fallback.-> Perplexity
    FinderAdapter --> KakaoSync
    LoaderAdapter --> OpenAPI

    style RecService fill:#fff4e1
    style SubwayService fill:#fff4e1
    style VoteService fill:#fff4e1

    style RouteFinder fill:#fff9c4
    style PlaceRec fill:#fff9c4
    style AsyncPlaceRec fill:#fff9c4
    style LocRec fill:#fff9c4
    style PlaceFinder fill:#fff9c4
    style MapLoader fill:#fff9c4

    style RouteAdapter fill:#e1f5ff
    style PlaceAdapter fill:#e1f5ff
    style AsyncPlaceAdapter fill:#e1f5ff
    style LocAdapter fill:#e1f5ff
    style FinderAdapter fill:#e1f5ff
    style LoaderAdapter fill:#e1f5ff

    style KakaoSync fill:#f3e5f5
    style KakaoAsync fill:#f3e5f5
    style Gemini fill:#f3e5f5
    style Perplexity fill:#f3e5f5
    style OpenAPI fill:#f3e5f5
```

## Port별 상세 설명

### 1. RouteFinder Port
```java
public interface RouteFinder {
    List<Route> findRoutes(List<StartEndPair> placePairs);
    List<Course> findCourses(List<StartEndPair> placePairs);
}
```
- **목적**: 두 지점 간 최적 경로 찾기
- **구현체**: `SubwayRouteFinderAdapter` (application layer)
- **알고리즘**: Dijkstra 최단 경로
- **의존성**: SubwayStationService, SubwayEdgeService (내부)

### 2. PlaceRecommender Port (동기)
```java
public interface PlaceRecommender {
    Map<Place, CategorizedRecommendedPlaces> recommendPlaces(
        List<Place> targetPlaces,
        List<RecommendCondition> requirements
    );
}
```
- **목적**: 특정 지점 주변 장소 추천 (동기)
- **구현체**: `PlaceRecommenderAdapter`
- **API**: Kakao Map API (RestClient)
- **특징**: 카테고리별 검색, 이미지 URL 추가, 도보 시간 계산

### 3. AsyncPlaceRecommender Port (비동기)
```java
public interface AsyncPlaceRecommender {
    Mono<Map<Place, CategorizedRecommendedPlaces>> recommendPlacesAsync(
        List<Place> targetPlaces,
        List<RecommendCondition> requirements
    );
}
```
- **목적**: 특정 지점 주변 장소 추천 (비동기)
- **구현체**: `PlaceRecommenderAsyncAdapter`
- **API**: Kakao Map API (WebClient)
- **특징**: Reactor 기반 병렬 처리, retry(2) 적용

### 4. LocationRecommender Port
```java
public interface LocationRecommender {
    RecommendedLocationsResponse recommendLocations(
        List<String> startingPlaces,
        List<String> candidatePlaces,
        List<RecommendCondition> requirements
    );
}
```
- **목적**: AI 기반 지역 추천
- **구현체**: `LocationRecommenderAdapter`
- **Primary API**: Google Gemini
- **Fallback API**: Perplexity
- **복원력**:
  - CircuitBreaker 2개 (normal, retryable)
  - Spring Retry (최대 2회)
  - Fallback 체인 (Gemini → Perplexity)

### 5. PlaceFinder Port
```java
public interface PlaceFinder {
    Place findPlaceByName(String placeName);
    List<Place> findPlacesByNames(List<String> placeNames);
}
```
- **목적**: 장소명으로 좌표 검색
- **구현체**: `KakaoPlaceFinderAdapter`
- **API**: Kakao Map API

### 6. SubwayMapLoader Port
```java
public interface SubwayMapLoader {
    List<RawRouteInfo> loadRawRoutes();
}
```
- **목적**: 지하철 노선 데이터 로드
- **구현체**: `SubwayMapLoaderAdapter`
- **데이터 소스**: CSV 파일 + Open API (공공 데이터)

## Adapter 계층 분리

### Application Layer Adapter
- **SubwayRouteFinderAdapter**: 내부 서비스만 사용 (외부 API 없음)
- 도메인 로직에 가까운 어댑터

### Infrastructure Layer Adapters
- **PlaceRecommenderAdapter**: Kakao API (동기)
- **PlaceRecommenderAsyncAdapter**: Kakao API (비동기)
- **LocationRecommenderAdapter**: Gemini + Perplexity API
- **KakaoPlaceFinderAdapter**: Kakao API
- **SubwayMapLoaderAdapter**: Open API
- 외부 시스템과의 실제 통신 담당

## 의존성 주입 (DI)

```java
@Service
public class RecommendationService {
    private final SubwayStationService subwayStationService;
    private final PlaceRecommender placeRecommender;  // @Qualifier로 선택
    private final AsyncPlaceRecommender asyncPlaceRecommender;
    private final LocationRecommender locationRecommender;
    private final RouteFinder routeFinder;  // @Qualifier로 선택

    public RecommendationService(
        SubwayStationService subwayStationService,
        @Qualifier("placeRecommenderAdapter") PlaceRecommender placeRecommender,
        @Qualifier("subwayRouteFinderAdapter") RouteFinder routeFinder,
        AsyncPlaceRecommender asyncPlaceRecommender,
        LocationRecommender locationRecommender
    ) {
        // ...
    }
}
```

## 테스트 용이성

Port-Adapter 패턴의 장점:
```java
// 프로덕션
@Component("placeRecommenderAdapter")
public class PlaceRecommenderAdapter implements PlaceRecommender { ... }

// 테스트
public class MockPlaceRecommender implements PlaceRecommender {
    @Override
    public Map<Place, CategorizedRecommendedPlaces> recommendPlaces(...) {
        return mockData;
    }
}
```

Port 인터페이스만 맞추면 Mock 구현체로 쉽게 교체 가능!