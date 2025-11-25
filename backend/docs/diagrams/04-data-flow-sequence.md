# 데이터 흐름 시퀀스 다이어그램

이 다이어그램은 추천 서비스의 실행 흐름을 시퀀스로 보여줍니다.

## 전체 추천 서비스 플로우

```mermaid
sequenceDiagram
    participant Client as HTTP Client
    participant Controller as RecommendationController
    participant Service as RecommendationService
    participant SubwayService as SubwayStationService
    participant LocPort as LocationRecommender<br/>(Port)
    participant LocAdapter as LocationRecommenderAdapter
    participant Gemini as Google Gemini API
    participant Perplexity as Perplexity API
    participant RoutePort as RouteFinder<br/>(Port)
    participant RouteAdapter as SubwayRouteFinderAdapter
    participant AsyncPort as AsyncPlaceRecommender<br/>(Port)
    participant AsyncAdapter as PlaceRecommenderAsyncAdapter
    participant Kakao as Kakao Map API
    participant Repo as MongoDB Repository

    Note over Client,Repo: 1. 요청 및 출발지 검증
    Client->>+Controller: POST /recommendations<br/>{startingPlaceNames: ["서울역", "시청역"],<br/>requirements: ["CAFE", "RESTAURANT"]}
    Controller->>+Service: recommendLocation(request)
    Service->>+SubwayService: findByNames(["서울역", "시청역"])
    SubwayService->>Repo: findByName("서울역")
    Repo-->>SubwayService: SubwayStation(서울역)
    SubwayService->>Repo: findByName("시청역")
    Repo-->>SubwayService: SubwayStation(시청역)
    SubwayService-->>-Service: List<SubwayStation>

    Note over Service,Perplexity: 2. AI 기반 지역 추천 (Gemini Primary)
    Service->>+LocPort: recommendLocations(출발지, 요구사항)
    LocPort->>+LocAdapter: recommendLocations(...)
    Note over LocAdapter: @Retryable<br/>CircuitBreaker 적용
    LocAdapter->>+Gemini: generateResponse(prompt)

    alt Gemini 성공
        Gemini-->>LocAdapter: {places: ["강남역", "홍대입구역"],<br/>descriptions: [...],<br/>reasons: [...]}
    else Gemini 실패 (Circuit Open / Exception)
        Gemini-->>LocAdapter: Exception
        Note over LocAdapter: Fallback 실행
        LocAdapter->>+Perplexity: generateResponse(prompt)
        Perplexity-->>-LocAdapter: {places: ["강남역", "홍대입구역"], ...}
    end

    LocAdapter-->>-LocPort: RecommendedLocationsResponse
    LocPort-->>-Service: RecommendedLocationsResponse

    Note over Service,RouteAdapter: 3. 경로 및 코스 찾기
    Service->>+RoutePort: findRoutes([서울역→강남역, 서울역→홍대입구역, ...])
    RoutePort->>+RouteAdapter: findRoutes(placePairs)
    loop 각 출발지-목적지 쌍
        RouteAdapter->>SubwayService: findShortestPath(start, end)
        SubwayService-->>RouteAdapter: SubwayPath (Dijkstra)
    end
    RouteAdapter-->>-RoutePort: List<Route>
    RoutePort-->>Service: List<Route>

    Service->>+RoutePort: findCourses([서울역→강남역, ...])
    RoutePort->>+RouteAdapter: findCourses(placePairs)
    RouteAdapter-->>-RoutePort: List<Course>
    RoutePort-->>-Service: List<Course>

    Note over Service: 4. 이동 시간 기준 필터링
    Note over Service: (평균 이동 시간 30분 이상 제거)
    Service->>Service: removePlacesBeyondRange(routes)

    Note over Service,Kakao: 5. 장소 추천 (비동기 병렬 처리)
    Service->>+AsyncPort: recommendPlacesAsync(["강남역", "홍대입구역"],<br/>["CAFE", "RESTAURANT"])
    AsyncPort->>+AsyncAdapter: recommendPlacesAsync(places, requirements)

    Note over AsyncAdapter,Kakao: Flux 기반 병렬 처리
    par Place: 강남역
        AsyncAdapter->>+Kakao: searchPlaces("카페", 강남역, 800m)
        Kakao-->>-AsyncAdapter: KakaoApiResponse (카페 목록)
        AsyncAdapter->>+Kakao: searchPlaces("식당", 강남역, 800m)
        Kakao-->>-AsyncAdapter: KakaoApiResponse (식당 목록)
    and Place: 홍대입구역
        AsyncAdapter->>+Kakao: searchPlaces("카페", 홍대입구역, 800m)
        Kakao-->>-AsyncAdapter: KakaoApiResponse (카페 목록)
        AsyncAdapter->>+Kakao: searchPlaces("식당", 홍대입구역, 800m)
        Kakao-->>-AsyncAdapter: KakaoApiResponse (식당 목록)
    end

    Note over AsyncAdapter: 도보 시간 계산<br/>이미지 URL 추가
    AsyncAdapter-->>-AsyncPort: Mono<Map<Place, CategorizedRecommendedPlaces>>
    AsyncPort-->>-Service: .block() → Map

    Note over Service,Repo: 6. 결과 변환 및 저장
    Service->>Service: RecommendationMapper.toRecommendation(...)
    Note over Service: Candidate 생성<br/>(destination, routes, courses,<br/>recommendedPlaces, description, reason)
    Service->>Service: Recommendation 생성<br/>(candidates를 평균 이동시간 순 정렬)

    Service->>+Repo: save(Result)
    Repo-->>-Service: ObjectId

    Service-->>-Controller: RecommendationCreateResponse(id)
    Controller-->>-Client: HTTP 201 Created<br/>{id: "507f1f77bcf86cd799439011"}
```

---

## 주요 처리 단계 상세 설명

### 1단계: 요청 및 출발지 검증
```java
// RecommendationService.java
List<SubwayStation> startingPlaces = getByNames(request.startingPlaceNames());
```
- MongoDB에서 출발지 역 정보 조회
- 존재하지 않는 역명 시 예외 발생

### 2단계: AI 기반 지역 추천
```java
RecommendedLocationsResponse locationResponse = locationRecommender.recommendLocations(
    request.startingPlaceNames(),
    candidatePlaces,
    request.requirements()
);
```

**복원력 흐름**:
```
Gemini 호출
  ↓
CircuitBreaker 1차 체크
  ↓
CircuitBreaker 2차 체크
  ↓
[성공] → 결과 반환
[실패] → Fallback → Perplexity 호출
[최대 재시도 초과] → @Recover → Perplexity 호출
```

### 3단계: 경로 및 코스 찾기
```java
List<Route> routes = routeFinder.findRoutes(allPairs);
List<Course> courses = routeFinder.findCourses(allPairs);
```

**경로 찾기 알고리즘**:
1. SubwayEdges에서 그래프 로드
2. Dijkstra 알고리즘으로 최단 경로 계산
3. SubwayPath → Route 도메인 모델 변환
4. Course 생성 (좌표 리스트)

### 4단계: 이동 시간 기준 필터링
```java
Map<Place, Routes> filteredPlaces = removePlacesBeyondRange(placeRoutes, generatedPlaces);
```

**필터링 조건**:
- 평균 이동 시간 계산: `Routes.getBestRecommendationTime()`
- 기준: 최선의 추천 시간 (가장 짧은 경로의 이동 시간)
- 제거: 30분 이상 소요되는 장소

### 5단계: 장소 추천 (비동기 병렬 처리)
```java
Map<Place, CategorizedRecommendedPlaces> recommendedPlaces =
    asyncPlaceRecommender.recommendPlacesAsync(generatedPlaces, conditions).block();
```

**병렬 처리 구조**:
```
Flux.fromIterable(places)  // [강남역, 홍대입구역, ...]
  .flatMap(place ->
    Flux.fromIterable(requirements)  // [CAFE, RESTAURANT, ...]
      .flatMap(req ->
        Flux.fromIterable(req.keywords())  // [카페, 커피숍, ...]
          .flatMap(keyword ->
            kakaoClient.searchPlacesAsync(keyword, place, 800m)
              .retry(2)
          )
      )
  )
```

**카테고리별 검색 키워드**:
- CAFE: ["카페", "커피숍", "디저트"]
- RESTAURANT: ["식당", "맛집", "음식점"]
- BAR: ["술집", "바", "펍"]
- STUDY_CAFE: ["스터디카페", "독서실"]
- PC_ROOM_KARAOKE: ["PC방", "노래방"]
- ACTIVITY: ["볼링", "당구", "스크린골프"]
- ENTERTAINMENT: ["영화관", "공연장", "방탈출"]

**도보 시간 계산**:
```java
int walkingTime = (int) (distance / 100 * 1.5);  // 100m당 1.5분
```

### 6단계: 결과 변환 및 저장
```java
Recommendation recommendation = recommendationMapper.toRecommendation(
    routes,
    courses,
    recommendedPlaces,
    locationResponse
);
```

**Recommendation 구조**:
```
Recommendation
├─ candidates: List<Candidate>  (평균 이동시간 순 정렬)
    ├─ destination: Place
    ├─ routes: Routes (출발지별 경로)
    ├─ courses: Courses (이동 코스)
    ├─ recommendedPlaces: CategorizedRecommendedPlaces
    ├─ description: String (AI 생성)
    ├─ reason: String (AI 생성)
    └─ votes: int (초기값 0)
```

**MongoDB 저장**:
```java
Result result = new Result(
    null,  // id는 MongoDB가 생성
    LocalDateTime.now(),
    recommendation,
    startingLocations,
    new HashSet<>(),  // votedLocations
    requirements
);
String id = recommendResultRepository.saveAndReturnId(result);
```

---

## 투표 기능 시퀀스

```mermaid
sequenceDiagram
    participant Client
    participant Controller as RecommendationController
    participant Service as VoteService
    participant Repo as RecommendResultRepository
    participant Mongo as MongoDB

    Client->>+Controller: POST /recommendations/{id}/votes<br/>{location: "강남역"}
    Controller->>+Service: vote(id, location)

    Service->>+Repo: findVotesByIdAndCandidate(id, "강남역")
    Repo->>+Mongo: Aggregation Query
    Mongo-->>-Repo: Optional<CandidateVote>
    Repo-->>-Service: Optional<CandidateVote>

    alt 이미 투표한 경우
        Service-->>Controller: throw AlreadyVotedException
        Controller-->>Client: HTTP 409 Conflict
    else 투표 가능
        Service->>+Repo: incrementVotesByIdAndCandidate(id, "강남역")
        Repo->>+Mongo: @Update { '$inc': { 'votes': 1 } }
        Mongo-->>-Repo: Updated
        Repo-->>-Service: void

        Service->>+Repo: findById(id)
        Repo->>Mongo: findById
        Mongo-->>Repo: Result
        Repo-->>-Service: Optional<Result>

        Service-->>-Controller: RecommendationResponse
        Controller-->>-Client: HTTP 200 OK<br/>{candidates: [...]}
    end
```

---

## 지하철 역 조회 시퀀스

```mermaid
sequenceDiagram
    participant Client
    participant Controller as SubwayStationController
    participant Service as SubwayStationService
    participant Repo as SubwayStationRepository
    participant Mongo as MongoDB

    Note over Client,Mongo: 역명 자동완성
    Client->>+Controller: GET /subway/stations?name=강남
    Controller->>+Service: findStationsStartingWith("강남")
    Service->>+Repo: findByNameStartingWith("강남")
    Repo->>+Mongo: Query: { name: /^강남/ }
    Mongo-->>-Repo: List<SubwayStationEntity>
    Repo-->>-Service: List<SubwayStationEntity>
    Service-->>-Controller: List<SubwayStation>
    Controller-->>-Client: HTTP 200 OK<br/>[{name: "강남역", ...},<br/> {name: "강남구청역", ...}]

    Note over Client,Mongo: 좌표 기반 주변 역 검색
    Client->>+Controller: GET /subway/stations/nearby?x=127.0&y=37.5&distance=1000
    Controller->>+Service: findStationsByDistance(x, y, distance)
    Service->>+Repo: findByPointNear(Point(127.0, 37.5), 1000m)
    Repo->>+Mongo: Geospatial Query
    Mongo-->>-Repo: List<SubwayStationEntity>
    Repo-->>-Service: List<SubwayStationEntity>
    Service-->>-Controller: List<SubwayStation>
    Controller-->>-Client: HTTP 200 OK<br/>[{name: "서울역", distance: 500m}, ...]
```

---

## 에러 처리 흐름

```mermaid
sequenceDiagram
    participant Service as RecommendationService
    participant Adapter as LocationRecommenderAdapter
    participant Gemini as Google Gemini API
    participant Perplexity as Perplexity API
    participant Handler as GlobalExceptionHandler
    participant Client

    Note over Service,Perplexity: 시나리오 1: Gemini Retryable 에러
    Service->>+Adapter: recommendLocations(...)
    Adapter->>+Gemini: generateResponse(...)
    Gemini-->>-Adapter: RetryableApiException<br/>(SERVER_UNAVAILABLE)
    Note over Adapter: Spring Retry 1차 시도
    Adapter->>+Gemini: generateResponse(...) [재시도 1]
    Gemini-->>-Adapter: RetryableApiException
    Note over Adapter: Spring Retry 2차 시도
    Adapter->>+Gemini: generateResponse(...) [재시도 2]
    Gemini-->>-Adapter: RetryableApiException
    Note over Adapter: @Recover 메서드 실행
    Adapter->>+Perplexity: generateResponse(...)
    Perplexity-->>-Adapter: RecommendedLocationsResponse
    Adapter-->>-Service: RecommendedLocationsResponse

    Note over Service,Perplexity: 시나리오 2: Gemini External 에러
    Service->>+Adapter: recommendLocations(...)
    Adapter->>+Gemini: generateResponse(...)
    Gemini-->>-Adapter: ExternalApiException<br/>(INVALID_API_KEY)
    Note over Adapter: CircuitBreaker Fallback 즉시 실행
    Adapter->>+Perplexity: generateResponse(...)
    Perplexity-->>-Adapter: RecommendedLocationsResponse
    Adapter-->>-Service: RecommendedLocationsResponse

    Note over Service,Client: 시나리오 3: 모든 API 실패
    Service->>+Adapter: recommendLocations(...)
    Adapter->>+Gemini: generateResponse(...)
    Gemini-->>-Adapter: ExternalApiException
    Adapter->>+Perplexity: generateResponse(...)
    Perplexity-->>-Adapter: ExternalApiException
    Adapter-->>-Service: throw ExternalApiException
    Service-->>Handler: throw ExternalApiException
    Handler-->>Client: HTTP 503 Service Unavailable<br/>{error: "AI 추천 서비스 일시 장애"}
```

---

## 성능 최적화 포인트

### 1. 비동기 병렬 처리
- **Before (동기)**: 장소 5개 × 조건 3개 × 키워드 3개 = 45회 순차 호출 (약 45초)
- **After (비동기)**: 45회 병렬 호출 (약 5초)
- **개선**: 9배 속도 향상

### 2. CircuitBreaker
- Gemini API 장애 시 즉시 Perplexity로 전환
- 연쇄 타임아웃 방지

### 3. Retry 전략
- RetryableApiException만 재시도 (일시적 오류)
- ExternalApiException은 즉시 Fallback (영구적 오류)

### 4. DTO 레이어 분리
- Port DTO (application.port.dto): Port 간 데이터 전달
- API DTO (ui.dto): HTTP 요청/응답
- Client DTO (infrastructure.client.*.dto): 외부 API 응답
- **효과**: 레이어 간 결합도 감소, 변경 영향 최소화

---

## 요약

이 시퀀스 다이어그램은 다음을 보여줍니다:

1. **요청부터 응답까지의 전체 흐름**
2. **Port-Adapter 패턴의 실제 동작**
3. **복원력 메커니즘의 작동 방식** (CircuitBreaker, Fallback, Retry)
4. **비동기 병렬 처리의 효율성**
5. **에러 처리 전략**

Port-Adapter 패턴 덕분에:
- Service는 외부 API 장애를 몰라도 됨
- Adapter가 복원력을 책임짐
- 외부 시스템 교체가 용이함