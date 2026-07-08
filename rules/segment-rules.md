---
domain: fsd
topic: segment-rules
triggers: [세그먼트 규칙, 세그먼트 자유, 단방향, segment rule, 필수 세그먼트, ui model api 필수]
status: published
---

# 세그먼트 — 하드 룰

이 문서는 팀에서 합의된 **하드 룰**만 담는다. 구체적 패턴·예시(`ui/model/api`, tier 모델, 매퍼 위치 등)는 [[segments]]를 본다.

## 규칙

세그먼트는 다음 세 가지만 만족하면 된다. 그 외는 자유.

1. **세그먼트 이름·구성은 어떤 레이어에서든 자유**
   - `ui`, `model`, `api`는 *필수* 세그먼트가 아니다.
   - 슬라이스마다 필요한 세그먼트만 두면 된다.

2. **단방향 의존성을 만족해야 한다**
   - 같은 슬라이스 안에서 세그먼트 간 사이클이 있어서는 안 된다 (= 임의의 두 세그먼트 사이에 import 방향은 하나만).
   - 구체적인 방향 가이드는 → [[segments]] 권장 패턴 참고.

3. **세그먼트 이름은 목적을 드러내야 한다**
   - 금지: `components`, `hooks`, `types`, `utils`, `helpers`, `lib`
   - 이유: *기술 타입* 명칭은 안에 무엇이 있는지 알려주지 않는다.

4. **`index.ts`(public API)는 캡슐화 경계 도구다 — 슬라이스든 세그먼트든 감출 내부가 있으면 둔다**
   - 슬라이스: 외부 계약이므로 `index.ts` **필수**.
   - 세그먼트: 내부 구현을 감출 필요가 있으면 `index.ts`로 public API를 **둘 수 있다**(선택). 감출 내부가 없거나 파일이 하나뿐이면 두지 않는다 — 전부 통과시키는 배럴은 관문만 하나 더 만든다.
   - 공통 가드레일: 큐레이팅된 named export만(`export *` 금지). 한 단위 **내부** 파일끼리는 상대경로로 참조해 자기 배럴을 경유하지 않는다(순환 방지).
   - 자세히 → [[public-api]] "`index.ts`는 캡슐화 도구다".

## 이 규칙이 답하지 않는 것

- "그래서 어느 세그먼트에 둬야 하나요?" — [[segments]] 권장 패턴 (`ui/model/api`, tier 모델, 매퍼 위치 등)
- "슬라이스 자체는 어떻게 나누나요?" — [[slices]]
- "레이어는 어떻게 선택하나요?" — [[layers]]

## Relations

- defined-by :: [[overview]]
- depends-on :: [[slices]]
- extended-by :: [[segments]]
- see-also :: [[public-api]]
- see-also :: [[cross-imports]]
