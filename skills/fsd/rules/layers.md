---
domain: fsd
topic: layers
triggers: [layer, 레이어, app, pages, widgets, features, entities, shared, 배치, 어디에]
status: published
---

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
├── auth/        # 토큰 관리, 현재 사용자 정보
├── format/      # formatDate, formatCurrency
├── validation/  # isValidEmailFormat 등 형식 검증
└── storage/     # localStorage, cookieStorage
```

세그먼트 이름은 *무엇을 하는가*를 드러내야 한다. `lib`, `utils`, `helpers` 같은 catch-all 이름은 쓰지 않는다.

### entities/
- **단일** 비즈니스 도메인 엔티티 (User, Post, Order 등)
- 비즈니스가 사용하는 용어 = 슬라이스 이름
- 추출 조건·DTO 처리·세그먼트 구성 예시 → [[entities]]

### features/
- **유즈케이스 / 시나리오** — 여러 엔티티를 오케스트레이션하거나 사용자 액션 흐름을 담는 슬라이스
- 두 곳 이상에서 반복되는 시나리오만 features로 추출 (한 페이지 한정이면 해당 page에)
- **포함**: 시나리오 UI(ui), 여러 엔티티 조합 query/mutation(model), 액션 흐름 로직(model), feature flag(config)

아래는 `ui/model/api` 패턴 예시 — 세그먼트 구성은 슬라이스마다 자유다.

```
features/
├── checkout/             # Order + Cart + Payment 오케스트레이션
│   ├── ui/               # 결제 폼·확인 화면
│   ├── api/              # 결제 요청 (유즈케이스 단위)
│   ├── model/            # 결제 흐름 로직
│   └── index.ts
└── comment-compose/
    └── ...
```

### widgets/
- 여러 페이지에서 재사용되는 **대형 독립 UI 블록**
- 한 페이지에서만 사용하고 대부분의 콘텐츠를 차지한다면 → page에 직접
- Remix 같은 중첩 라우팅에서는 라우터 블록 단위로 활용 가능

### pages/
- 보통 하나의 페이지 = 하나의 슬라이스
- 비슷한 페이지들(로그인/회원가입, 상품 목록/상세 등)은 *하나의 슬라이스*로 묶일 수 있음 — 1:1 매핑이 강제되지는 않음
- 재사용되지 않는 UI는 page 슬라이스 안에 넣어도 괜찮음
- **포함**: 페이지 UI, 로딩/에러 상태(ui), 데이터 패칭(api)

### 프레임워크 적응: Next.js App Router

Next.js에서는 `app/`(App Router)과 `pages/`(Pages Router)가 모두 라우팅 예약어다. 그래서 FSD `pages` 레이어는 **`views/`로 이름을 바꿔** 쓴다. `app/`은 FSD app 레이어(초기화) + 라우팅 진입점으로 공용한다. 레이어의 역할·import 방향은 표준과 동일하다.

| FSD 표준 | Next.js App Router |
|---------|-------------------|
| `app/` | `app/` (라우팅 진입점 + 앱 초기화 — `providers/`, `config/` 등) |
| `pages/` | `views/` |
| `widgets`·`features`·`entities`·`shared` | 동일 |

`get-query-client`처럼 페이지가 쓰는 인프라는 `app/`이 아니라 `shared/`에 둔다 — 그래야 `views`(pages)가 `app`을 역참조하지 않는다.

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

## Relations

- defined-by :: [[overview]]
- applies-to :: [[slices]]
- required-by :: [[slices]]
- required-by :: [[entities]]
- required-by :: [[cross-imports]]
- see-also :: [[segment-rules]]
- see-also :: [[public-api]]
