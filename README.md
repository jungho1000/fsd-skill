# fsd skill

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6?logo=anthropic&logoColor=white)](https://claude.ai/code)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-000000?logo=cursor&logoColor=white)](https://cursor.com)
[![FSD](https://img.shields.io/badge/Feature--Sliced%20Design-Architecture-4A90E2)](https://feature-sliced.design)

Feature-Sliced Design 아키텍처 결정을 AI에게 위임하는 스킬입니다.

---

## 왜 필요한가요?

FSD는 원칙이 명확하지만, 실제 코드를 작성할 때는 판단이 필요한 순간이 계속 생깁니다.

- 이 코드는 `features`인가요, `entities`인가요?
- mapper는 `api/`에 두나요, `model/`에 두나요?
- cross-import가 발생했는데 어떻게 풀어야 하나요?
- TanStack Query의 queryOptions는 어디에 위치해야 하나요?

이런 질문에 매번 공식 문서를 찾아보는 대신, `/fsd` 스킬에 바로 물어볼 수 있습니다.

---

## 공식 문서와의 차이

공식 FSD 문서는 원칙과 구조를 정의합니다. 이 스킬은 실전에서 자주 마주치는 판단 영역을 추가로 다룹니다.

- **세그먼트 내부 의존성 방향** (`ui → model → api`)
- **DTO/Mapper 처리 기준** — DTO와 도메인 모델의 차이에 따른 3단계 처리
- **Cross-import 해결 전략** — 금지 원칙 외에 구체적인 4가지 해결 방법
- **TanStack Query 통합** — 공식 문서에 없는 영역. Thin/Thick client 구분, Query Factory 패턴, 낙관적 업데이트

---

## 참고 자료

- [Feature-Sliced Design 공식 문서](https://feature-sliced.design)
- [Steiger](https://github.com/feature-sliced/steiger) — FSD 규칙 위반 자동 감지 린터
- [FSD 공식 예시 모음](https://github.com/feature-sliced/examples)
