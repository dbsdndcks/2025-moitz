# 전체 아키텍처 개요 (Hexagonal Architecture)

이 다이어그램은 프로젝트의 헥사고날 아키텍처 전체 구조를 간단하게 보여줍니다.

```mermaid
graph TB
    subgraph "API Layer"
        API[API<br/>REST Controllers]
    end

    subgraph "Application Layer"
        APP[Application<br/>Services]
    end

    subgraph "Port Layer"
        PORT[Port<br/>Interfaces]
    end

    subgraph "Domain Layer"
        DOMAIN[Domain<br/>Models & Logic]
    end

    subgraph "Adapter Layer"
        ADAPTER[Adapter<br/>Implementations]
    end

    subgraph "Infrastructure Layer"
        INFRA[Infrastructure<br/>External APIs]
        DB[(Database<br/>MongoDB)]
    end

    API --> APP
    APP --> PORT
    APP --> DOMAIN
    PORT -.implements.-> ADAPTER
    ADAPTER --> INFRA
    ADAPTER --> DB
    DOMAIN -.uses.-> DB

    style API fill:#e1f5ff
    style APP fill:#fff4e1
    style PORT fill:#fff9c4
    style DOMAIN fill:#e8f5e9
    style ADAPTER fill:#ffe0b2
    style INFRA fill:#f3e5f5
    style DB fill:#f3e5f5
```

## 아키텍처 레이어 설명

### API
REST Controllers - 외부 HTTP 요청 처리

### Application
비즈니스 로직 조율 및 유스케이스 구현

### Port
외부 시스템과의 추상화된 인터페이스

### Domain
핵심 비즈니스 로직 및 도메인 모델

### Adapter
Port 인터페이스의 실제 구현체

### Infrastructure
외부 API 클라이언트 및 데이터베이스 연결

## 의존성 역전 원칙

```
Application (고수준)
    ↓ 의존
Port (인터페이스)
    ↑ 구현
Adapter (저수준)
```