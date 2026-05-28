---
name: fsd
description: Feature-Sliced Design architectural guidance for development decisions — layer selection, file placement, import rules, and FSD compliance review. Triggers on "FSD 적용해줘", "어느 레이어에 넣어야해?", "FSD 확인해줘", "FSD 리뷰해줘".
argument-hint: "[question, file path, or code context]"
---

# Feature-Sliced Design Advisor

FSD 아키텍처 가이드 — 코드 배치, 구조 설계, 규칙 준수 여부 판단.

**Arg:** `$ARGUMENTS` (질문, 파일 경로, 또는 코드 컨텍스트)
**Language:** 모든 사용자 메시지는 한국어로.

---

## Step 1: 질문 분류

`$ARGUMENTS`를 보고 해당하는 FSD 관심사를 파악하라.
아래 매핑 테이블로 어떤 룰 파일을 읽을지 결정한다.

| 관심사 | 트리거 키워드 | 로드할 룰 파일 |
|--------|-------------|--------------|
| 레이어 선택 | "어디에", "어느 레이어", "layer", "배치" | `rules/layers.md` |
| 슬라이스 식별·그룹핑·크기 | "슬라이스", "slice", "도메인 분리", "그룹핑" | `rules/slices.md` |
| 세그먼트·매퍼·DTO·폴더 구조 | "세그먼트", "segment", "ui", "model", "api", "mapper", "DTO", "폴더 구조" | `rules/segments.md` |
| Public API / index | "index.ts", "export", "공개 API", "import 경로", "배럴" | `rules/public-api.md` |
| Cross-import | "같은 레이어", "cross-import", "import 오류", "순환 참조" | `rules/cross-imports.md` |
| Entities 설계 | "entities", "엔티티", "과도한", "CRUD", "단일 엔티티" | `rules/entities.md` |
| TanStack Query | "useQuery", "useMutation", "queryKey", "react-query", "tanstack" | `rules/tanstack-query.md` |
| 전반적 개요 | (분류 불가 또는 광범위한 질문) | `rules/overview.md` |

> 질문이 모호하면 `rules/overview.md`를 먼저 읽어 컨텍스트를 파악한 뒤 추가 파일을 로드한다.

각 룰 파일의 끝에는 `## Relations` 섹션이 있다. 그 안의 `predicate :: [[다른-룰]]` 항목을 따라 *다음에 읽을 룰*을 결정하라. predicate 어휘:

| Predicate | 의미 |
|-----------|------|
| `defines` / `defined-by` | 어떤 룰이 다른 룰에서 쓰는 용어를 정의 |
| `depends-on` / `required-by` | 읽기 선행 의존 (먼저 읽어야 이해됨) |
| `extends` / `extended-by` | 적용 범위·세부를 확장 |
| `applies-to` / `applied-by` | 한 룰이 다른 영역의 코드에 적용됨 |
| `part-of` / `has-part` | 구성 관계 |
| `see-also` | 약한 연관 |

---

## Step 2: 룰 파일 로드

Read 툴로 매핑된 파일을 읽는다:

- 경로: `~/.ai-agent/skills/fsd/rules/`
- **1~3개 파일**만 로드 (과도한 정보 방지)
- 여러 관심사가 겹치면 모두 로드
- 룰 파일을 읽은 뒤 `## Relations` 섹션의 링크를 따라가 *질문에 답하기 위해 추가로 필요한 룰*만 선택적으로 더 로드

---

## Step 3: 컨텍스트에 규칙 적용

1. **대상 코드/파일 확인** — `$ARGUMENTS`에 없으면 사용자에게 요청
2. **규칙 매칭** — 로드한 파일의 구체적 규칙 인용
3. **명확한 권고 제시** — 어느 레이어/슬라이스에 넣어야 하는지, import 경로는 어떻게 해야 하는지
4. **파일 트리 예시** — 권장 구조를 시각적으로 보여줌

---

## Step 4: FSD 준수 체크 (코드 리뷰 시)

기존 코드 구조를 검토할 때 아래 항목을 확인한다:

- [ ] 레이어 순서 준수 (app → pages → widgets → features → entities → shared)
- [ ] 상위 레이어 import 없음 (하위 레이어는 상위를 참조 불가)
- [ ] 같은 레이어 간 cross-import 없음 (entities의 @x 제외)
- [ ] Public API가 `index.ts`로 노출됨
- [ ] 세그먼트 이름이 목적을 기술함 (`components`, `hooks`, `types`는 금지)
- [ ] Shared/App 레이어는 슬라이스 없이 세그먼트 직접 배치

---

## Quick Reference

```
src/
├── app/        # 라우터, 글로벌 스타일, 프로바이더
├── pages/      # 페이지 단위 슬라이스
├── widgets/    # 재사용 대형 UI 블록
├── features/   # 유즈케이스/시나리오 (여러 엔티티 조합·사용자 액션 흐름)
├── entities/   # 단일 도메인 엔티티
└── shared/     # 프레임워크 무관 공통 코드
```

**핵심 규칙:** 하위 레이어는 상위 레이어를 import할 수 없다. 같은 레이어의 슬라이스끼리 직접 import할 수 없다.

## 개념 그래프

룰 간 관계(에이전트 navigation surface):

```
                  overview
                 ↙   ↓   ↘
            layers  slices  segments
              │     ↕  │       ↑
              │     │  └── public-api
              │     │       │
              │     └── cross-imports
              ↓                ↑
          entities ←──── tanstack-query
                         (model 확장)
```

- `overview` 가 모든 룰을 정의 (defines)
- `layers` 가 슬라이스의 컨테이너, `slices` 가 세그먼트의 컨테이너
- `public-api` 와 `cross-imports` 는 슬라이스 경계를 다룸
- `entities` 와 `tanstack-query` 는 특정 영역 심화
- 각 파일의 `## Relations` 섹션에서 정확한 predicate를 확인
