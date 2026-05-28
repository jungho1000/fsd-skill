# FSD 레이어 규칙

## 레이어 계층과 Import 규칙

```
app       ┐
pages     │  위 레이어는 아래를 import 가능
widgets   │  아래 레이어는 위를 import 불가
features  │
entities  │
shared    ┘
```

**핵심**: `features/`의 파일은 `entities/`, `shared/`를 import할 수 있지만 `pages/`, `widgets/`를 import할 수 없다.

## 각 레이어 정의

### shared/
- 비즈니스 로직 없는 범용 코드
- 슬라이스 없음 → 세그먼트 직접 배치
- 내부 파일들끼리 자유롭게 import 가능
- **포함**: API 클라이언트, UI 킷, env 설정, i18n 설정, 도메인 무관 유틸 (포맷팅·검증·저장 등 목적이 드러나는 세그먼트로 분리)

```
shared/
├── api/         # API 클라이언트, 공통 request 함수
├── ui/          # 디자인 시스템, 공통 컴포넌트
├── config/      # 환경 변수, 글로벌 feature flag
├── routes/      # 라우트 상수
├── i18n/        # 번역 설정
├── format/      # formatDate, formatCurrency
├── validation/  # isValidEmailFormat 등 형식 검증
└── storage/     # localStorage, cookieStorage
```

세그먼트 이름은 *무엇을 하는가*를 드러내야 한다. `lib`, `utils`, `helpers` 같은 catch-all 이름은 쓰지 않는다.

### entities/
- 비즈니스 도메인 모델 (User, Post, Order 등)
- 비즈니스가 사용하는 용어 = 슬라이스 이름
- **포함**: 데이터 스토어(model), 유효성 검증 스키마(model), API 요청(api), UI 표현(ui)

```
entities/
├── user/
│   ├── model/    # 스토어, 인터페이스
│   ├── api/      # 사용자 관련 API
│   ├── ui/       # 사용자 표현 컴포넌트
│   └── index.ts  # Public API
└── product/
    └── ...
```

### features/
- 사용자가 실제로 수행하는 행동/기능
- **재사용되는 기능**만 features에 넣는다 (한 페이지에서만 쓰이면 해당 page에)
- **포함**: 폼 UI(ui), API 호출(api), 상태/검증(model), feature flag(config)

```
features/
├── auth/
│   ├── ui/       # 로그인 폼
│   ├── api/      # 로그인 요청
│   ├── model/    # 인증 상태
│   └── index.ts
└── comment/
    └── ...
```

### widgets/
- 여러 페이지에서 재사용되는 **대형 독립 UI 블록**
- 한 페이지에서만 사용하고 대부분의 콘텐츠를 차지한다면 → page에 직접
- Remix 같은 중첩 라우팅에서는 라우터 블록 단위로 활용 가능

### pages/
- 하나의 페이지 = 보통 하나의 슬라이스
- 재사용되지 않는 UI는 page 슬라이스 안에 넣어도 괜찮음
- **포함**: 페이지 UI, 로딩/에러 상태(ui), 데이터 패칭(api)

### app/
- 앱 전체를 초기화하는 코드
- 슬라이스 없음 → 세그먼트 직접 배치

```
app/
├── routes/      # 라우터 설정
├── store/       # 글로벌 스토어 설정
├── styles/      # 글로벌 스타일
└── entrypoint/  # 앱 진입점
```

## 자주 하는 실수

| 상황 | 잘못된 위치 | 올바른 위치 |
|------|------------|------------|
| 모든 페이지에서 쓰는 헤더 | pages/header | widgets/header |
| 한 페이지에서만 쓰는 사이드바 | widgets/sidebar | pages/{page}/ui/Sidebar |
| 로그인 요청 함수 | entities/user/api | shared/api 또는 pages/login/api |
| 앱 전체 설정값 | entities/config | shared/config 또는 app/ |
