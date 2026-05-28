# FSD + TanStack Query 규칙

## 핵심 원칙: TanStack Query는 model 계층

TanStack Query는 서버 상태를 캐싱하고 동기화하는 **모델 계층 도구**다.  
Redux/Zustand 같은 클라이언트 상태 관리 도구가 `model/`에 위치하는 것과 같은 이유다.

```
클라이언트 상태 (Redux, Zustand store)  →  model/
서버 상태 (TanStack Query cache)        →  model/
```

단, queryOptions의 정확한 위치는 **DTO와 도메인 모델이 얼마나 다른가**에 따라 달라진다.

---

## Thin Client vs Thick Client

| | Thin Client | Thick Client |
|--|------------|-------------|
| DTO vs 도메인 모델 | 같거나 형태만 다름 | 개념 자체가 다름 |
| Mapper | 불필요 또는 정규화만 | 필요 |
| queryOptions 위치 | 슬라이스의 `api/` | `model/` |
| 캐시에 저장되는 것 | DTO (= 도메인 모델) | 도메인 모델 |

**Thin client** — DTO가 곧 도메인 모델이므로 변환이 없다. queryOptions는 사용하는 슬라이스의 `api/` 또는 `shared/api/`에 둔다.

```ts
// pages/product/api/product.queries.ts (thin client)
export const productQueries = {
  detail: (id: string) => queryOptions({
    queryKey: ['products', id],
    queryFn: () => fetchProduct(id),  // DTO를 그대로 캐시에
  }),
};
```

**Thick client** — DTO와 도메인 모델이 다르므로 mapper가 필요하다. queryOptions는 `model/`에 두고, `queryFn` 안에서 mapper를 적용한다.

```ts
// features/product/model/product.queries.ts (thick client)
export const productQueries = {
  detail: (id: string) => queryOptions({
    queryKey: ['products', id],
    queryFn: async () => adaptProduct(await fetchProduct(id)),  // 도메인 모델을 캐시에
  }),
};
```

> **슬라이스 위치 선택**  
> 위 예시는 `features/product/`에 product query를 두지만, 이는 *한 슬라이스 한정* 사용을 가정한 시연이다.  
> 단일 도메인 엔티티이고 두 곳 이상에서 반복 사용된다면 `entities/product/`로 끌어올린다 (자세한 추출 조건은 `rules/05-entities.md` 참고).  
> 여러 엔티티를 조합하는 유즈케이스 단위 query라면 그대로 `features/<유즈케이스>/`에 둔다.

**판단 기준**: "백엔드 응답 구조와 내가 코드에서 다루고 싶은 데이터 구조가 다른가?"

```
api/    = 순수 HTTP 통신 (DTO 반환)
model/  = 도메인 데이터 관리 (queryOptions, useQuery, useMutation, mapper, 도메인 타입)
ui/     = 컴포넌트 (model/의 훅을 호출)
```

세그먼트 간 의존성은 그대로 유지된다:

```
ui  →  model  →  api
```

---

## 캐시에는 도메인 모델을 저장

`queryFn` 안에서 mapper를 적용해 **도메인 모델을 캐시에 저장**한다.  
`api/`의 fetch 함수는 DTO를 반환하고, `model/`의 query factory가 변환을 책임진다.

```
fetch 실행 (api/) → DTO → adaptProduct (model/) → [캐시: Product]
                                                          ↓
                                                    useQuery → 컴포넌트: Product
```

캐시가 도메인 모델이면:
- 앱 내부 코드는 DTO 구조를 알 필요가 없다
- `setQueryData`가 도메인 타입으로 자연스럽게 동작한다
- 낙관적 업데이트가 도메인 언어로 표현된다

---

## 파일 구조

```
features/product/
├── api/
│   ├── product.dto.ts       ← DTO 타입 (백엔드 응답 형태)
│   ├── fetchProduct.ts      ← 순수 fetch 함수 (DTO 반환)
│   └── index.ts
├── model/
│   ├── product.ts           ← 도메인 모델 타입 (Product)
│   ├── productMapper.ts     ← DTO → 도메인 모델 변환
│   ├── product.queries.ts   ← queryOptions (cache = 도메인 모델)
│   ├── useProduct.ts        ← useQuery 훅
│   ├── useUpdateProduct.ts  ← useMutation 훅
│   └── index.ts
└── ui/
    └── ProductPage.tsx      ← model/ 훅만 호출, DTO 무관
```

---

## `api/` — 순수 HTTP 통신

```ts
// features/product/api/product.dto.ts
export interface ProductDTO {
  id: number;
  price_cents: number;
  is_available: boolean;
}
```

```ts
// features/product/api/fetchProduct.ts
import { apiClient } from '@/shared/api';
import type { ProductDTO } from './product.dto';

export async function fetchProduct(id: string): Promise<ProductDTO> {
  return apiClient.get(`/products/${id}`);
}

export async function updateProduct(id: string, data: UpdateProductDTO): Promise<ProductDTO> {
  return apiClient.patch(`/products/${id}`, data);
}
```

`api/`는 DTO 타입과 fetch 함수만 갖는다. TanStack Query를 직접 다루지 않는다.

---

## `model/` — 도메인 데이터 관리

### 도메인 모델 + Mapper

```ts
// features/product/model/product.ts
export interface Product {
  id: string;
  price: number;
  isAvailable: boolean;
}
```

```ts
// features/product/model/productMapper.ts
import type { ProductDTO } from '../api/product.dto';  // model → api ✅
import type { Product } from './product';

export function adaptProduct(dto: ProductDTO): Product {
  return {
    id: String(dto.id),
    price: dto.price_cents / 100,
    isAvailable: dto.is_available,
  };
}
```

### Query Factory — `model/`에

`queryFn` 안에서 mapper를 적용해 캐시에 도메인 모델을 저장한다.

```ts
// features/product/model/product.queries.ts
import { queryOptions } from '@tanstack/react-query';
import { fetchProduct } from '../api/fetchProduct';    // model → api ✅
import { adaptProduct } from './productMapper';

export const productQueries = {
  all: () => ['products'] as const,
  details: () => [...productQueries.all(), 'detail'] as const,
  detail: (id: string) => queryOptions({
    queryKey: [...productQueries.details(), id],
    queryFn: async () => adaptProduct(await fetchProduct(id)),
    //                   ↑ 캐시에는 Product (도메인 모델) 저장
  }),
};
```

### Mutation Key도 `model/`에

```ts
// features/product/model/product.queries.ts (계속)
export const productMutations = {
  update: () => ['products', 'update'] as const,
};
```

### `useQuery` 훅

```ts
// features/product/model/useProduct.ts
import { useQuery } from '@tanstack/react-query';
import { productQueries } from './product.queries';

export function useProduct(id: string) {
  return useQuery(productQueries.detail(id));
  // data 타입: Product — select 불필요
}
```

### `useMutation` + 낙관적 업데이트

캐시가 도메인 모델이므로 낙관적 업데이트도 도메인 언어로 표현된다.

```ts
// features/product/model/useUpdateProduct.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { updateProduct } from '../api/fetchProduct';   // model → api ✅
import { productQueries, productMutations } from './product.queries';
import type { Product } from './product';

interface UpdateInput {
  id: string;
  price: number;  // 도메인 모델 필드 — price_cents를 알 필요 없음
}

export function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationKey: productMutations.update(),
    mutationFn: ({ id, price }: UpdateInput) =>
      updateProduct(id, { price_cents: price * 100 }),  // DTO 변환은 여기서만
    onMutate: async ({ id, price }) => {
      await queryClient.cancelQueries({ queryKey: productQueries.detail(id).queryKey });

      const previous = queryClient.getQueryData<Product>(
        productQueries.detail(id).queryKey,
      );

      // 도메인 모델로 낙관적 업데이트 — price_cents 같은 DTO 필드 불필요
      queryClient.setQueryData(productQueries.detail(id).queryKey, (old: Product) => ({
        ...old,
        price,
      }));

      return { previous };
    },
    onError: (_, { id }, context) => {
      queryClient.setQueryData(productQueries.detail(id).queryKey, context?.previous);
    },
    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: productQueries.detail(id).queryKey });
    },
  });
}
```

---

## `ui/` — 컴포넌트

`ui/`는 `model/`의 훅만 호출한다. DTO를 직접 다루지 않는다.

```tsx
// features/product/ui/ProductPage.tsx
import { useProduct } from '../model/useProduct';          // ui → model ✅
import { useUpdateProduct } from '../model/useUpdateProduct';

export function ProductPage({ id }: { id: string }) {
  const { data: product, isLoading } = useProduct(id);
  const { mutate: updatePrice } = useUpdateProduct();

  if (isLoading) return <Skeleton />;

  return (
    <div>
      <p>{product.price}</p>  {/* Product 타입, DTO 무관 */}
      <button onClick={() => updatePrice({ id, price: 29.99 })}>
        가격 변경
      </button>
    </div>
  );
}
```

---

## `QueryClient` 전역 설정 — `app/providers/`

```tsx
// app/providers/query-provider.tsx
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) => toast.error(error.message),
  }),
  mutationCache: new MutationCache({
    onError: (error) => toast.error(error.message),
  }),
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      gcTime: 5 * 60 * 1000,
    },
  },
});

export function QueryProvider({ children }: { children: ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools />
    </QueryClientProvider>
  );
}
```

Suspense 경계도 `app/providers/`에 둔다:

```tsx
// app/providers/suspense-provider.tsx
export const SuspenseProvider = ({ children }: { children: ReactNode }) => (
  <ErrorBoundary fallback={<ErrorPage />}>
    <Suspense fallback={<GlobalSpinner />}>
      {children}
    </Suspense>
  </ErrorBoundary>
);
```

---

## Query Factory 위치 결정

`model/` 안에 두되, 어느 슬라이스의 `model/`인지는 데이터의 도메인 소속에 따라 결정한다.

| 상황 | 위치 |
|------|------|
| 여러 슬라이스에서 같은 API를 공유 | `shared/api/` (fetch) + `entities/{entity}/model/` (query factory) |
| 특정 엔티티 단위 | `entities/{entity}/model/` |
| 특정 기능 전용 | `features/{feature}/model/` |
| 특정 페이지 전용 | `pages/{page}/model/` 또는 `pages/{page}/api/` |

---

## 배치 결정 요약

| 코드 | 위치 | 이유 |
|------|------|------|
| DTO 타입 | `api/` | 백엔드 응답 형태 |
| 순수 fetch 함수 | `api/` | 순수 HTTP 통신 |
| 도메인 모델 타입 | `model/` | 프론트엔드 도메인 표현 |
| Mapper | `model/` | `model → api` 방향 준수 |
| Query factory (`queryOptions`) | `model/` | 캐시 = 도메인 모델, TQ는 model 계층 도구 |
| Query key / Mutation key | `model/` (query factory와 같은 파일) | 캐시 식별자 = model 관심사 |
| `useQuery` / `useSuspenseQuery` 훅 | `model/` | 도메인 데이터 접근 추상화 |
| `useMutation` 훅 | `model/` | 도메인 액션, 낙관적 업데이트 |
| `setQueryData` 인자 타입 | 도메인 모델 | 캐시와 동일한 타입 |
| `QueryClient` 전역 설정 | `app/providers/` | 앱 전역 초기화 |
| Suspense 경계 | `app/providers/` | 앱 전역 초기화 |
