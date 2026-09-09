# _drafts

`teams/` 밖이라 런타임이 읽지 않는 자리다. 선언은 남기되 아직 뜨게 할 수 없는 것을 둔다.

## apm

APM 조사 팀의 위임 패턴 예시다 (`master` → `slowdown`, `output_schema` 로 판정 객체).
`teams/` 로 되돌리려면 두 가지가 먼저 필요하다.

1. **도구가 실재해야 한다.** 지금 `scan_metrics`·`transaction_delta`·`wait_breakdown` 를
   가리키는데 AI Backend(argus)에 그 이름의 도구가 없다. 런타임은 기동 시 모든 에이전트를
   묶어보므로(요청 시점까지 미루면 프로세스는 건강한데 모든 조사가 500 이 된다) 없는 도구를
   가리키는 선언 하나가 **팀 전체를 뜨지 못하게 한다** — 실제로 그래서 여기로 내렸다.
2. **`output_schema` 를 쓸 수 있어야 한다.** 백엔드의 LLM 게이트웨이가 아직
   `response_format: json_schema` 를 받지 않는다(400). 채팅으로 답하는 에이전트는 스키마
   없이 선언한다.
