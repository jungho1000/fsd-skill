# FSD Entities 레이어 설계 규칙

## 주의사항

`entities/`는 모든 상위 레이어에서 접근 가능하므로, 변경 시 파급 범위가 넓다.  
신중하게 설계하고 **필요할 때만** 추가하라.

## entities는 도메인 모델

`entities/<엔티티>/`는 *단일 도메인 엔티티*를 표현하는 슬라이스다. 도메인 모델은 다음을 책임진다.

- 속성 (도메인 타입)
- 액션 (속성을 변경하고 비즈니스 규칙을 지키는 메서드)
- 인스턴스 hydration (DTO → 도메인 모델 변환)

여러 엔티티를 오케스트레이션하거나 유즈케이스를 구성하는 것은 **features의 역할**이다.

## 1. Entities 없이도 괜찮다

앱이 thin client(백엔드가 대부분 처리, 클라이언트 비즈니스 로직 최소)라면 `entities` 레이어 자체를 두지 않아도 된다. 이는 FSD 위반이 아니며, 오히려 구조를 단순하게 유지한다.

**thin client**: 데이터를 백엔드와 교환하는 것이 주 역할  
**thick client**: 클라이언트에서 중요한 비즈니스 로직 처리

## 2. 선제적 슬라이싱 피하기

처음부터 entities를 세분화하지 말 것. 요구사항이 안정화될 때까지 `page`나 `feature`의 `model` 세그먼트에 코드를 두고, 나중에 리팩토링으로 entities로 올려라.

> entities로 코드를 옮기는 시점이 늦을수록, 리팩토링 위험이 줄어든다.

## 3. 추출 조건 — 두 조건이 모두 충족될 때만

| 조건 | 의미 |
|------|------|
| 반복 사용 | 두 곳 이상의 슬라이스/페이지에서 같은 도메인을 사용 |
| 도메인 변환·파생·액션 | DTO 변환, 파생 데이터 구성, 도메인 메서드 중 하나라도 있음 |

두 조건이 함께 충족되어야 entities로 끌어올린다. 한쪽만 있으면 다른 위치를 쓴다.

```
# ❌ 과도한 엔티티화 — 단순 유틸은 entities에 안 들어감
entities/
└── order/
    └── model/
        └── getOrderId.ts   ← 도메인 액션이 아닌 단순 유틸

# ✅ 단일 엔티티 + 변환 + 반복 사용
entities/
└── order/
    ├── model/
    │   ├── order.ts            # 도메인 타입
    │   ├── orderMapper.ts      # DTO → Order 변환
    │   ├── order.queries.ts    # useOrderQuery (도메인 모델 캐싱)
    │   └── apply-discount.ts   # 도메인 액션
    ├── api/
    │   ├── order.dto.ts        # DTO
    │   └── fetchOrder.ts       # 단일 엔티티 fetch
    └── index.ts
```

## 4. DTO를 그대로 쓴다면 shared/api/

DTO를 변환 없이 렌더링만 하는 경우는 *도메인 모델*이 아니다. entities에 끌어올리지 않는다.

```
# ✅ 변환 없는 데이터는 shared/api/
shared/
└── api/
    └── endpoints/
        ├── report.ts     # 보고서 — 단발성 데이터
        └── stats.ts      # 통계 데이터
```

이름만 어색하다면 `shared/api/`에서 re-export로 의미 있는 이름을 부여한다(자세한 매트릭스는 `rules/02-slices-segments.md`의 "API 응답 타입 처리" 참고).

```ts
// shared/api/product.ts
export type { GetProductResponse as ProductDTO } from 'api-client-package';
```

*얇은 entities re-export* 슬라이스는 만들지 않는다 — entities는 모두 *진짜 도메인 모델*이라는 보증을 유지한다.

## 5. 인증 데이터는 shared에

`user` 엔티티를 만들어 인증 정보를 저장하는 것은 피하라.  
인증 토큰, 백엔드에서 받은 user DTO는 컨텍스트 특화적이고 재사용 범위가 좁다.

```
# ✅ 인증 관련 데이터
shared/
├── auth/
│   ├── use-auth.ts   # 현재 사용자 정보, 토큰
│   └── index.ts
└── api/
    └── client.ts     # 인증 헤더 자동 첨부
```

**왜?** entities의 user를 shared/api가 참조하면 레이어 import 규칙 위반.  
인증 로직 상세는 `rules/01-layers.md`의 shared 섹션 참조.

## 6. Cross-import 최소화

`@x` 노테이션은 **최후 수단**이다. 남용하면 엔티티 간 결합이 증가하고 리팩토링 비용이 올라간다.

```
# ❌ 과도한 @x 사용 — 분리가 지나치게 세분화된 신호
entities/
├── order/       @x 사용
├── order-item/  @x 사용
└── order-customer-info/  @x 사용

# ✅ 연관된 것을 하나의 컨텍스트로 묶기
entities/
└── order-info/
    └── model/
        └── order-info.ts  # order, order-item, customer-info 통합
```

## 엔티티 설계 체크리스트

- [ ] 두 곳 이상에서 반복 사용되는가?
- [ ] DTO 변환·파생 데이터·도메인 액션 중 하나라도 있는가?
- [ ] 단일 엔티티의 책임에 머무는가? (여러 엔티티 조합이면 features로)
- [ ] @x 사용 전 슬라이스 병합을 먼저 고려했는가?
