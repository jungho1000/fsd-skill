# Feature-Sliced Design 빠른 학습 가이드

> 기술 역할 기반 구조에 익숙한 개발자를 위한 FSD 인식 전환 가이드

---

## 목차

1. [핵심 개념 한눈에 보기](#1-핵심-개념-한눈에-보기)
2. [사고의 전환: "기술 역할" → "도메인 + 역할"](#2-사고의-전환)
3. [레이어 계층 — 7개 폴더의 의미](#3-레이어-계층)
4. ["어디에 넣을까?" 결정 흐름](#4-어디에-넣을까-결정-흐름)
5. [세그먼트 — 코드의 목적으로 분류](#5-세그먼트)
6. [Import 규칙](#6-import-규칙)
7. [Public API 패턴](#7-public-api-패턴)
8. [흔한 실수와 해결책](#8-흔한-실수와-해결책)
9. [실전 예시: 기존 구조 → FSD 변환](#9-실전-예시)

---

## 1. 핵심 개념 한눈에 보기

FSD는 프론트엔드 코드를 **"이 코드가 하는 일이 무엇인가"** 기준으로 일관되게 분류하는 아키텍처다.  
레이어·슬라이스는 **"앱의 어느 도메인을 위한 코드인가"**,  
세그먼트는 **"그 안에서 화면 표현인가, 상태/로직인가, 백엔드 통신인가"** 로 나눈다.  
기술 타입(컴포넌트, 훅, 함수)이 아닌 **목적**이 분류 기준이다.

```
src/
├── app/       ← 앱 전체 초기화 (라우터, 프로바이더, 글로벌 스타일)
├── pages/     ← 각 페이지/화면
├── widgets/   ← 여러 페이지에서 재사용되는 독립적인 대형 UI 블록
├── features/  ← 재사용되는 비즈니스 기능 (사용자 행동)
├── entities/  ← 비즈니스 도메인 모델 (User, Product, Order...)
└── shared/    ← 도메인 무관한 공통 코드 (UI 킷, API 클라이언트, 유틸)
```

**핵심 3원칙**
- 하위 레이어는 상위 레이어를 참조할 수 없다
- 같은 레이어의 슬라이스끼리는 직접 참조할 수 없다
- 슬라이스 외부는 반드시 `index.ts`(Public API)를 통해서만 접근한다

---

## 2. 사고의 전환

### 기존 방식: "이 코드는 어떤 **종류**야?"

```
src/
├── components/    ← UserCard, ProductList, LoginForm, Header, Modal...
├── hooks/         ← useAuth, useCart, useProducts, useModal...
├── services/      ← authService, productService, cartService...
├── store/         ← authSlice, cartSlice, productSlice...
├── utils/         ← formatDate, validate, formatPrice...
├── types/         ← User, Product, Cart, Order...
└── pages/         ← LoginPage, HomePage, CartPage...
```

**이 구조의 한계**

프로젝트가 커지면 `components/`에 수백 개의 파일이 쌓인다.  
로그인 기능 하나를 이해하려면 `components/`, `hooks/`, `services/`, `store/`를 모두 뒤져야 한다.  
무언가를 삭제할 때 어디까지 지워야 하는지 파악하기 어렵다.

```
# 로그인 기능이 어디에 있지?
components/LoginForm.tsx      ← UI
components/LoginButton.tsx    ← UI
hooks/useAuth.ts              ← 상태/로직
services/authService.ts       ← API 호출
store/authSlice.ts            ← 스토어
types/User.ts                 ← 타입
```

---

### FSD 방식: "이 코드는 앱의 **어느 도메인**을 위한 거야? 그 안에서 **어떤 역할**이야?"

```
src/
├── features/
│   └── auth/          ← "로그인" 도메인
│       ├── ui/        ← LoginForm, LoginButton (역할: UI)
│       ├── model/     ← useAuth, authStore (역할: 상태/로직)
│       ├── api/       ← login, logout 요청 함수 (역할: API)
│       └── index.ts   ← 외부에 노출할 것만 export
├── entities/
│   └── user/          ← "사용자" 도메인 모델
│       ├── model/     ← User 타입, userStore
│       ├── ui/        ← UserAvatar, UserCard
│       └── index.ts
└── shared/
    ├── ui/            ← Button, Modal (도메인 무관한 컴포넌트)
    ├── api/           ← HTTP 클라이언트
    └── lib/           ← formatDate, formatCurrency 등 목적이 모호한 유틸
```

**"로그인 기능이 어디에 있지?"** → `features/auth/` 하나만 열면 된다.

---

### 인식 전환 요약

| 기존 구조 | FSD 구조 |
|----------|---------|
| **기술 타입**이 최상위 분류 | **도메인(비즈니스 의미)**이 최상위 분류 |
| `components/LoginForm.tsx` | `features/auth/ui/LoginForm.tsx` |
| `hooks/useAuth.ts` | `features/auth/model/useAuth.ts` |
| `services/authService.ts` | `features/auth/api/authService.ts` |
| 분류 질문: "이게 **무엇**이야?" (컴포넌트, 훅, 서비스) | 분류 질문: "이게 **무슨 일**을 해?" (도메인, 화면 표현, 상태 관리, 통신) |

> **주의 — 이 비교표에서 놓치기 쉬운 함정**
>
> `hooks/useAuth.ts → features/auth/model/useAuth.ts` 를 보고  
> "훅은 `model/`로 이름만 바꾸면 되는구나" 라고 생각하면 안 된다.
>
> **세그먼트도 기술 타입이 아닌 목적으로 분류한다.**
>
> - `components/` = "컴포넌트**이기 때문에**" (기술 타입 기준)  
> - `ui/` = "화면을 **그리는 역할**이기 때문에" (목적 기준)
>
> - `hooks/` = "훅**이기 때문에**" (기술 타입 기준)  
> - `model/` = "데이터·상태를 **다루는 역할**이기 때문에" (목적 기준)
>
> 따라서 같은 "훅"이라도 목적에 따라 다른 세그먼트에 들어간다:
>
> ```
> useAuthState()          → model/  # 인증 상태 관리 — 목적: 데이터/상태
> useScrollAnimation()    → ui/     # 스크롤 애니메이션 — 목적: 화면 표현
> ```
>
> 분류 기준이 "무엇이야?"에서 "무슨 일을 해?"로 바뀐 것은  
> 레이어, 슬라이스, **세그먼트 모두** 동일하게 적용된다.

---

## 3. 레이어 계층

```
높은 책임, 많은 의존성
┌─────────────┐
│    app      │  앱 전체 초기화. 라우터, 글로벌 스타일, context 프로바이더
├─────────────┤
│   pages     │  한 화면 = 한 슬라이스. 재사용 안 되는 UI는 여기 두면 됨
├─────────────┤
│  widgets    │  여러 페이지에서 재사용되는 독립적 대형 UI 블록
├─────────────┤
│  features   │  재사용되는 비즈니스 기능 (사용자가 수행하는 행동)
├─────────────┤
│  entities   │  비즈니스 도메인 모델 (User, Product, Order...)
├─────────────┤
│   shared    │  도메인 무관한 공통 코드. 어디서든 자유롭게 사용
└─────────────┘
낮은 책임, 적은 의존성
```

### 각 레이어 판단 기준

**shared** — "비즈니스를 모르는 코드"  
Button 컴포넌트, HTTP 클라이언트, formatDate 유틸, env 설정.  
특정 기능이나 도메인을 알아서는 안 된다.

**entities** — "비즈니스가 사용하는 용어 = 슬라이스 이름"  
User, Product, Order, Comment. 백엔드 도메인 모델과 그 UI 표현.  
여러 features에서 재사용되는 도메인 데이터와 로직.

**features** — "사용자가 실제로 하는 행동"  
로그인, 장바구니 담기, 댓글 작성, 좋아요.  
**단, 한 페이지에서만 쓰이면 features가 아니라 그 page 안에 둔다.**

**widgets** — "여러 페이지에서 재사용되는 대형 블록"  
Header, Sidebar, 댓글 위젯, 지도 위젯.  
한 페이지에서만 쓰이고 페이지 콘텐츠 대부분을 차지한다면 → page에 직접.

**pages** — "화면 단위 슬라이스"  
재사용 안 되는 UI는 page 안에 얼마든지 넣어도 괜찮다.

**app** — "앱 전체를 아우르는 것"  
라우터, 글로벌 스토어 초기화, 글로벌 스타일, 에러 바운더리.

### app과 shared의 특수성

```
# app, shared: 슬라이스 없이 세그먼트 바로 배치
app/
├── routes/
├── store/
└── styles/

shared/
├── api/
├── ui/
└── lib/

# 나머지 레이어: 레이어/슬라이스/세그먼트 3단 구조 필수
features/
└── auth/          ← 슬라이스
    ├── ui/        ← 세그먼트
    └── model/     ← 세그먼트
```

---

## 4. "어디에 넣을까?" 결정 흐름

아래 단계를 **순서대로** 확인한다. 해당하는 첫 번째 단계에서 결정한다.

**Step 1: 비즈니스 로직 없는 인프라인가?**

| 해당하는 코드 | 위치 |
|-------------|------|
| UI 컴포넌트 키트 (Button, Input) | `shared/ui/` |
| 유틸 함수 (formatDate, debounce) | `shared/lib/` |
| API 클라이언트, CRUD 함수 | `shared/api/` |
| 인증 토큰, 세션 관리 | `shared/auth/` |
| 환경 변수, 설정값 | `shared/config/` |

**Step 2: 앱 전역 초기화 코드인가?**
- 라우터, 글로벌 프로바이더, 글로벌 스타일 → `app/`

**Step 3: 한 페이지에서만 쓰이는가?**
- Yes → 그 `pages/` 슬라이스 안에 (추출 불필요, 중복도 허용)

**Step 4: 여러 페이지에서 재사용되는 사용자 액션인가?**
- Yes, 경계가 안정적 → `features/`
- 불확실 → `pages/`에 유지

**Step 5: 여러 곳에서 재사용되는 비즈니스 도메인 모델인가?**
- Yes, 경계가 안정적 → `entities/`
- 불확실 → 현재 슬라이스의 `model/`에 유지

> **황금 규칙**: 의심스러우면 `pages/`에 유지한다. 실제로 여러 곳에서 사용되고 경계가 명확해질 때만 추출한다.

### 자주 헷갈리는 판단

| 상황 | 잘못된 배치 | 올바른 배치 | 이유 |
|------|------------|------------|------|
| 모든 페이지에 있는 헤더 | `pages/header/` | `widgets/header/` | 여러 페이지에서 재사용 |
| 특정 페이지의 사이드바 | `widgets/sidebar/` | `pages/{name}/ui/Sidebar` | 한 페이지에서만 사용 |
| 로그인 API 함수 | `entities/user/api/` | `shared/api/` 또는 `pages/login/api/` | 비즈니스 엔티티 ≠ 인증 로직 |
| 인증 토큰 관리 | `entities/user/model/token` | `shared/auth/` | shared가 entities를 import 불가 |
| 전체에서 쓰는 toast | `features/toast/` | `shared/ui/toast/` | 비즈니스 행동이 아닌 UI 유틸 |

---

## 5. 세그먼트 — 코드의 목적으로 분류

세그먼트는 슬라이스 안에서 코드를 **목적(이 코드가 하는 일)**으로 분류하는 폴더다.

### 세그먼트 이름의 원칙: 타입이 아닌 목적

기존 `components/`, `hooks/`는 코드가 **무엇인지**(기술 타입)를 기준으로 나눴다.  
FSD 세그먼트는 그 코드가 **무슨 일을 하는지**(목적)를 기준으로 나눈다.

```
기존 구조 질문: "이건 React 컴포넌트야, 훅이야, 함수야?"
         ↓
FSD 세그먼트 질문: "이 코드의 역할이 화면 표현이야? 상태 관리야? 백엔드 통신이야?"
```

```
# ❌ 기술 타입 기준 — 무엇인지만 말함
features/auth/
├── components/   ← "React 컴포넌트들이 있음"
├── hooks/        ← "커스텀 훅들이 있음"
└── types/        ← "타입 정의들이 있음"
   → 로그인 폼을 찾으려면 세 폴더를 모두 확인해야 함

# ✅ 목적 기준 — 무슨 일을 하는지 말함
features/auth/
├── ui/           ← "화면에 그리는 것들이 있음"
├── model/        ← "상태와 비즈니스 로직이 있음"
└── api/          ← "백엔드와 통신하는 것들이 있음"
   → 로그인 폼은 ui/, 인증 상태는 model/로 바로 찾아감
```

### 핵심: 같은 기술 타입이어도 목적에 따라 다른 세그먼트

"훅이니까 `model/`" 처럼 기술 타입으로 세그먼트를 결정하면 안 된다.

```ts
// 같은 "커스텀 훅"이어도 목적에 따라 세그먼트가 다름
useAuthState()            → model/  // 인증 상태 관리        → 목적: 상태/비즈니스 로직
useScrollAnimation()      → ui/     // 스크롤 애니메이션 제어 → 목적: 화면 표현
useIntersectionObserver() → ui/     // 뷰포트 진입 감지       → 목적: 화면 표현

// 순수 함수도 목적에 따라 다름
validateUserPolicy(email) → model/      // 회사 이메일 정책 검증 → 비즈니스 규칙
isValidEmailFormat(email) → shared/lib/ // RFC 포맷 검사 → 도메인 무관 유틸
fetchProducts()           → api/        // 상품 목록 요청 → 백엔드 통신
```

> **포맷터는 `shared/lib/`에**  
> `formatDate`, `formatCurrency` 같은 포맷터는 UI 렌더링에 쓰이기도 하고 로깅에 쓰이기도 한다.  
> 목적을 단정짓기 어렵기 때문에 도메인 무관 유틸로 보고 `shared/lib/`에 두는 것이 적합하다.

### 표준 세그먼트 — 각각의 목적

| 세그먼트 | 목적 | 포함되는 코드 예시 |
|---------|------|-----------------|
| `ui/` | **화면에 그리는 것** | 컴포넌트, 스타일, 화면 렌더링 훅 (애니메이션, 인터섹션 등) |
| `model/` | **데이터·상태를 다루는 것** | 클라이언트 상태(Redux/Zustand), 서버 상태(TanStack Query), 도메인 타입, mapper |
| `api/` | **백엔드와 통신하는 것** | 요청 함수, DTO 타입 |
| `lib/` | **여러 세그먼트가 공유하는 내부 헬퍼** | `ui/`와 `model/` 양쪽에서 필요한 유틸, 특정 세그먼트에 귀속되지 않는 코드 |
| `config/` | **설정·플래그** | 설정값, feature flag |

> **`lib/` 사용 원칙**  
> 헬퍼 함수는 **사용하는 세그먼트 가까이 두는 것이 기본** (응집도 우선).  
> 한 세그먼트에서만 쓰인다면 그 세그먼트 안에, 여러 세그먼트에서 공유해야 할 때 비로소 `lib/`로 이동한다.

### model/ 내부 구성

`model/` 안에 도메인 개념이 하나라면 파일을 평탄하게 두고, 여럿이라면 도메인별 서브 디렉토리로 묶는다.

```
# 단일 모델 — 평탄한 구조
features/product/
└── model/
    ├── product.ts           # 도메인 타입
    ├── productMapper.ts     # mapper
    ├── product.queries.ts   # TanStack Query (서버 상태)
    └── useProduct.ts        # useQuery 훅

# 복수 모델 — 도메인별 서브 디렉토리
features/order-management/
└── model/
    ├── order/
    │   ├── order.ts
    │   ├── orderMapper.ts
    │   ├── order.queries.ts
    │   └── useOrder.ts
    └── order-status/
        └── orderStatusStore.ts   # Zustand (클라이언트 상태)
```

> **슬라이스 분리 신호**  
> `model/` 안에 서브 디렉토리가 여럿 필요할 만큼 모델이 많다면, 슬라이스를 분리해야 한다는 신호일 수 있다.  
> 먼저 슬라이스 분리를 검토하고, 분리 후에도 같은 슬라이스에 모델이 여럿이라면 그때 서브 디렉토리로 구성한다.

### 파일명 규칙: 도메인 기반 명명

세그먼트 이름이 목적을 기술하듯, **파일명도 도메인 기반**으로 짓는다.

```
❌ 기술적 역할명 — 안에 뭐가 있는지 알 수 없음
model/types.ts
model/utils.ts
model/helpers.ts

✅ 도메인 기반명 — 파일명 자체가 문서
model/user.ts           ← 사용자 도메인 타입 + 로직
model/order.ts          ← 주문 도메인 타입 + 로직
api/fetch-profile.ts    ← 프로필 페칭
api/update-settings.ts  ← 설정 업데이트
```

`types.ts`라는 파일이 있다면 "어떤 타입?"이라는 질문이 생긴다.  
`user.ts`라면 파일을 열기 전에 이미 내용을 알 수 있다.

### 세그먼트를 결정하는 질문

> "이 코드를 삭제하면 무엇이 영향을 받는가?"
>
> - 화면이 바뀐다 → `ui/`
> - 앱의 상태나 비즈니스 로직이 바뀐다 → `model/`
> - 백엔드와의 통신이 바뀐다 → `api/`
> - `ui/`와 `model/` 양쪽 모두 영향받는다 → `lib/` (단, 이 슬라이스 내부에서만 쓰인다면)

---

## 6. Import 규칙

### 레이어 Import 규칙

```
상위 레이어는 하위 레이어를 ✅ import 가능
하위 레이어는 상위 레이어를 ❌ import 불가

app     →  pages, widgets, features, entities, shared  ✅
pages   →  widgets, features, entities, shared          ✅
widgets →  features, entities, shared                   ✅
features →  entities, shared                            ✅
entities →  shared                                      ✅
shared  →  상위 레이어 import 불가 (shared 내부끼리는 자유)  —

# ❌ 이건 금지
// shared/api/client.ts
import { authToken } from "@/entities/user";  // 하위가 상위 참조 불가!
```

### 슬라이스 간 Import 규칙

같은 레이어의 슬라이스끼리는 직접 참조할 수 없다.

```ts
// ❌ 금지: 같은 레이어 cross-import
// features/cart/ui/Cart.tsx
import { ProductCard } from "@/features/product";  // features → features 금지!

// ✅ 해결: 상위 레이어(pages)에서 조합
// pages/shop/ui/ShopPage.tsx
import { Cart } from "@/features/cart";
import { ProductList } from "@/features/product";
```

### 슬라이스 내부 Import 규칙

```ts
// ✅ 내부에서 내부 참조: 상대 경로 + 전체 경로
// features/auth/ui/LoginForm.tsx
import { loginApi } from "../api/login";        // 상대 경로로 내부 직접 참조

// ✅ 외부 슬라이스 참조: 절대 경로 + index.ts
// features/auth/model/useAuth.ts
import { User } from "@/entities/user";          // 절대 경로로 Public API 참조

// ❌ 금지: 내부에서 자신의 index.ts 참조 (순환 import 발생)
// features/auth/ui/LoginForm.tsx
import { loginApi } from "../";                  // 순환 import!
```

---

## 7. Public API 패턴

모든 슬라이스는 `index.ts`를 통해 외부에 인터페이스를 제공한다.

```ts
// ✅ features/auth/index.ts — 외부에 노출할 것만 명시적으로 export
export { LoginForm } from "./ui/LoginForm";
export { useAuth } from "./model/useAuth";
export type { AuthUser } from "./model/types";
// loginApi 같은 내부 구현은 노출하지 않음
```

```ts
// ✅ 외부에서 사용할 때
import { LoginForm, useAuth } from "@/features/auth";  // index.ts 경유

// ❌ 내부 파일 직접 접근 금지
import { LoginForm } from "@/features/auth/ui/LoginForm";  // 금지!
```

**Public API의 역할**  
- 내부 구조를 리팩토링해도 외부 코드는 영향 없음
- 슬라이스 인터페이스가 한 곳에서 명확하게 파악됨
- 실수로 내부 구현에 의존하는 것을 방지

### entities 간 참조: @x 노테이션

entities끼리 참조가 필요할 때만 (최후 수단):

```
entities/
├── song/
│   ├── @x/
│   │   └── artist.ts   ← artist 슬라이스 전용 public API
│   └── index.ts
└── artist/
    └── model/
        └── artist.ts   ← song/@x/artist에서 import
```

```ts
// entities/artist/model/artist.ts
import type { Song } from "entities/song/@x/artist";  // A/@x/B = A와 B의 교차
```

---

## 8. 흔한 실수와 해결책

### ① shared/가 utils 덤프가 되는 경우

```
# ❌ 흔한 실수
shared/
└── utils/           ← 온갖 것이 다 쌓임
    ├── authUtils.ts
    ├── cartUtils.ts
    └── formatDate.ts

# ✅ 목적별로 분리
shared/
├── lib/
│   └── date/        ← 날짜 관련 유틸
├── auth/            ← 인증 관련 (토큰, 현재 사용자)
└── api/             ← HTTP 클라이언트
```

### ② 모든 도메인 개념을 entities로 만드는 경우

entities는 **여러 레이어에서 재사용되는** 도메인 모델에만 사용한다.  
한 페이지에서만 쓰이는 타입은 그 page에, feature에서만 쓰이면 그 feature에.

```
# ❌ 과도한 엔티티화
entities/
├── loginFormData/    ← 로그인 폼에서만 씀
├── productFilter/    ← 상품 목록 페이지에서만 씀
└── cartTotal/        ← 장바구니에서만 씀

# ✅ 실제로 여러 곳에서 쓰이는 것만
entities/
├── user/            ← 여러 features에서 사용
└── product/         ← 여러 pages/features에서 사용
```

### ③ features에 재사용 안 되는 기능을 넣는 경우

```
# ❌ 한 페이지에서만 쓰이는데 features로
features/
└── homePageBanner/    ← 홈 페이지에서만 사용

# ✅ 해당 페이지 안에
pages/
└── home/
    └── ui/
        └── HomeBanner.tsx
```

### ④ 세그먼트 이름을 기술 타입으로 짓는 경우

```
# ❌ 기술 타입 기준 — "이게 뭐야?"
features/auth/
├── components/   ← React 컴포넌트들
├── hooks/        ← 커스텀 훅들
└── types/        ← 타입 정의들

# ✅ 목적 기준 — "무슨 일을 해?"
features/auth/
├── ui/           ← 화면에 그리는 것들
├── model/        ← 인증 상태와 비즈니스 로직
└── api/          ← 로그인/로그아웃 요청
```

`ui/`와 `model/`은 `components/`와 `hooks/`의 이름만 바꾼 것이 **아니다**.  
예를 들어 `useScrollEffect` 훅은 화면 표현 목적이므로 `model/`이 아닌 `ui/`에 들어간다.

### ⑤ DTO와 도메인 모델을 무조건 분리하는 경우

DTO → 도메인 모델 변환이 항상 필요한 건 아니다.  
**"DTO와 앱에서 다루고 싶은 데이터가 얼마나 다른가"** 를 기준으로 결정한다.

| 상황 | 처리 방식 |
|------|---------|
| 이름만 어색함 (API 패키지의 `GetProductResponse` 등) | `shared/api/`에서 re-export로 의미있는 이름 부여 |
| 형태도 다름 (snake_case, 중첩 wrapper) | `shared/api/`에 정규화 mapper (인프라 관심사) |
| 개념 자체가 다름 (프론트엔드 전용 도메인 개념) | 슬라이스 `model/`에 도메인 mapper + 도메인 타입 |

```ts
// 1. 이름만 어색: re-export로 해결
export type { GetProductResponse as ProductDTO } from 'api-client-package';

// 2. 형태도 다름: shared/api/에 정규화 mapper
export async function fetchProduct(id: string): Promise<Product> {
  const raw = await apiPackage.getProduct(id);
  return { name: raw.product_name, isAvailable: raw.is_in_stock, price: raw.price_cents / 100 };
}

// 3. 개념 자체가 다름: 슬라이스 model/에 도메인 mapper
// features/order/model/orderMapper.ts
```

TanStack Query도 마찬가지다:
- **Thin client** (DTO == 도메인 모델): queryOptions → 슬라이스의 `api/` 또는 `shared/api/`
- **Thick client** (DTO ≠ 도메인 모델): queryOptions → `model/`, queryFn에서 mapper 적용

### ⑥ index.ts에서 wildcard export

```ts
// ❌ 절대 금지
export * from "./ui/LoginForm";
export * from "./model/useAuth";

// ✅ 명시적으로 필요한 것만
export { LoginForm } from "./ui/LoginForm";
export { useAuth } from "./model/useAuth";
```

---

## 9. 실전 예시: 기존 구조 → FSD 변환

### 변환 전 (기술 역할 기반)

```
src/
├── components/
│   ├── UserCard.tsx
│   ├── UserProfile.tsx
│   ├── ProductCard.tsx
│   ├── ProductList.tsx
│   ├── CartItem.tsx
│   ├── CartSummary.tsx
│   ├── LoginForm.tsx
│   └── Header.tsx
├── hooks/
│   ├── useAuth.ts
│   ├── useCart.ts
│   └── useProducts.ts
├── services/
│   ├── authService.ts
│   ├── productService.ts
│   └── cartService.ts
├── store/
│   ├── authSlice.ts
│   ├── cartSlice.ts
│   └── productSlice.ts
├── types/
│   ├── User.ts
│   ├── Product.ts
│   └── Cart.ts
└── pages/
    ├── LoginPage.tsx
    ├── HomePage.tsx
    ├── ProductPage.tsx
    └── CartPage.tsx
```

### 변환 후 (FSD)

```
src/
├── app/
│   ├── routes/          ← 라우터 설정 (기존 pages/ 라우팅 로직)
│   └── store/           ← 스토어 초기화
│
├── pages/
│   ├── login/
│   │   └── ui/
│   │       └── LoginPage.tsx
│   ├── home/
│   │   └── ui/
│   │       └── HomePage.tsx
│   ├── product/
│   │   └── ui/
│   │       └── ProductPage.tsx
│   └── cart/
│       └── ui/
│           └── CartPage.tsx
│
├── widgets/
│   └── header/
│       ├── ui/
│       │   └── Header.tsx     ← 기존 components/Header.tsx
│       └── index.ts
│
├── features/
│   ├── auth/
│   │   ├── ui/
│   │   │   └── LoginForm.tsx  ← 기존 components/LoginForm.tsx
│   │   ├── model/
│   │   │   └── useAuth.ts     ← 기존 hooks/useAuth.ts + store/authSlice.ts
│   │   ├── api/
│   │   │   └── authService.ts ← 기존 services/authService.ts
│   │   └── index.ts
│   └── cart/
│       ├── ui/
│       │   ├── CartItem.tsx   ← 기존 components/CartItem.tsx
│       │   └── CartSummary.tsx
│       ├── model/
│       │   └── useCart.ts     ← 기존 hooks/useCart.ts + store/cartSlice.ts
│       ├── api/
│       │   └── cartService.ts
│       └── index.ts
│
├── entities/
│   ├── user/
│   │   ├── model/
│   │   │   └── user.ts        ← 기존 types/User.ts
│   │   ├── ui/
│   │   │   ├── UserCard.tsx   ← 기존 components/UserCard.tsx
│   │   │   └── UserProfile.tsx
│   │   └── index.ts
│   └── product/
│       ├── model/
│       │   └── product.ts     ← 기존 types/Product.ts
│       ├── ui/
│       │   ├── ProductCard.tsx ← 기존 components/ProductCard.tsx
│       │   └── ProductList.tsx
│       ├── api/
│       │   └── productService.ts ← 기존 services/productService.ts
│       └── index.ts
│
└── shared/
    ├── api/              ← HTTP 클라이언트, 공통 request 설정
    ├── ui/               ← Button, Modal 등 도메인 무관한 컴포넌트
    └── lib/              ← formatDate, validate 등 순수 유틸
```

### 변환의 핵심 원칙

변환은 **기술 타입 폴더 → 동일한 이름의 세그먼트** 이름 변경이 아니다.  
각 파일의 **도메인**과 **목적**을 파악해서 위치를 결정한다.

1. **도메인 파악 먼저**: 이 코드가 auth를 위한 건지, cart를 위한 건지, 공통인지 → 레이어/슬라이스 결정
2. **목적 파악 두 번째**: 화면을 그리는가, 상태를 다루는가, 백엔드와 통신하는가 → 세그먼트 결정

구체적인 이동 패턴 (타입이 아닌 목적으로 판단):

- `components/LoginForm.tsx` → `features/auth/ui/` (auth 도메인, 화면 표현 목적)
- `hooks/useAuth.ts` → `features/auth/model/` (auth 도메인, 상태 관리 목적)
- `hooks/useScrollEffect.ts` → 해당 슬라이스의 `ui/` (화면 표현 목적)
- `services/authService.ts` → `features/auth/api/` (auth 도메인, 백엔드 통신 목적)
- `store/authSlice.ts` → `features/auth/model/` (auth 도메인, 상태 관리 목적)
- `types/User.ts` → `entities/user/model/` (User 도메인 모델, 데이터 정의 목적)
- `utils/formatDate.ts` → `shared/lib/date/` (목적이 UI인지 로깅인지 단정 불가 → 도메인 무관 유틸)

---

## 참고 자료

- [공식 문서](https://feature-sliced.design)
- [Steiger 아키텍처 린터](https://github.com/feature-sliced/steiger) — FSD 규칙 위반 자동 감지
- `/fsd` 스킬 — 개발 중 구체적인 질문에 룰 파일을 동적으로 참조하여 답변
