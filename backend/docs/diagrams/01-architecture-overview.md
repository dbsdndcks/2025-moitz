# 전체 아키텍처 개요 (Hexagonal Architecture)

이 다이어그램은 프로젝트의 헥사고날 아키텍처 전체 구조를 보여줍니다.

```mermaid
graph TB
    subgraph "External Layer (Driving Side)"
        HTTP[HTTP Requests<br/>RecommendationController<br/>SubwayStationController]
    end

    subgraph "Application Layer"
        Service[Application Services<br/>RecommendationService<br/>VoteService<br/>SubwayStationService]

        subgraph "Ports (Interfaces)"
            RouteFinder[RouteFinder<br/>Port]
            PlaceRec[PlaceRecommender<br/>Port]
            AsyncPlaceRec[AsyncPlaceRecommender<br/>Port]
            LocRec[LocationRecommender<br/>Port]
            PlaceFinder[PlaceFinder<br/>Port]
            MapLoader[SubwayMapLoader<br/>Port]
        end
    end

    subgraph "Domain Layer"
        Domain[Domain Models<br/>Place, Route, Path<br/>Candidate, Recommendation<br/>SubwayStation, Course]
        Repo[Repository Interfaces<br/>SubwayStationRepository<br/>RecommendResultRepository]
    end

    subgraph "Infrastructure Layer (Driven Side)"
        subgraph "Adapters"
            RouteAdapter[SubwayRouteFinderAdapter]
            PlaceAdapter[PlaceRecommenderAdapter]
            AsyncPlaceAdapter[PlaceRecommenderAsyncAdapter]
            LocAdapter[LocationRecommenderAdapter]
            FinderAdapter[KakaoPlaceFinderAdapter]
            LoaderAdapter[SubwayMapLoaderAdapter]
        end

        subgraph "External Clients"
            KakaoSync[KakaoMapClient<br/>동기]
            KakaoAsync[KakaoMapAsyncClient<br/>비동기]
            Gemini[GoogleGeminiClient]
            Perplexity[PerplexityClient]
            OpenAPI[OpenApiClient]
        end

        subgraph "Database"
            MongoDB[(MongoDB<br/>Stations, Edges<br/>Results)]
        end
    end

    %% HTTP to Application
    HTTP --> Service

    %% Application to Ports
    Service --> RouteFinder
    Service --> PlaceRec
    Service --> AsyncPlaceRec
    Service --> LocRec
    Service --> PlaceFinder
    Service --> MapLoader

    %% Application uses Domain
    Service --> Domain
    Service --> Repo

    %% Ports to Adapters (Dependency Inversion)
    RouteFinder -.implements.-> RouteAdapter
    PlaceRec -.implements.-> PlaceAdapter
    AsyncPlaceRec -.implements.-> AsyncPlaceAdapter
    LocRec -.implements.-> LocAdapter
    PlaceFinder -.implements.-> FinderAdapter
    MapLoader -.implements.-> LoaderAdapter

    %% Adapters to External Clients
    RouteAdapter --> Service
    PlaceAdapter --> KakaoSync
    AsyncPlaceAdapter --> KakaoAsync
    LocAdapter --> Gemini
    LocAdapter -.fallback.-> Perplexity
    FinderAdapter --> KakaoSync
    LoaderAdapter --> OpenAPI

    %% Repository to DB
    Repo -.implements.-> MongoDB

    style HTTP fill:#e1f5ff
    style Service fill:#fff4e1
    style Domain fill:#e8f5e9
    style MongoDB fill:#f3e5f5
    style RouteFinder fill:#fff9c4
    style PlaceRec fill:#fff9c4
    style AsyncPlaceRec fill:#fff9c4
    style LocRec fill:#fff9c4
    style PlaceFinder fill:#fff9c4
    style MapLoader fill:#fff9c4
```

## 아키텍처 레이어 설명

### 1. External Layer (Driving Side / Inbound)
- **역할**: 외부에서 들어오는 요청을 처리
- **구성**: REST Controllers (RecommendationController, SubwayStationController)
- **의존성 방향**: Application Layer를 호출

### 2. Application Layer
- **역할**: 비즈니스 로직 조율 및 유스케이스 구현
- **구성**:
  - Application Services (RecommendationService, VoteService 등)
  - Ports (Interface) - 외부 시스템과의 추상화된 계약
- **의존성 방향**: Domain Layer와 Ports에 의존

### 3. Domain Layer
- **역할**: 핵심 비즈니스 로직 및 도메인 모델
- **구성**:
  - Domain Models (Place, Route, Candidate, Recommendation 등)
  - Repository Interfaces (인바운드 포트)
- **의존성 방향**: 외부에 의존하지 않음 (순수 도메인)

### 4. Infrastructure Layer (Driven Side / Outbound)
- **역할**: 외부 시스템과의 실제 연결 구현
- **구성**:
  - Adapters - Port 인터페이스 구현체
  - External Clients - 외부 API 호출 클라이언트
  - Database - MongoDB 영속성
- **의존성 방향**: Ports를 구현하고 Domain을 사용

## 의존성 역전 원칙 (Dependency Inversion Principle)

```
고수준 모듈 (Application/Domain)
    ↑ (의존)
인터페이스 (Ports)
    ↑ (구현)
저수준 모듈 (Infrastructure)
```

- Application은 Port 인터페이스에만 의존
- Infrastructure는 Port를 구현
- 따라서 외부 API 변경 시 Application 코드는 수정 불필요