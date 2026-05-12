# fsd skill

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6?logo=anthropic&logoColor=white)](https://claude.ai/code)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-000000?logo=cursor&logoColor=white)](https://cursor.com)
[![FSD](https://img.shields.io/badge/Feature--Sliced%20Design-Architecture-4A90E2)](https://feature-sliced.design)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Feature-Sliced Design 아키텍처 전문 AI 스킬 — 레이어 선택, 파일 배치, import 규칙, FSD 준수 검토를 AI에게 위임할 수 있습니다.

---

## 개요

`/fsd` 스킬은 FSD 아키텍처를 적용 중인 프로젝트에서 개발자가 마주치는 구조적 질문에 즉시 답변합니다. 질문 유형을 자동으로 분류해 관련 규칙 파일을 동적으로 로드하고, 파일 트리 예시와 함께 명확한 권고를 제시합니다.

```
"어느 레이어에 넣어야 해?" → 레이어 결정
"이 import 맞아?"          → import 규칙 검증
"FSD 구조 리뷰해줘"         → 준수 여부 체크리스트 실행
```

---

## 트리거

| 입력 예시 | 동작 |
|----------|------|
| `FSD 적용해줘` | 현재 코드/파일에 FSD 구조 권고 |
| `어느 레이어에 넣어야 해?` | 레이어 결정 가이드 |
| `FSD 확인해줘` | 기존 구조 준수 여부 검토 |
| `FSD 리뷰해줘` | 전체 FSD 체크리스트 실행 |

---

## 파일 구조

```
fsd/
├── SKILL.md              # 스킬 실행 프롬프트 (질문 분류 → 룰 로드 → 답변)
├── LEARN.md              # FSD 빠른 학습 가이드 (컨텍스트 기반 개발자 학습용)
└── rules/
    ├── 00-overview.md    # FSD 핵심 개념 요약, 의사결정 트리
    ├── 01-layers.md      # 6개 레이어 정의 및 import 규칙
    ├── 02-slices-segments.md  # 슬라이스·세그먼트·mapper 설계 규칙
    ├── 03-public-api.md  # index.ts Public API 패턴
    ├── 04-cross-imports.md    # 레이어 간 cross-import 해결
    ├── 05-entities.md    # entities 과설계 방지
    └── 06-tanstack-query.md   # TanStack Query 배치 규칙
```

---

## FSD 핵심 개념

```
src/
├── app/       ← 앱 전체 초기화 (라우터, 프로바이더, 글로벌 스타일)
├── pages/     ← 각 페이지/화면
├── widgets/   ← 여러 페이지에서 재사용되는 독립적인 대형 UI 블록
├── features/  ← 재사용되는 비즈니스 기능 (사용자 행동)
├── entities/  ← 비즈니스 도메인 모델 (User, Product, Order...)
└── shared/    ← 도메인 무관한 공통 코드 (UI 킷, API 클라이언트, 유틸)
```

**핵심 3원칙**
- 하위 레이어는 상위 레이어를 참조할 수 없습니다
- 같은 레이어의 슬라이스끼리는 직접 참조할 수 없습니다
- 슬라이스 외부는 반드시 `index.ts`(Public API)를 통해서만 접근해야 합니다

---

## 동작 방식

스킬은 3단계로 실행됩니다.

**1단계 — 질문 분류**

| 관심사 | 트리거 키워드 | 로드할 파일 |
|--------|-------------|------------|
| 레이어 선택 | "어디에", "어느 레이어", "배치" | `rules/01-layers.md` |
| 슬라이스·세그먼트 | "슬라이스", "폴더 구조", "DTO" | `rules/02-slices-segments.md` |
| Public API | "index.ts", "export", "배럴" | `rules/03-public-api.md` |
| Cross-import | "cross-import", "순환 참조" | `rules/04-cross-imports.md` |
| Entities 설계 | "entities", "엔티티", "과도한" | `rules/05-entities.md` |
| TanStack Query | "useQuery", "queryKey", "tanstack" | `rules/06-tanstack-query.md` |
| 전반적 개요 | (분류 불가 또는 광범위한 질문) | `rules/00-overview.md` |

**2단계 — 룰 파일 로드** (1~3개, 필요한 것만)

**3단계 — 권고 제시** (규칙 인용 + 파일 트리 예시)

---

## 공식 문서 대비 보완된 내용

공식 FSD 문서는 원칙과 구조를 정의합니다. 이 스킬은 실전에서 자주 마주치는 판단 영역을 추가로 다룹니다.

### 세그먼트 내부 의존성 방향

공식 문서는 레이어 간 방향만 규정합니다. 이 스킬은 슬라이스 내부에서도 단방향 규칙을 명시합니다.

```
ui → model → api
```

`api/`는 슬라이스 내 다른 세그먼트를 참조할 수 없습니다.

### DTO / Mapper 처리 3단계

"DTO와 도메인 모델이 얼마나 다른가"를 기준으로 처리 위치를 결정합니다.

| 상황 | 처리 위치 |
|------|---------|
| 이름만 어색함 | `shared/api/` re-export |
| 형태가 다름 (snake_case 등) | `shared/api/` 정규화 mapper |
| 개념 자체가 다름 | 슬라이스 `model/` 도메인 mapper |

Mapper는 `api/`가 아닌 `model/`에 두어야 합니다. `model → api` 의존성 방향을 지키기 위해서입니다. (`api/`에 mapper를 두면 `api → model` 방향 위반이 발생합니다.)

### 페이지네이션·에러 처리 패턴

- `Paginated<T>` 범용 인터페이스를 `shared/model/`에 정의하고, `queryFn` 안에서 아이템 매핑과 함께 처리합니다.
- 에러는 **범용 HTTP 에러** (`app/providers/QueryCache.onError`)와 **비즈니스 의미를 가진 에러** (슬라이스 `model/useMutation onError`)로 분리합니다.

### Cross-import 해결 4가지 전략

공식 문서는 cross-import 금지만 언급합니다. 이 스킬은 상황별 해결 전략을 구체적 코드와 함께 제시합니다.

| 전략 | 적용 상황 |
|------|---------|
| **A. 슬라이스 병합** | 두 슬라이스가 항상 함께 변경될 때 |
| **B. entities로 내리기** | 여러 features가 공유하는 도메인 로직 |
| **C. 상위 레이어에서 조합** | React Render Props / Vue Slots로 IoC 적용 |
| **D. Public API 통해서만** | 위 전략이 맞지 않을 때의 최후 수단 |

### `shared/ui` 번들 최적화

무거운 의존성을 가진 컴포넌트는 단일 `index.ts` 대신 컴포넌트별 개별 `index.ts`로 분리해 트리쉐이킹이 방해받지 않도록 합니다.

```
shared/ui/
├── button/index.ts
├── text-field/index.ts
└── carousel/index.ts   ← 무거운 의존성 격리
```

### TanStack Query 통합 가이드 (`rules/06-tanstack-query.md`)

공식 FSD 문서에는 TanStack Query 관련 내용이 없습니다. 이 스킬이 독자적으로 다루는 영역입니다.

- **Thin client vs Thick client** 구분으로 `queryOptions` 위치를 결정합니다
  - Thin: DTO ≈ 도메인 모델 → `api/` 또는 `shared/api/`
  - Thick: DTO ≠ 도메인 모델 → `model/` (queryFn 안에서 mapper 적용)
- **캐시에는 도메인 모델을 저장** — 앱 내부가 DTO 구조를 알 필요가 없어집니다
- **Query Factory 패턴** (`productQueries.detail(id)`)으로 queryKey를 한 곳에서 관리합니다
- **낙관적 업데이트**를 DTO 필드 없이 도메인 언어로 표현합니다
- **QueryClient·Suspense 전역 설정** → `app/providers/` 배치

---

## FSD 준수 체크리스트

코드 리뷰 시 아래 항목을 자동으로 확인합니다.

- [ ] 레이어 순서 준수 (`app → pages → widgets → features → entities → shared`)
- [ ] 상위 레이어 import 없음 (하위 레이어는 상위 참조 불가)
- [ ] 같은 레이어 간 cross-import 없음 (entities의 `@x` 제외)
- [ ] Public API가 `index.ts`로 노출됨
- [ ] 세그먼트 이름이 목적을 기술함 (`components`, `hooks`, `types` 금지)
- [ ] `shared`, `app` 레이어는 슬라이스 없이 세그먼트 직접 배치

---

## 설치 방법

### Claude Code

```bash
# ~/.claude/settings.json 의 skills 경로에 등록
git clone <this-repo> ~/.ai-agent/skills/fsd
```

`~/.claude/settings.json`:
```json
{
  "skills": [
    {
      "name": "fsd",
      "description": "Feature-Sliced Design architectural guidance",
      "skillFile": "~/.ai-agent/skills/fsd/SKILL.md"
    }
  ]
}
```

### Cursor

`SKILL.md`를 Cursor Rules로 등록하거나, `.cursor/rules/fsd.mdc`에 링크합니다.

---

## 참고 자료

- [Feature-Sliced Design 공식 문서](https://feature-sliced.design) — 아키텍처 명세
- [FSD 공식 예시 모음](https://github.com/feature-sliced/examples) — 실제 프로젝트 적용 사례
- [Steiger](https://github.com/feature-sliced/steiger) — FSD 규칙 위반 자동 감지 린터
- [FSD 슬랙 커뮤니티](https://feature-sliced.design/community) — 질문 및 토론
