# 📁 MOITZ 프로젝트 구조

## 프로젝트 개요
MOITZ는 사람들의 위치를 기반으로 취향, 목적, 거리를 고려한 약속 장소 추천 서비스입니다.

## 디렉토리 구조

```
2025-moitz/
├── README.md                    # 프로젝트 메인 문서
├── backend/                     # 백엔드 애플리케이션 (Spring Boot)
│   └── .gitkeep
├── docs/                        # 프로젝트 문서 및 리소스
│   ├── icons/                   # README 및 문서에 사용되는 아이콘
│   │   ├── JavaScript.png
│   │   ├── react.png
│   │   ├── springboot.png
│   │   ├── java.png
│   │   ├── docker.jpg
│   │   ├── aws.png
│   │   ├── git.png
│   │   ├── github.png
│   │   ├── figma.png
│   │   ├── intellij.png
│   │   ├── vscode.png
│   │   ├── notion.png
│   │   ├── web.png
│   │   └── 아키텍처.png
│   └── PROJECT_STRUCTURE.md     # 이 문서
└── frontend/                    # 프론트엔드 애플리케이션 (React + TypeScript)
    ├── .husky/                  # Git hooks 설정
    │   └── pre-commit          # pre-commit 훅
    ├── .vscode/                 # VS Code 설정
    │   └── settings.json
    ├── src/                     # 소스 코드
    │   └── App.tsx             # 메인 App 컴포넌트
    ├── .gitignore              # Git 무시 파일 목록
    ├── .prettierrc             # Prettier 설정
    ├── eslint.config.mjs       # ESLint 설정
    ├── index.html              # HTML 진입점
    ├── main.tsx                # React 진입점
    ├── package.json            # 프로젝트 의존성 및 스크립트
    ├── package-lock.json       # 의존성 잠금 파일
    ├── tsconfig.json           # TypeScript 설정
    └── webpack.config.js       # Webpack 빌드 설정
```

## 기술 스택

### 프론트엔드
- **언어**: TypeScript
- **프레임워크**: React 19.1.0
- **스타일링**: Emotion (CSS-in-JS)
- **빌드 도구**: Webpack 5
- **개발 도구**:
  - ESLint (코드 품질)
  - Prettier (코드 포맷팅)
  - Husky (Git hooks)
  - lint-staged (staged 파일 린팅)

### 백엔드
- **프레임워크**: Spring Boot
- **언어**: Java

## 주요 파일 설명

### 루트 디렉토리
- **README.md**: 프로젝트 전체 소개, 기능 설명, 팀원 정보

### 백엔드 (`/backend`)
- 현재 초기 설정 단계
- Spring Boot 기반의 REST API 서버 예정

### 프론트엔드 (`/frontend`)

#### 설정 파일
- **package.json**:
  - 프로젝트 의존성 관리
  - npm 스크립트 정의 (start, build, serve 등)
  - lint-staged 설정

- **webpack.config.js**:
  - Webpack 빌드 설정
  - 개발 서버 설정
  - TypeScript 로더 설정

- **tsconfig.json**:
  - TypeScript 컴파일러 옵션
  - 타입 체크 규칙

- **eslint.config.mjs**:
  - ESLint 규칙 설정
  - React 플러그인 설정

- **.prettierrc**:
  - 코드 포맷팅 규칙

#### 소스 코드
- **index.html**:
  - HTML 템플릿
  - React 앱이 마운트되는 진입점

- **main.tsx**:
  - React 애플리케이션 초기화
  - ReactDOM.render 진입점

- **src/App.tsx**:
  - 메인 App 컴포넌트
  - 라우팅 및 전역 상태 관리 예정

#### 개발 도구
- **.husky/pre-commit**:
  - 커밋 전 자동으로 실행되는 훅
  - lint-staged를 통한 코드 품질 검사

### 문서 (`/docs`)
- **icons/**: README 및 문서에 사용되는 기술 스택 아이콘
- **PROJECT_STRUCTURE.md**: 프로젝트 구조 문서 (현재 파일)

## 개발 워크플로우

### 프론트엔드 개발
```bash
cd frontend

# 의존성 설치
npm install

# 개발 서버 실행
npm start

# 프로덕션 빌드
npm run build

# 개발 빌드
npm run build:dev
```

### 코드 품질 관리
- 모든 커밋은 팀 Git 컨벤션 준수
- Pre-commit 훅을 통한 자동 Prettier 포맷팅
- 코드 리뷰 필수
- ESLint를 통한 코드 품질 검사

## 향후 확장 예정

### 백엔드
- Spring Boot 프로젝트 구조 설정
- REST API 엔드포인트 구현
- 데이터베이스 연동
- 외부 API 통합 (지도, 장소 검색 등)

### 프론트엔드
- 컴포넌트 디렉토리 구조화 (`src/components/`)
- 페이지 라우팅 설정 (`src/pages/`)
- 전역 상태 관리 (`src/store/` 또는 Context)
- API 통신 레이어 (`src/api/`)
- 유틸리티 함수 (`src/utils/`)
- 스타일 관리 (`src/styles/`)

## 아키텍처 다이어그램

```
┌─────────────────┐         ┌─────────────────┐
│                 │         │                 │
│   Frontend      │◄───────►│   Backend       │
│   (React)       │  HTTP   │ (Spring Boot)   │
│                 │         │                 │
└─────────────────┘         └────────┬────────┘
                                     │
                                     ▼
                            ┌─────────────────┐
                            │                 │
                            │   External APIs │
                            │  (Maps, Places) │
                            │                 │
                            └─────────────────┘
```

## 팀 구성

### 백엔드 (4명)
- 아이나, 시소, 줄리, 레몬

### 프론트엔드 (3명)
- 클레어, 메타, 헤일리

---

*최종 업데이트: 2025-11-13*
