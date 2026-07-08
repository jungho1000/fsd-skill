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

## `index.ts`는 캡슐화 도구다

> **`index.ts`(배럴 public API)의 본질은 *캡슐화 경계*다. 감출 내부가 있는 단위에 두며, 슬라이스인지 세그먼트인지로 갈리지 않는다.**

- **슬라이스**: 외부 계약이므로 `index.ts` **필수**.
- **세그먼트**: 내부 구현을 감출 필요가 있으면 `index.ts`로 public API를 **둘 수 있다**(선택). 파일이 하나뿐이거나 감출 내부가 없으면 두지 않는다 — 전부 그대로 통과시키는 배럴은 관문만 하나 더 만들 뿐이다.
- **공통 가드레일**: 큐레이팅된 named export만(`export *` 금지). 한 단위 **내부** 파일끼리는 상대경로로 참조해 자기 배럴을 경유하지 않는다(순환 방지).

슬라이스 `index.ts`는 세그먼트가 자체 배럴을 가졌으면 그 배럴을, 없으면 세그먼트 파일을 직접 re-export한다 — 둘 다 허용된다.

```ts
// pages/auth/index.ts — 슬라이스 공개 API
export { LoginPage } from "./ui/login-page";   // ✅ 배럴 없는 세그먼트 → 파일에서 직접
export type { AuthUser } from "./model";       // ✅ model이 자체 배럴(공개 API)을 가진 경우
export * from "./ui";                          // ❌ 와일드카드 → 내부 노출·트리쉐이킹 저해
```

### 세그먼트 배럴을 어디에 둘까 — 세그먼트가 하나의 경계인가로 판단

세그먼트에 `index.ts`를 둘지, 어느 깊이에 둘지는 **그 세그먼트가 하나의 캡슐화 경계인가**로 갈린다.

- **단일 목적 모듈**(자체 공개 API가 뚜렷한 응집된 세그먼트 — `shared/mobile-interface`, `shared/auth`, 슬라이스 안의 잘 정의된 `model/` 등): 루트 `index.ts`가 곧 그 모듈의 공개 계약이다. 둔다.
- **catch-all 세그먼트**(성격이 다른 항목을 모으는 `shared/ui` 등): 세그먼트 전체가 하나의 경계가 아니므로 루트 배럴을 두지 않는다. 대신 **항목 폴더 배럴**(`shared/ui/card/index.ts`)로 항목 단위 캡슐화한다 — 트리쉐이킹·discoverability에도 유리(아래 "번들 최적화" 참고).

허용 조건(가드레일):
- **큐레이팅된 named export만.** `export *`(와일드카드)는 금지 — 내부 노출·트리쉐이킹 저해·discoverability 저하.
- 세그먼트 **내부** 파일끼리는 여전히 상대경로로 import한다(자기 배럴 경유 금지, 순환 방지).

```
shared/
├── mobile-interface/
│   ├── mobile-interface.ts
│   ├── contract.ts
│   └── index.ts        ✅ 단일 목적 모듈 — 큐레이팅된 배럴이 곧 공개 계약
└── ui/
    ├── card/
    │   ├── card.tsx
    │   └── index.ts    ✅ 항목 폴더 배럴
    └── index.ts        ❌ 성격이 다른 컴포넌트를 모은 catch-all → 루트 배럴 금지
```

판단 기준: **한 가지 일을 하는 모듈이면 루트 배럴이 곧 그 모듈의 공개 계약**이라 둔다. `shared/ui`처럼 서로 무관한 항목을 담는 catch-all이면 루트 배럴 대신 항목 폴더 배럴만 둔다.

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

> **슬라이스 식별**: 슬라이스는 슬라이스 레이어의 도메인 단위 직속 폴더이며 하위는 세그먼트로만 구성된다. `index.ts` 유무가 아니라 레이어·위치로 식별한다 (자세한 식별 룰은 `rules/slices.md` "식별 룰" 절 참고).

슬라이스 **내부** 파일 간에는 순환 import를 피하기 위해:

```ts
// pages/home/ui/HomePage.jsx

// ❌ 잘못된 방식 — 같은 슬라이스의 index를 참조 (순환 import 발생)
import { loadUserStatistics } from "../";

// ✅ 올바른 방식 — 내부에서는 전체 경로로 상대 import
import { loadUserStatistics } from "../api/loadUserStatistics";
```

**규칙**:
- **같은 세그먼트 내부**: 상대 경로 + 전체 경로 사용. 자기 세그먼트/자기 슬라이스의 `index.ts`는 경유하지 않는다(순환 방지)
- **같은 슬라이스의 다른 세그먼트 참조**: 대상 세그먼트가 자체 배럴을 가졌으면 그 배럴을, 없으면 상대 경로로 참조
- **다른 슬라이스 참조**: 절대 경로(alias) + 슬라이스 `index.ts` 경유

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
