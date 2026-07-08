---
domain: fsd
topic: slices
triggers: [slice, 슬라이스, 도메인 분리, 그룹핑, 슬라이스 식별, index.ts 보유 폴더, 슬라이스 크기]
status: published
---

# FSD 슬라이스 규칙

## 슬라이스 (Slice)

### 정의
- 레이어 안에서 **비즈니스 도메인**으로 코드를 분리하는 단위
- 이름은 표준화되지 않음 — 비즈니스 용어 그대로 사용
- `shared`, `app` 레이어는 슬라이스 없음

### 식별 룰
- 슬라이스는 **슬라이스 레이어(`entities`/`features`/`widgets`/`views`(또는 `pages`))에서만** 존재한다. `shared`·`app`은 슬라이스가 없다 — 이 두 레이어의 직속 폴더는 `index.ts` 유무와 무관하게 세그먼트다.
- 슬라이스 레이어 안에서 슬라이스는 **도메인 단위 직속 폴더**이며, 하위로 세그먼트만 갖고 공개 API(`index.ts`)로 외부에 노출한다.
- **`index.ts`는 슬라이스 판별기가 아니라 캡슐화(공개 API) 도구다.** 슬라이스에는 필수지만, 캡슐화가 필요한 세그먼트(슬라이스 내부든 shared/app이든)에도 큐레이팅된 배럴로 둘 수 있다 (와일드카드 금지 → `rules/public-api.md`).
- 그룹 폴더(아래 "슬라이스 그룹핑" 참고)는 `index.ts`가 없고 하위에 (세그먼트가 아니라) 슬라이스를 담는 슬라이스 묶음이다.
- 이 구분 덕분에 *외부에서 참조*하는 것(절대 경로 + `index.ts`)과 *내부에서 참조*하는 것(상대 경로)의 경계가 유지된다 (→ `rules/public-api.md`)

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

## 전형적인 슬라이스 구조

아래는 `ui/model/api` 패턴 예시 — 세그먼트 이름·구성은 자유다 ([[segment-rules]]).

```
features/
└── user-profile/
    ├── ui/
    │   └── UserProfileCard.tsx    ← 슬라이스 index.ts가 이 파일에서 직접 re-export
    ├── api/
    │   └── updateProfile.ts
    ├── model/
    │   └── profileStore.ts
    └── index.ts            ← 슬라이스의 public API (외부가 참조하는 유일한 경로)
```

세그먼트의 하드 룰은 → [[segment-rules]], 권장 패턴·매퍼 위치·tier 등은 → [[segments]].

## 슬라이스 크기 기준

- **너무 작음**: 한 파일짜리 슬라이스 → shared나 상위 슬라이스로 병합 고려
- **너무 큼**: 무관한 기능이 한 슬라이스에 → 슬라이스 분리 고려
- **판단 기준**: 팀원이 이 슬라이스를 찾을 때 직관적으로 이름을 떠올릴 수 있는가?

## Relations

- defined-by :: [[overview]]
- part-of :: [[layers]]
- has-part :: [[segments]]
- has-part :: [[segment-rules]]
- required-by :: [[segments]]
- required-by :: [[segment-rules]]
- required-by :: [[public-api]]
- required-by :: [[cross-imports]]
- applies-to :: [[entities]]
