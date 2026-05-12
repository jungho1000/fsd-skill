# fsd skill

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6?logo=anthropic&logoColor=white)](https://claude.ai/code)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-000000?logo=cursor&logoColor=white)](https://cursor.com)
[![FSD](https://img.shields.io/badge/Feature--Sliced%20Design-Architecture-4A90E2)](https://feature-sliced.design)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Feature-Sliced Design 아키텍처 전문 AI 스킬 — 레이어 선택, 파일 배치, import 규칙, FSD 준수 검토를 AI에게 위임한다.

---

## 개요

`/fsd` 스킬은 FSD 아키텍처를 적용 중인 프로젝트에서 개발자가 마주치는 구조적 질문에 즉시 답변한다. 질문 유형을 자동으로 분류해 관련 규칙 파일을 동적으로 로드하고, 파일 트리 예시와 함께 명확한 권고를 제시한다.

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
- 하위 레이어는 상위 레이어를 참조할 수 없다
- 같은 레이어의 슬라이스끼리는 직접 참조할 수 없다
- 슬라이스 외부는 반드시 `index.ts`(Public API)를 통해서만 접근한다

---

## 동작 방식

스킬은 3단계로 실행된다.

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

## FSD 준수 체크리스트

코드 리뷰 시 아래 항목을 자동으로 확인한다.

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

`SKILL.md`를 Cursor Rules로 등록하거나, `.cursor/rules/fsd.mdc`에 링크한다.

---

## 참고 자료

- [Feature-Sliced Design 공식 문서](https://feature-sliced.design) — 아키텍처 명세
- [FSD 공식 예시 모음](https://github.com/feature-sliced/examples) — 실제 프로젝트 적용 사례
- [Steiger](https://github.com/feature-sliced/steiger) — FSD 규칙 위반 자동 감지 린터
- [FSD 슬랙 커뮤니티](https://feature-sliced.design/community) — 질문 및 토론
