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
- **`index.ts`가 존재하는 폴더 = 슬라이스**
- 슬라이스의 하위는 세그먼트로만 구성된다 (슬라이스 안에 또 다른 슬라이스가 들어가지 않는다)
- 그룹 폴더(아래 "슬라이스 그룹핑" 참고)는 `index.ts`를 갖지 않으므로 슬라이스가 아니다 — 슬라이스 묶음일 뿐
- 이 식별 룰 덕분에 *외부에서 참조*하는 것(절대 경로 + `index.ts`)과 *내부에서 참조*하는 것(상대 경로)의 경계가 명확해진다 (→ `rules/public-api.md`)

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

세그먼트(ui/api/model 등)의 책임 정의와 결정 룰은 → `rules/segments.md`.

## 슬라이스 크기 기준

- **너무 작음**: 한 파일짜리 슬라이스 → shared나 상위 슬라이스로 병합 고려
- **너무 큼**: 무관한 기능이 한 슬라이스에 → 슬라이스 분리 고려
- **판단 기준**: 팀원이 이 슬라이스를 찾을 때 직관적으로 이름을 떠올릴 수 있는가?

## Relations

- defined-by :: [[overview]]
- part-of :: [[layers]]
- has-part :: [[segments]]
- required-by :: [[segments]]
- required-by :: [[public-api]]
- required-by :: [[cross-imports]]
- applies-to :: [[entities]]
