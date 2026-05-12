# FSD Entities 레이어 설계 규칙

## 주의사항

`entities/`는 모든 상위 레이어에서 접근 가능하므로, 변경 시 파급 범위가 넓다.  
신중하게 설계하고 **필요할 때만** 추가하라.

## 1. Entities 없이도 괜찮다

앱이 thin client(백엔드가 대부분 처리, 클라이언트 비즈니스 로직 최소)라면 `entities` 레이어 자체를 두지 않아도 된다. 이는 FSD 위반이 아니며, 오히려 구조를 단순하게 유지한다.

**thin client**: 데이터를 백엔드와 교환하는 것이 주 역할  
**thick client**: 클라이언트에서 중요한 비즈니스 로직 처리

## 2. 선제적 슬라이싱 피하기

처음부터 entities를 세분화하지 말 것. 요구사항이 안정화될 때까지 `page`나 `feature`의 `model` 세그먼트에 코드를 두고, 나중에 리팩토링으로 entities로 올려라.

> entities로 코드를 옮기는 시점이 늦을수록, 리팩토링 위험이 줄어든다.

## 3. 불필요한 Entities 생성 금지

모든 비즈니스 로직에 엔티티를 만들지 말 것. `shared/api`의 타입을 활용하고 로직은 현재 슬라이스의 `model`에 두어라.

```
# ❌ 과도한 엔티티화
entities/
└── order/
    └── model/
        └── getOrderId.ts   ← 단순 유틸에 불과

# ✅ 재사용 비즈니스 로직만 entities에
entities/
└── order/
    ├── model/
    │   └── apply-discount.ts   ← 여러 곳에서 재사용되는 실제 비즈니스 로직
    └── index.ts

shared/
└── api/
    └── endpoints/
        └── order.ts            ← 타입 및 DTO 정의
```

## 4. CRUD 작업은 shared/api에

단순 CRUD는 비즈니스 로직이 없으므로 `entities`에 넣지 말 것.

```
# ✅ CRUD는 shared/api에
shared/
└── api/
    └── endpoints/
        ├── order.ts      # 주문 CRUD
        ├── products.ts   # 상품 CRUD
        └── cart.ts       # 장바구니 CRUD
```

복잡한 CRUD(트랜잭션, 롤백)는 entities 고려 가능하지만, 신중하게.

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

- [ ] 여러 레이어에서 재사용되는 도메인 개념인가?
- [ ] 단순 CRUD가 아닌 실제 비즈니스 로직이 있는가?
- [ ] entities로 올리지 않으면 코드 중복이 명백한가?
- [ ] @x 사용 전 슬라이스 병합을 먼저 고려했는가?
