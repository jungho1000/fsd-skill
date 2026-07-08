---
domain: fsd
topic: segments
triggers: [segment, 세그먼트, ui, model, api, config, mapper, DTO, 커스텀 세그먼트, 폴더 구조, 의존성 방향, 공유 코드]
status: published
---

# FSD 세그먼트 — 권장 패턴

> **이 문서는 권장 패턴이다.** 세그먼트의 하드 룰(이름·구성 자유 + 단방향 의존성 + 이름 금지 목록)은 [[segment-rules]]에 있다. 아래 내용은 "이 패턴을 따를 때" 가장 흔히 마주치는 결정들을 모아 둔 것이다.

## 세그먼트 (Segment)

### 권장 의존성 방향

`ui/model/api` 패턴을 따를 때의 흐름:

```
ui  →  model  →  api
```

- `ui/`는 `model/`과 `api/` 모두 참조 가능
- `model/`은 `api/` 참조 가능
- `api/`는 슬라이스 내 다른 세그먼트를 참조할 수 없음

이는 하드 룰("단방향 의존성")의 한 인스턴스다. 다른 세그먼트 구성을 쓰더라도 *임의의 두 세그먼트 사이에 방향은 하나*라는 원칙은 동일하게 적용된다.

### tier 개념 (참고)

> 팀에서 합의된 분류는 아니다 — 참고 모델로만 다룬다.

`ui → model → api`는 더 일반적인 *tier* 개념의 한 인스턴스로 볼 수 있다.

| tier | 역할 | `ui/model/api` 매핑 |
|------|------|--------------------|
| presentation | 화면, 상호작용 | `ui` |
| domain | 비즈니스 규칙, 상태 | `model` |
| infrastructure | 외부 통신, 저장소 | `api` |

방향성은 **바깥(presentation) → 안쪽(infrastructure)**. 새 커스텀 세그먼트(`forms/`, `validation/`, `services/` 등)를 만들 때 어느 tier에 해당하는지 식별해두면 단방향 의존성을 자연스럽게 만족시킬 수 있다.

### 표준 세그먼트 이름

`ui/api/model/config`는 가장 흔히 쓰이는 이름 모음이다. **필수는 아니다** — 슬라이스에 필요한 것만 두면 된다.

| 세그먼트 | 목적 | 포함 내용 |
|---------|------|---------|
| `ui` | 화면에 그리는 것 | 컴포넌트, 스타일, 화면 렌더링 훅 |
| `api` | 백엔드와 통신하는 것 | 요청 함수, DTO 타입 |
| `model` | 데이터·상태를 다루는 것 | 도메인 모델 타입, 스토어, 비즈니스 로직, 검증, **매퍼** |
| `config` | 설정·플래그 | 설정값, feature flag |

이외에 커스텀 세그먼트를 자유롭게 추가할 수 있다. 단, 이름은 *무엇을 하는가*를 드러내야 한다 (이름 규칙·금지 목록은 → `세그먼트 이름 규칙` 절).

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

**데이터** → thin/thick client 구분에 따라 처리 (→ `rules/tanstack-query.md` 참조)

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

### 매퍼는 데이터를 소비하는 세그먼트에

매퍼는 *세그먼트(또는 tier) 경계*에서 데이터를 변환하는 코드다. **변환된 데이터를 소비하는 쪽** 세그먼트에 둔다 — 그래야 단방향 의존성이 자연스럽게 유지된다.

| 변환 방향 | 매퍼 위치 |
|---------|---------|
| DTO(infrastructure) → 도메인 모델 | 도메인을 정의한 세그먼트 (예: `model/`) |
| 도메인 모델 → UI ViewModel | 표현 세그먼트 (예: `ui/`) |
| 변환이 굳이 필요 없음 | 매퍼 두지 않음 — `shared/api/` re-export로 충분 |

매퍼를 *데이터 출처* 쪽 세그먼트에 두면 출처 세그먼트가 소비 쪽의 타입을 알아야 하므로 의존성이 역류한다.

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

위 예시는 `ui/model/api` 패턴에서의 구체화일 뿐이다. 다른 세그먼트 구성(예: `presentation/domain/services/`)에서도 같은 원칙이 적용된다 — 매퍼는 *변환된 데이터를 쓰는 쪽*에 둔다.

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

### model은 순수 데이터 + 액션 — 표시 포맷은 ui로

`model/`에는 **순수 데이터와 그 데이터를 다루는 액션/셀렉터**만 둔다. "어떻게 보여줄지"(통화 기호, 날짜 라벨, 말줄임 요약, 배지 색상/className 등)는 표현 관심사이므로 `model/`·매퍼·도메인 엔티티에 넣지 않는다.

판별 기준: **표시용 문자열을 만들면 ui, 데이터만 다루면 model.**

| 코드 | 분류 | 위치 |
|------|------|------|
| `amount: number`, `createdAt: ISO` | 순수 데이터 | `model/` (도메인 엔티티) |
| `filterList()`, `countNeedConfirm()`, `createDefault()` | 데이터 액션/셀렉터 | `model/` |
| `formatAmount(n) → "₩30,000"`, `formatDateTime(iso)` | 표시 포맷 | `ui/` (뷰 모델) |
| `getBadge(type) → { tone, className }` | 표시 파생 | `ui/` (뷰 모델) |

도메인 엔티티에 `amountLabel`/`dateLabel`처럼 *이미 포맷된* 필드를 두지 않는다 — 매퍼가 표시 포맷을 수행하게 되어 도메인이 표현에 오염된다. 엔티티는 원천 데이터(`amount`, `createdAt`)만 갖고, 포맷은 렌더 시점에 뷰 모델이 수행한다.

### 뷰 모델은 ui 세그먼트에 colocate — `{컴포넌트}.model.ts`

특정 컴포넌트의 렌더링을 위해 도메인 데이터를 가공하는 코드(포맷터, 배지 매핑, 표시용 파생 타입)는 그 컴포넌트 옆 `ui/{컴포넌트}.model.ts`에 둔다.

```
ui/
├── receipt-card.tsx
├── receipt-card.model.ts          # formatAmount, getReceiptTypeBadge … (카드 뷰 모델)
├── filter-receipt-sheet.tsx
└── filter-receipt-sheet.model.ts  # 시트 앵커 설정 등 (시트 뷰 모델)
```

배치 판단 (= 공유 코드 분리 원칙 + 사용처 응집):

| 사용 범위 / 성격 | 위치 |
|----------------|------|
| 여러 슬라이스가 공유하는 도메인 모델 | `entities/` (→ [[entities]]) |
| 한 슬라이스 안에서만 쓰는 표현 가공 | 그 슬라이스 `ui/`의 `{컴포넌트}.model.ts` |
| 도메인 무관 범용 포맷터(`formatDate` 등) | `shared/format/` (아래 "포맷터" 항목) |

### 값 객체는 팩토리 함수로 불변식을 강제

값 객체(연·월, 좌표, 금액 범위 등)는 객체 리터럴로 직접 만들지 않고 **팩토리 함수**(`createX()`)를 통해 만든다. 팩토리가 불변식을 검증해 위반 시 에러를 던지고, 동결된 객체를 반환한다.

- 타입 위반(정수 아님, `NaN` 등) → `TypeError`
- 범위 위반(`month` ∉ 1~12 등) → `RangeError`
- 반환값은 `readonly` 필드 + `Object.freeze`로 불변 보장

```ts
export interface YearMonth {
  readonly year: number;
  readonly month: number;
}

export function createYearMonth(year: number, month: number): YearMonth {
  if (!Number.isInteger(year) || !Number.isInteger(month)) {
    throw new TypeError(`정수 year/month만 허용 (year=${year}, month=${month})`);
  }
  if (month < 1 || month > 12) {
    throw new RangeError(`month는 1~12 (month=${month})`);
  }
  return Object.freeze({ year, month });
}
```

외부 입력(URL 쿼리, 폼 등)에서 값 객체를 만들 때는 팩토리를 `try/catch`로 감싸 위반 값을 기본값으로 복구한다 — **경계에서 던지고 호출부에서 복구**한다. 앱 내부(항상 유효한 값)에서는 그대로 통과시킨다.

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

`shared/`의 커스텀 세그먼트 예시 트리는 → [[layers]] `shared/` 섹션.

이름 규칙·금지 목록은 위 `세그먼트 이름 규칙` 절 참고.

새 파일이 들어올 때 "이건 무엇을 위한 것인가?"를 물어 적절한 세그먼트로 보낸다. 진짜로 단발성이라 카테고리가 어색하면 *해당 사용처 슬라이스 안*에 둔다 — shared 루트에 catch-all 세그먼트는 만들지 않는다.

### 정적 리소스 (이미지·SVG·폰트) 배치

정적 리소스(이미지·SVG·아이콘·폰트 등)도 코드와 동일하게 **사용하는 곳 가까이** 둔다. `public/assets/` 같은 루트 덤프 폴더에 전부 모아두면 *어디서 쓰는지* 알기 어렵고, 슬라이스를 옮기거나 제거할 때 리소스가 고아로 남는다.

판단 흐름 (= 공유 코드 분리 원칙과 동일):

| 사용 범위 | 위치 |
|---------|------|
| 한 컴포넌트/한 슬라이스에서만 사용 | 해당 슬라이스 내 `assets/` 또는 `ui/` 옆 |
| 한 슬라이스 내 여러 세그먼트가 공유 | 슬라이스 루트의 `assets/` 세그먼트 |
| 여러 슬라이스가 공유 | `shared/assets/` (또는 `shared/icons/`, `shared/illustrations/`) |

```
# ❌ 루트 덤프 — 사용처를 모름
public/assets/images/profile/delivery/img_ssoreder01.svg   # delivery 페이지 1곳에서만 씀
public/assets/images/agency/img_agency_macau.png           # agency 페이지 1곳에서만 씀

# ✅ 사용처 슬라이스로 colocate
pages/profile-delivery/assets/img_ssoreder01.svg
pages/profile-agency/assets/img_agency_macau.png
shared/assets/img_FAQ.png                                  # 여러 페이지에서 공유
```

**Next.js `public/` 사용 기준**: 정적 URL로 *반드시 직접 서빙되어야 하는* 리소스만 둔다 (favicon, robots.txt, sitemap.xml, OG/메타 이미지 등). 컴포넌트에서 `import`로 쓰는 이미지는 슬라이스 안에 둬도 `next/image` + webpack 로더가 정상적으로 번들링한다.

**세그먼트 이름**: `assets/`, `icons/`, `illustrations/`, `images/`처럼 *무엇을 담는지*를 드러내는 이름을 쓴다. `static/`, `files/`처럼 catch-all 이름은 피한다.

## Relations

- defined-by :: [[overview]]
- extends :: [[segment-rules]]
- part-of :: [[slices]]
- depends-on :: [[slices]]
- extended-by :: [[tanstack-query]]
- see-also :: [[public-api]]
