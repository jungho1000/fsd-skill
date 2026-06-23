---
domain: fsd
topic: public-api
triggers: [index.ts, public api, export, import 경로, 배럴, "@x", 슬라이스 경계, 트리쉐이킹]
status: published
---

# FSD Public API 규칙

## Public API란?

슬라이스(또는 세그먼트)와 외부 코드 사이의 **계약(contract)**이자 **관문**이다.  
실제로는 `index.ts` 파일에 re-export로 구현한다.

```ts
// pages/auth/index.ts
export { LoginPage } from "./ui/LoginPage";
export { RegisterPage } from "./ui/RegisterPage";
```

## Public API 규칙

> **슬라이스 외부 코드는 반드시 `index.ts`만 참조해야 한다. 내부 파일 직접 접근 금지.**

```ts
// ✅ 올바른 import
import { LoginPage } from "@/pages/auth";

// ❌ 잘못된 import — 내부 파일 직접 접근
import { LoginPage } from "@/pages/auth/ui/LoginPage";
```

## 세그먼트에는 배럴 index.ts를 두지 않는다

> **`index.ts`(배럴 public API)는 *슬라이스*의 계약이다. 세그먼트(`ui`/`model`/`api` 등)에는 배럴 `index.ts`를 두지 않는다.**

슬라이스의 public API(`index.ts`)는 세그먼트 *파일*에서 **직접** re-export한다. 세그먼트 배럴(`./ui`, `./model`)을 경유하지 않는다.

```ts
// pages/auth/index.ts
// ✅ 세그먼트 파일에서 직접 re-export
export { LoginPage } from "./ui/login-page";
export type { AuthUser } from "./model/auth-user";

// ❌ 세그먼트 배럴(ui/index.ts, model/index.ts)을 만들고 그걸 경유
export { LoginPage } from "./ui";      // ./ui/index.ts 라는 세그먼트 배럴 전제 → 금지
```

이유:
- 세그먼트 배럴은 슬라이스 안에 *두 번째 관문*을 만들어 "어디가 진짜 계약인가"를 흐린다.
- 슬라이스 **내부** 파일끼리는 어차피 전체 경로 상대 import를 써야 한다(순환 import 방지, 위 "Slice 내부에서의 Import 규칙"). 그러니 세그먼트 배럴은 내부에서도 불필요하다.

### app·shared 레이어 (슬라이스 없음)

`app`과 `shared`는 **슬라이스가 없는 세그먼트-only 레이어**다(→ `rules/layers.md`). 따라서 *슬라이스형 배럴* `index.ts`(예: `shared/ui/index.ts`, `app/index.ts`)를 두지 않는다.

- 외부에서 필요한 단위로 **직접 import**한다: `import { Card } from "@/shared/ui/card"`.
- `shared/ui`처럼 항목이 많은 세그먼트는 *항목(컴포넌트) 폴더 단위*의 `index.ts`만 둔다(→ 아래 "shared 세그먼트의 번들 최적화"). 세그먼트 루트(`shared/ui/index.ts`)에 모아 배럴하지 않는다.

```
shared/ui/
├── card/
│   ├── card.tsx
│   └── index.ts     ✅ 컴포넌트 폴더 단위 public API
└── index.ts         ❌ 세그먼트 루트 배럴 — 두지 않음
```

## 좋은 Public API 3원칙

1. **리팩토링 보호**: 슬라이스 내부 구조 변경이 외부에 영향 없어야 함
2. **파괴적 변경 명시**: 동작이 크게 바뀌면 public API도 변경되어야 함
3. **최소 노출**: 필요한 것만 export

## 흔한 실수: Wildcard Export

```ts
// ❌ 절대 금지
export * from "./ui/Comment";
export * from "./model/comments";
```

문제:
- 인터페이스가 무엇인지 파악 불가 (discoverability 저하)
- 내부 구현이 실수로 외부에 노출될 수 있음
- 리팩토링 어려움

## Slice 내부에서의 Import 규칙

> **슬라이스 식별**: `index.ts`가 존재하는 폴더가 슬라이스이고, 그 하위는 세그먼트로만 구성된다 (자세한 식별 룰은 `rules/slices.md` "식별 룰" 절 참고).

슬라이스 **내부** 파일 간에는 순환 import를 피하기 위해:

```ts
// pages/home/ui/HomePage.jsx

// ❌ 잘못된 방식 — 같은 슬라이스의 index를 참조 (순환 import 발생)
import { loadUserStatistics } from "../";

// ✅ 올바른 방식 — 내부에서는 전체 경로로 상대 import
import { loadUserStatistics } from "../api/loadUserStatistics";
```

**규칙**:
- **같은 슬라이스 내부**: 상대 경로 + 전체 경로 사용
- **다른 슬라이스 참조**: 절대 경로(alias) + index.ts 경유

## @x 노테이션 (Cross-import용 특수 Public API)

엔티티 간 참조가 필요할 때만 사용. `entities/` 레이어에만 권장.

```
entities/
├── artist/
│   ├── @x/
│   │   └── song.ts   ← song 슬라이스 전용 public API
│   └── index.ts      ← 일반 public API
└── song/
    └── ...
```

```ts
// entities/artist/model/artist.ts
import type { Song } from "entities/song/@x/artist";  // ✅

// entities/song/@x/artist.ts
export type { Song } from "../model/song";
```

## shared 세그먼트의 번들 최적화

`shared/ui` 처럼 파일이 많은 세그먼트는 단일 index.ts가 트리쉐이킹을 방해할 수 있다.  
무거운 의존성이 있다면 항목별 index 분리:

```
shared/
└── ui/
    ├── button/
    │   └── index.ts   ← 개별 index
    ├── text-field/
    │   └── index.ts   ← 개별 index
    └── carousel/
        └── index.ts   ← 개별 index (무거운 의존성 격리)
```

```ts
// 소비하는 쪽
import { Button } from "@/shared/ui/button";
import { TextField } from "@/shared/ui/text-field";
```

## Steiger로 자동 검사

Public API 규칙 위반을 자동으로 잡으려면 [Steiger](https://github.com/feature-sliced/steiger) 아키텍처 린터를 사용하라.

## Relations

- applies-to :: [[slices]]
- depends-on :: [[slices]]
- see-also :: [[segments]]
- see-also :: [[cross-imports]]
