# FSD Cross-import 규칙

## Cross-import이란?

**같은 레이어의 다른 슬라이스 간 import**.

```ts
// ❌ cross-import 예시
// features/cart/ui/Cart.tsx
import { ProductCard } from "@/features/product";  // 같은 features 레이어
```

`shared`, `app` 레이어는 슬라이스가 없으므로 해당 없음.

## 왜 코드 스멜인가?

| 문제 | 설명 |
|------|------|
| 소유권 모호 | `cart`가 `product`를 알면 누가 책임자인지 불분명 |
| 테스트 격리 불가 | `cart` 테스트 시 `product`도 세팅해야 함 |
| 인지 부하 증가 | 변경 범위를 파악하기 위해 더 많은 컨텍스트 필요 |
| 순환 의존성 위험 | A→B, B→A로 진화할 수 있음 |

## 4가지 해결 전략

### Strategy A: 슬라이스 병합

두 슬라이스가 항상 함께 변경된다면 → 하나의 슬라이스로 합쳐라.

```
# Before (서로 cross-import)
features/profile
features/profileSettings

# After (실질적으로 하나의 기능)
features/profile/
├── ui/
│   ├── ProfilePanel.tsx
│   └── ProfileSettings.tsx
└── index.ts
```

### Strategy B: 공통 도메인 로직을 entities로 내리기

여러 features가 공유하는 도메인 로직 → `entities`로 이동.

```
# features/auth와 features/profile 모두 세션 검증 필요
entities/
└── session/
    ├── model/
    │   └── validateSession.ts   ← 여기로 이동
    └── index.ts

# features에서 참조
import { validateSession } from "@/entities/session";
```

### Strategy C: 상위 레이어에서 조합 (IoC)

슬라이스끼리 직접 참조하는 대신, `pages`나 `app`에서 조합.

```tsx
// ✅ pages/UserDashboardPage.tsx
import { UserProfilePanel } from "@/features/userProfile";
import { ActivityFeed } from "@/features/activityFeed";

export function UserDashboardPage() {
    return (
        <div>
            <UserProfilePanel />
            <ActivityFeed />
        </div>
    );
}
// features/userProfile와 features/activityFeed는 서로를 모름
```

**Render Props 패턴** (React):
```tsx
// features/commentList/ui/CommentList.tsx
interface CommentListProps {
    renderUserAvatar?: (userId: string) => React.ReactNode;
}

// pages/PostPage.tsx
import { CommentList } from "@/features/commentList";
import { UserAvatar } from "@/features/userProfile";

<CommentList renderUserAvatar={(id) => <UserAvatar userId={id} />} />
// CommentList는 UserAvatar를 직접 import하지 않음
```

**Vue Slots 패턴**:
```vue
<!-- pages/PostPage.vue -->
<CommentList :comments="comments">
    <template #avatar="{ userId }">
        <UserAvatar :userId="userId" />
    </template>
</CommentList>
```

### Strategy D: Public API 통해서만 허용

위 전략이 맞지 않을 때 마지막 선택지. **store/model 직접 접근은 금지**하고 exported hook/component만 사용.

```ts
// features/auth/index.ts
export { useAuth } from "./model/useAuth";   // ✅ 허용된 것만 노출
export { AuthButton } from "./ui/AuthButton";

// features/profile/ui/ProfileMenu.tsx
import { useAuth } from "@/features/auth";  // ✅ Public API를 통해서만
// ❌ import from "@/features/auth/model/internal/*"  직접 접근 금지
```

## 언제 cross-import를 허용하나?

**경고 신호 (반드시 해결)**:
- 다른 슬라이스의 store/model/내부 로직 직접 접근
- 내부 파일 경로 직접 import
- 양방향 의존성 (A→B, B→A)
- 한 슬라이스 변경이 자주 다른 슬라이스를 깨뜨림

**팀 맥락에 따른 판단**:
- 초기 스타트업: 실용적 속도를 위해 일부 허용 가능
- 장기 운영/규제 서비스: 엄격한 경계가 유지보수에 이득

cross-import를 허용한다면:
1. 의도적 아키텍처 결정임을 팀과 공유
2. 이유를 문서화
3. 주기적으로 재검토

## entities의 @x 노테이션

entities 레이어에서 불가피한 cross-import에는 `@x` 사용 (최후 수단).  
사용 방법과 구조는 `rules/03-public-api.md` 참조.
