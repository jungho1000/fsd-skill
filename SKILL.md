---
name: fsd
description: Feature-Sliced Design architectural guidance for development decisions — layer selection, file placement, import rules, and FSD compliance review. Triggers on "FSD 적용해줘", "어느 레이어에 넣어야해?", "FSD 확인해줘", "FSD 리뷰해줘".
argument-hint: "[question, file path, or code context]"
---

# Feature-Sliced Design Advisor

FSD 아키텍처 가이드 — 코드 배치, 구조 설계, 규칙 준수 여부 판단.

**Arg:** `$ARGUMENTS` (질문, 파일 경로, 또는 코드 컨텍스트)
**Language:** 모든 사용자 메시지는 한국어로.
**SKILL_DIR:** 이 스킬의 base 디렉터리 (스킬 로드 시 안내된 "Base directory for this skill" 경로). 아래 파일 경로는 모두 이 기준이다.

---

## Step 1: 질문 분류

`$ARGUMENTS`를 보고 해당하는 FSD 관심사를 파악하라.
아래 매핑 테이블로 어떤 룰 파일을 읽을지 결정한다.

| 관심사 | 트리거 키워드 | 로드할 룰 파일 |
|--------|-------------|--------------|
| 레이어 선택 | "어디에", "어느 레이어", "layer", "배치" | `rules/layers.md` |
| 슬라이스 식별·그룹핑·크기 | "슬라이스", "slice", "도메인 분리", "그룹핑" | `rules/slices.md` |
| 세그먼트 하드 룰 (자유·단방향·금지 이름) | "세그먼트 규칙", "필수 세그먼트", "ui/model/api 필수냐", "단방향", "세그먼트 이름 자유" | `rules/segment-rules.md` |
| 세그먼트 권장 패턴·매퍼·DTO·폴더 구조 | "세그먼트", "segment", "ui", "model", "api", "mapper", "DTO", "tier", "폴더 구조" | `rules/segments.md` |
| 정적 리소스(이미지·SVG·폰트·아이콘) 배치 | "정적 리소스", "asset", "이미지", "아이콘", "SVG", "폰트", "public", "assets" | `rules/segments.md` |
| Public API / index | "index.ts", "export", "공개 API", "import 경로", "배럴" | `rules/public-api.md` |
| Cross-import | "같은 레이어 import", "cross-import", "슬라이스 간 import", "순환 참조" | `rules/cross-imports.md` |
| Entities 설계 | "entities", "엔티티", "과도한 entities", "도메인 모델 추출", "단일 엔티티" | `rules/entities.md` |
| TanStack Query | "useQuery", "useMutation", "queryKey", "react-query", "tanstack" | `rules/tanstack-query.md` |
| 전반적 개요 | (분류 불가 또는 광범위한 질문) | `rules/overview.md` |

> 질문이 모호하면 `rules/overview.md`를 먼저 읽어 컨텍스트를 파악한 뒤 추가 파일을 로드한다.
>
> `segment-rules.md`와 `segments.md`는 의도적으로 분리되어 있다. **하드 룰**(팀이 합의한 강제 조건)은 `segment-rules.md`에, **권장 패턴**(흔히 마주치는 결정·예시·매퍼 위치·tier 등)은 `segments.md`에 있다. 모호하면 둘 다 로드한다.

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

- 경로: `<SKILL_DIR>/rules/` (`SKILL_DIR`은 상단 정의 — 오너 홈 경로에 하드코딩하지 않는다)
- **초기에는 1~3개** 파일만 로드 (과도한 정보 방지)
- 여러 관심사가 겹치면 모두 로드
- 로드 후 `## Relations` 섹션의 링크를 따라 *질문에 답하기 위해 추가로 필요한 룰*만 선택적으로 더 로드 (초기 1~3개 제한은 *시작점*에 대한 것이며, Relations 확장은 제한에서 제외)

---

## Step 3: 컨텍스트에 규칙 적용

1. **대상 코드/파일 확인** — `$ARGUMENTS`에 없으면 사용자에게 요청
2. **규칙 매칭** — 로드한 파일의 구체적 규칙 인용
3. **명확한 권고 제시** — 어느 레이어/슬라이스에 넣어야 하는지, import 경로는 어떻게 해야 하는지
4. **파일 트리 예시** — 권장 구조를 시각적으로 보여줌

---

## Step 4: FSD 준수 체크 (코드 리뷰 시)

기존 코드 구조를 검토할 때 아래 항목을 확인한다:

- [ ] 레이어 import 방향 준수 (위 레이어만 아래 레이어를 import — 역방향 금지)
- [ ] 하위 → 상위 레이어 import 없음
- [ ] 같은 레이어 슬라이스끼리 직접 import 없음 (entities의 `@x` 예외)
- [ ] Public API가 `index.ts`로 노출됨
- [ ] 세그먼트 이름이 금지 목록(`components`, `hooks`, `types`, `utils`, `helpers`, `lib`)에 해당하지 않음
- [ ] 슬라이스 안에서 세그먼트 간 사이클 없음 (단방향 의존성)
- [ ] Shared/App 레이어는 슬라이스 없이 세그먼트 직접 배치

---

## Quick Reference

```
src/
├── app/        # 라우터, 글로벌 스타일, 프로바이더
├── pages/      # 페이지/화면 슬라이스 (1:1 매핑 강제 아님 — 비슷한 페이지는 한 슬라이스로 묶을 수 있음)
├── widgets/    # 재사용 대형 UI 블록
├── features/   # 유즈케이스/시나리오 (여러 엔티티 조합·사용자 액션 흐름)
├── entities/   # 단일 도메인 엔티티
└── shared/     # 프레임워크 무관 공통 코드
```

**핵심 규칙:** 하위 레이어는 상위 레이어를 import할 수 없다. 같은 레이어의 슬라이스끼리 직접 import할 수 없다 (entities의 `@x` 예외 — `rules/cross-imports.md`).

각 룰 간 정확한 관계는 룰 파일의 `## Relations` 섹션을 따른다.

---

## 룰에 없는 질문을 받으면

매핑 테이블·Relations 어디에도 닿지 않는 영역의 질문이라면:

1. **직접 추론으로 답하지 않는다.** 룰셋 외 답변은 스킬의 보증 범위를 벗어남.
2. 사용자에게 *룰셋에 없는 영역*임을 먼저 알린다.
3. 사용자가 명시적으로 요청한 경우에만 https://fsd.how 를 `WebFetch` 로 조회한다.
4. 조회 결과로 답한 뒤, 룰셋에 추가가 필요해 보이면 **변경 제안 리포트**만 제시한다 (어떤 룰 파일에 어떤 단락을 추가/수정할지).
5. 파일 수정·커밋·PR 작성은 사용자 명시 승인 후에만.

> 자동 sync·자동 PR 금지. 룰셋은 팀이 합의한 hard rule + 권장 패턴이므로, 외부 문서 변경이 곧바로 룰이 되어서는 안 된다.
