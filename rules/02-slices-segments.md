# FSD 슬라이스 & 세그먼트 규칙

## 슬라이스 (Slice)

### 정의
- 레이어 안에서 **비즈니스 도메인**으로 코드를 분리하는 단위
- 이름은 표준화되지 않음 — 비즈니스 용어 그대로 사용
- `shared`, `app` 레이어는 슬라이스 없음

### 핵심 원칙: 제로 커플링 & 높은 응집도

```
좋은 슬라이스:
┌──────────────────┐  ┌──────────────────┐
│   features/auth  │  │ features/comments│  ← 서로 독립
│  - model         │  │  - model         │
│  - ui            │  │  - ui            │
│  - api           │  │  - api           │
└──────────────────┘  └──────────────────┘

나쁜 슬라이스:
┌──────────────────┐  ┌──────────────────┐
│   features/auth  │──▶ features/comments│  ← cross-import ❌
└──────────────────┘  └──────────────────┘
```

### 슬라이스 그룹핑

연관된 슬라이스를 폴더로 묶을 수 있지만, **그 폴더 안에 공유 코드는 금지**:

```
features/
└── post/           ← 그룹 폴더
    ├── compose/    ← 독립 슬라이스
    ├── like/       ← 독립 슬라이스
    └── delete/     ← 독립 슬라이스
    # ❌ some-shared-code.ts 금지 — 공유 코드는 entities 또는 shared로
```

## 세그먼트 (Segment)

### 세그먼트 간 의존성 방향

슬라이스 내부에서 세그먼트 간에도 단방향 의존성을 적용한다:

```
ui  →  model  →  api
```

- `ui/`는 `model/`과 `api/` 모두 참조 가능
- `model/`은 `api/` 참조 가능
- `api/`는 슬라이스 내 다른 세그먼트를 참조할 수 없음

### 표준 세그먼트 이름

| 세그먼트 | 목적 | 포함 내용 |
|---------|------|---------|
| `ui` | 화면에 그리는 것 | 컴포넌트, 스타일, 화면 렌더링 훅 |
| `api` | 백엔드와 통신하는 것 | 요청 함수, DTO 타입 |
| `model` | 데이터·상태를 다루는 것 | 도메인 모델 타입, 스토어, 비즈니스 로직, 검증, **매퍼** |
| `config` | 설정·플래그 | 설정값, feature flag |

표준 세그먼트 외에 커스텀 세그먼트를 자유롭게 추가할 수 있다. 단, 이름은 *무엇을 하는가*를 드러내야 한다 (`components`, `hooks`, `types`, `utils`, `helpers`, `lib` 같은 *기술 타입* 명칭 금지 — `→ 세그먼트 이름 규칙` 참고).

### API 응답 타입 처리

앱 내부 코드가 API 패키지나 백엔드 응답 구조에 직접 의존하지 않도록, `shared/api/`를 경계로 삼는다.  
처리 깊이는 **DTO와 앱에서 다루고 싶은 데이터가 얼마나 다른가**에 따라 결정한다.

| 상황 | 처리 방식 | 위치 |
|------|---------|------|
| 이름만 어색함 (HTTP 메서드 기반 네이밍 등) | re-export로 의미있는 이름 부여 | `shared/api/` |
| 형태도 다름 (snake_case, 중첩 wrapper 등) | 정규화 mapper (인프라 관심사) | `shared/api/` |
| 개념 자체가 다름 (프론트엔드 전용 도메인) | 도메인 mapper + 도메인 모델 | 해당 슬라이스 `model/` |

```ts
// 1. 이름만 어색한 경우 — re-export
// shared/api/product.ts
export type { GetProductResponse as ProductDTO } from 'api-client-package';

// 2. 형태도 다른 경우 — 정규화 mapper (shared/api/에서 처리)
export type Product = { name: string; isAvailable: boolean; price: number };

function adaptProduct(dto: GetProductResponse): Product {
  return { name: dto.product_name, isAvailable: dto.is_in_stock, price: dto.price_cents / 100 };
}

export async function fetchProduct(id: string): Promise<Product> {
  return adaptProduct(await apiPackage.getProduct(id));
}

// 3. 개념 자체가 다른 경우 — 도메인 mapper는 슬라이스 model/에
// features/order/model/orderMapper.ts
```

**데이터** → thin/thick client 구분에 따라 처리 (→ `rules/06-tanstack-query.md` 참조)

**페이지네이션** → 범용 래퍼 타입을 `shared/model/`에 정의, `queryFn` 안에서 아이템 매핑과 함께 처리:

```ts
// shared/model/pagination.ts
export interface Paginated<T> {
  items: T[];
  totalCount: number;
  page: number;
  pageSize: number;
  hasNextPage: boolean;
}

// features/product/model/product.queries.ts (queryFn 안)
queryFn: async () => {
  const dto = await fetchProducts(params);
  return {
    items: dto.data.map(adaptProduct),
    totalCount: dto.total_count,
    page: dto.page,
    pageSize: dto.per_page,
    hasNextPage: dto.has_next,
  } satisfies Paginated<Product>;
}
```

**에러** → 범용 에러 타입을 `shared/api/`에 정의:

```ts
// shared/api/error.ts
export interface ApiError {
  code: string;
  message: string;
  status: number;
}
```

| 에러 종류 | 처리 위치 |
|----------|---------|
| 범용 에러 (HTTP/네트워크) | `app/providers/`의 `QueryCache.onError` |
| 도메인 에러 (비즈니스 의미가 있는 코드) | `model/`의 `useMutation` `onError` |

판단 기준: **"이 에러 코드가 비즈니스 의미를 가지는가?"**

### 매퍼(Mapper)는 `model/`에

DTO → 도메인 모델 변환 함수(mapper)를 `api/`에 두면 의존성 방향이 깨진다.

```
# ❌ api/에 mapper — api가 model(도메인 타입)에 의존 → 방향 위반
api/
└── productMapper.ts   # Product 타입을 쓰기 위해 model/을 import → api → model ❌

# ✅ model/에 mapper — model이 api(DTO 타입)에 의존 → 방향 준수
model/
└── productMapper.ts   # ProductDTO를 쓰기 위해 api/를 import → model → api ✅
```

```ts
// ✅ features/product/model/productMapper.ts
import type { ProductDTO } from '../api/product.dto';  // model → api ✅
import type { Product } from './product';

export function adaptProduct(dto: ProductDTO): Product {
  return {
    id: String(dto.id),
    name: dto.name,
    price: dto.priceCents / 100,
  };
}
```

### model/ 내부 구성

`model/` 안에 들어오는 것: 클라이언트 상태(Redux/Zustand), 서버 상태(TanStack Query), 도메인 타입, mapper.  
도메인 개념이 하나라면 파일을 평탄하게, 여럿이라면 도메인별 디렉토리로 묶는다.

```
# 단일 모델 — 평탄한 구조
features/product/
└── model/
    ├── product.ts
    ├── productMapper.ts
    ├── product.queries.ts
    └── useProduct.ts

# 복수 모델 — 도메인별 디렉토리
features/order-management/
└── model/
    ├── order/
    │   ├── order.ts
    │   ├── orderMapper.ts
    │   ├── order.queries.ts
    │   └── useOrder.ts
    └── order-status/
        └── orderStatusStore.ts   # 클라이언트 상태
```

**주의**: `model/` 안에 서브 디렉토리가 여럿 필요할 만큼 모델이 다양해진다면, 슬라이스를 분리해야 한다는 신호일 수 있다. 먼저 슬라이스 분리를 검토하고, 분리 후에도 같은 슬라이스에 모델이 여럿이라면 그때 서브 디렉토리로 구성한다.

### 파일명 규칙

파일명도 **도메인 기반**으로 짓는다. 기술적 역할명은 금지한다.

```
❌ 기술적 역할명 — "이게 뭐야?"
model/types.ts
model/utils.ts
model/helpers.ts

✅ 도메인 기반명 — "무슨 데이터/로직이야?"
model/user.ts        ← 사용자 도메인 타입 + 로직
model/order.ts       ← 주문 도메인 타입 + 로직
api/fetch-profile.ts ← 프로필 페칭
api/update-settings.ts ← 설정 업데이트
```

파일명이 `types.ts`나 `utils.ts`라면 그 안에 무엇이 있는지 알 수 없다.  
도메인 이름을 쓰면 파일명 자체가 문서가 된다.

### 세그먼트 이름 규칙

세그먼트 이름은 **내용의 목적**을 기술해야 한다, 본질(기술 타입)이 아니라.

```
❌ 금지 (기술 타입 기술 — "이게 뭐야?"):
features/auth/
├── components/   ← "React 컴포넌트들이 있음"
├── hooks/        ← "커스텀 훅들이 있음"
└── types/        ← "타입 정의들이 있음"

✅ 권장 (목적 기술 — "무슨 일을 해?"):
features/auth/
├── ui/           ← 화면에 그리는 것들
├── model/        ← 인증 상태와 비즈니스 로직
└── api/          ← 로그인/로그아웃 요청
```

### 중요: ui/model은 components/hooks의 이름 변경이 아님

`ui/`와 `model/`은 기술 타입이 아닌 **목적** 기준이다.  
같은 "훅"이어도 목적에 따라 들어가는 세그먼트가 달라진다:

```ts
// ✅ 상태/로직 목적 → model/
useAuthState()         // 인증 상태 관리
useFormValidation()    // 폼 유효성 검증 (비즈니스 로직)

// ✅ 화면 표현 목적 → ui/
useScrollAnimation()   // 스크롤 애니메이션
useIntersectionObserver() // 뷰포트 진입 감지
useResizeObserver()    // 화면 크기 변화 감지

// ✅ 백엔드 통신 목적 → api/
useProductsQuery()     // TanStack Query를 이용한 상품 조회 훅 (데이터 패칭)
```

마찬가지로 "순수 함수"도 목적에 따라 다른 세그먼트에 들어간다:

```ts
validateUserPolicy(email) → model/             // 회사 이메일 정책 — 비즈니스 규칙
isValidEmailFormat(email) → shared/validation/ // RFC 포맷 검사 — 도메인 무관 유틸
fetchProducts()           → api/               // 백엔드 요청
```

**포맷터(`formatDate`, `formatCurrency` 등)는 `shared/format/`에**  
포맷의 목적이 UI 렌더링인지 로깅인지 단정짓기 어렵지만, *포맷팅*이라는 책임 자체는 명확하므로 `shared/format/` 세그먼트로 묶는다.

**공유 코드 분리 원칙**  
헬퍼 함수는 사용하는 세그먼트 가까이 두는 것이 기본 (응집도 우선).  
한 세그먼트에서만 쓰인다면 그 세그먼트 안에, **여러 세그먼트에서 공유**해야 한다면 별도 세그먼트로 분리해 단방향 참조를 유지한다. 분리된 세그먼트의 이름은 *무엇을 하는가*를 드러내야 한다 (`validation/`, `mapping/`, `format/` 등).

```
# ❌ ui와 model이 한쪽에 헬퍼를 둠 → 역방향 의존
features/order/
├── ui/Order.tsx
└── model/order.ts          # ui의 헬퍼를 import → model → ui ❌

# ✅ 공유 헬퍼는 별도 세그먼트 → ui·model 모두 단방향으로 의존
features/order/
├── ui/Order.tsx            # validation/ import ✅
├── model/order.ts          # validation/ import ✅
└── validation/orderRules.ts
```

**세그먼트 결정 질문**: "이 코드를 변경하면 무엇이 영향을 받는가?"
- 화면이 바뀐다 → `ui/`
- 앱의 상태나 비즈니스 로직이 바뀐다 → `model/`
- 백엔드 통신이 바뀐다 → `api/`

### 커스텀 세그먼트

모든 레이어에서 커스텀 세그먼트를 자유롭게 만들 수 있다. 단, 이름은 *무엇을 하는가*를 드러내야 한다.

```
shared/
├── api/
├── ui/
├── config/
├── routes/      ← 라우팅
├── i18n/        ← 국제화
├── auth/        ← 토큰 관리
├── format/      ← 포맷팅 (formatDate, formatCurrency)
├── validation/  ← 형식 검증 (isValidEmailFormat)
└── storage/     ← 영속 저장 (localStorage, cookieStorage)
```

**금지되는 이름**: `components`, `hooks`, `types`, `utils`, `helpers`, `lib` — *기술 타입* 명칭은 안에 무엇이 있는지 알려주지 않는다.

새 파일이 들어올 때 "이건 무엇을 위한 것인가?"를 물어 적절한 세그먼트로 보낸다. 진짜로 단발성이라 카테고리가 어색하면 *해당 사용처 슬라이스 안*에 둔다 — shared 루트에 catch-all 세그먼트는 만들지 않는다.

## 전형적인 슬라이스 구조

```
features/
└── user-profile/
    ├── ui/
    │   ├── UserProfileCard.tsx
    │   └── index.ts        ← ui 세그먼트의 public API
    ├── api/
    │   └── updateProfile.ts
    ├── model/
    │   └── profileStore.ts
    └── index.ts            ← 슬라이스의 public API (외부가 참조하는 유일한 경로)
```

## 슬라이스 크기 기준

- **너무 작음**: 한 파일짜리 슬라이스 → shared나 상위 슬라이스로 병합 고려
- **너무 큼**: 무관한 기능이 한 슬라이스에 → 슬라이스 분리 고려
- **판단 기준**: 팀원이 이 슬라이스를 찾을 때 직관적으로 이름을 떠올릴 수 있는가?
