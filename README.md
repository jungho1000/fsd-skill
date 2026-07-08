# fsd skill

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6?logo=anthropic&logoColor=white)](https://claude.ai/code)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-000000?logo=cursor&logoColor=white)](https://cursor.com)
[![FSD](https://img.shields.io/badge/Feature--Sliced%20Design-Architecture-4A90E2)](https://feature-sliced.design)

Feature-Sliced Design 아키텍처 결정을 AI에게 위임하는 스킬입니다.

---

## 왜 필요한가요?

FSD는 원칙이 명확합니다. 그런데 막상 코드를 짤 때는 판단이 필요한 순간이 끊임없이 생깁니다.

- 이 코드는 `features`인가요, `entities`인가요?
- mapper는 `api/`에 두나요, `model/`에 두나요?
- 세그먼트에도 `index.ts`(공개 API)를 둬야 하나요?
- cross-import가 발생했는데 어떻게 풀어야 하나요?
- TanStack Query의 queryOptions는 어디에 위치해야 하나요?

매번 공식 문서를 뒤지는 대신 `/fsd` 스킬에 바로 물어보면 됩니다.

---

## 다루는 범위

- **레이어 선택** — features / entities / widgets / shared 중 어디에 둘지
- **슬라이스** — 식별, 그룹핑, 크기 판단
- **세그먼트 규칙** — 이름 자유, 단방향 의존성, 금지 이름(`utils`, `helpers` 등)
- **Public API** — `index.ts`를 슬라이스와 세그먼트의 캡슐화 경계로 두는 기준
- **Cross-import** — 4가지 해결 전략과 entities의 `@x` 노테이션
- **Entities** — 추출 기준, DTO/Mapper 처리
- **세그먼트 패턴** — 뷰 모델, 값 객체, 정적 리소스 배치
- **TanStack Query** — thin/thick client 구분, Query Factory, 낙관적 업데이트

---

## 공식 문서와의 차이

공식 FSD 문서는 원칙과 구조를 정합니다. 이 스킬은 실전에서 자주 부딪히는 판단 지점을 더 깊이 다룹니다.

- **Public API 설계** — `index.ts`는 슬라이스뿐 아니라 세그먼트에서도 캡슐화가 필요할 때 두는 도구라는 기준, 와일드카드 금지, catch-all과 단일 목적 모듈 구분
- **세그먼트 내부 의존성 방향** (`ui → model → api`)
- **DTO/Mapper 처리 기준** — DTO와 도메인 모델의 차이에 따른 3단계 처리
- **Cross-import 해결 전략** — 금지 원칙 외에 구체적인 4가지 해결 방법
- **TanStack Query 통합** — 공식 문서에 없는 영역. thin/thick client 구분, Query Factory 패턴, 낙관적 업데이트

---

## 사용법

`/fsd`로 직접 부르거나 자연어로 물어보면 됩니다.

- "이 컴포넌트 어느 레이어에 둬야 해?"
- "세그먼트에 `index.ts` 둬야 하나?"
- "FSD 리뷰해줘" — 기존 구조의 규칙 위반 점검

---

## 설치

```bash
npx skills add jungho1000/fsd-skill
```

Claude Code를 재시작하면 `/fsd` 스킬이 활성화됩니다. 업데이트가 필요하면 같은 명령을 다시 실행하세요. 최신 스냅샷을 받아옵니다.

---

## 참고 자료

- [Feature-Sliced Design 공식 문서](https://feature-sliced.design)
- [Steiger](https://github.com/feature-sliced/steiger) — FSD 규칙 위반 자동 감지 린터
- [FSD 공식 예시 모음](https://github.com/feature-sliced/examples)
