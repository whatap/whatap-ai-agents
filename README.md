# whatap-ai-agents

와탭 AI 에이전트. **에이전트는 마크다운 한 장**이다.

새 에이전트를 만드는 데 파이썬 코드는 필요 없다 — md 파일 하나를 추가하면 그게
에이전트다. 실행은 [whatap-ai-runtime](https://github.com/whatap/whatap-ai-runtime) 이,
도구의 실제 구현은 **AI Backend** 가 한다. 이 저장소는 "무엇을 하는 에이전트가 어떤
도구를 쓰는가" 만 갖는다.

## 구조

```
whatap_ai_agents/
  teams/                  채팅 에이전트
    apm/                  팀 = 디렉토리
      master.md           팀 진입점 (이 이름은 규약이다)
      slowdown.md         같은 팀의 다른 에이전트
      schemas/            output_schema 가 가리키는 JSON Schema
  v1/extra/teams/         단발 분석(oneshot) 에이전트
    log/  report/  dpm/
```

**이름이 `teams` 인 디렉토리는 전부 카탈로그 루트**이고 런타임이 하나로 합친다. 그래서
어디에 묶어 두든 **에이전트 이름은 바뀌지 않는다** — 이름은 `teams` 아래의 `<팀>/<파일>`
만 본다. 같은 이름이 두 루트에 있으면 기동 시점에 터진다.

도구는 이 저장소에 없다. AI Backend 가 구현하고, 런타임이 시작할 때 백엔드에 목록을
물어 등록한다. md 의 `tools:` 는 **그 도구의 이름을 가리키는 것**이고, 없는 이름을
쓰면 서버가 뜨기 전에 잡힌다.

에이전트 이름은 경로에서 정해진다 — `apm/master.md` 는 `apm.master` 다. frontmatter 에
`name` 이나 `team` 을 쓰지 않는다. 그리고 이 이름이 와탭 대시보드의 트랜잭션 이름
(`[Agent] apm.master`)에 그대로 쓰이므로, **파일 트리가 곧 대시보드 구조다.**

## 에이전트 추가하기

`teams/<팀>/<이름>.md` 를 만든다. 필수는 `description` 한 줄이다.

```markdown
---
description: 이 에이전트가 하는 일. 상위 에이전트가 위임 대상을 고를 때 이 줄을 읽는다.
---
여기부터 아래 전부가 시스템 프롬프트다.
```

파일명은 소문자로 시작하고 소문자·숫자·`_`·`-` 만 쓴다 (`master`, `slow-response`,
`db_lock`). 이 이름이 트랜잭션 이름에 들어가기 때문이다.

`_` 로 시작하는 md 와 `README.md` 는 에이전트로 읽지 않는다 — 초안이나 문서를 둘 자리다.

### 쓸 수 있는 키

| 키 | 기본값 | 설명 |
| --- | --- | --- |
| `description` | **필수** | 이 에이전트가 하는 일 |
| `kind` | `agent` | `agent` 는 스스로 판단해 도구를 쓰는 루프, `workflow` 는 정해진 절차 |
| `subagents` | 없음 | 위임할 에이전트. 같은 팀은 이름만, 다른 팀은 `팀.이름` |
| `tools` | 없음 | 쓸 수 있는 도구 이름. AI Backend 가 가진 도구를 가리킨다 |
| `model` | 런타임 기본 | 이 에이전트만 다른 모델을 쓸 때 |
| `temperature` | 런타임 기본 | 0.0 ~ 2.0 |
| `thinking` | `false` | 추론 토큰 사용. 켜면 느리고 출력이 잘릴 수 있다 |
| `max_output_tokens` | 런타임 기본 | 출력 상한 |
| `max_steps` | `12` | 도구 호출 루프 상한. 무한 루프를 막는다 |
| `timeout_s` | 없음 | 이 에이전트 실행의 제한 시간(초) |
| `output_schema` | 없음 | md 기준 상대 경로의 JSON Schema. 구조화 출력을 강제한다 |
| `prompt_version` | 본문 해시 | 대시보드에서 프롬프트 개정판을 갈라 보는 값 |
| `labels` | 없음 | 대시보드에 붙일 라벨 (`operation_type` 등) |

키를 잘못 쓰면 실행 전에 잡힌다. 오타면 가까운 후보를 알려준다.

```
frontmatter 를 이해할 수 없다
    - subagent: 모르는 키다 - 혹시 'subagents' 인가?
  파일: teams/apm/master.md
```

## 프롬프트를 쓸 때

`teams/apm/master.md` 와 `slowdown.md` 가 참고할 템플릿이다. 두 파일이 공통으로
지키는 규칙 셋이 있고, 이건 취향이 아니라 이 시스템이 틀리지 않기 위한 조건이다.

- **수치를 직접 만들지 않게 한다.** 도구가 돌려준 증거의 값만 인용하고 `evidence_id`
  를 남기게 한다. 기억이나 추정으로 쓴 숫자는 그 자체로 오답이다. 모델은 요약과 상위
  20행만 보므로, 그 밖의 수치는 만들어낸 것이다.
- **절대값으로 정상/비정상을 판정하지 않게 한다.** "응답시간 500ms 라서 느리다" 는
  판정이 아니다. 기준선 대비 delta 와 정상범위로만 말하게 한다.
- **못 찾았으면 못 찾았다고 하게 한다.** 근거 없는 결론보다 낮은 신뢰도와 다음 확인
  항목이 낫다.

## 모델은 논리 이름으로만 쓴다

`model:` 에 실제 모델명(`claude-sonnet-5`, bedrock id …)을 적지 말 것. 백엔드(argus)가
`LLMGW_MODELS` 로 논리 이름 → 실제 모델을 고르고, 그래야 모델 교체·A/B 가 **이 레포 배포
없이** 된다. 실제 이름을 박으면 모델 풀이 다른 배포(온프렘 등)에서 그 에이전트만 404 로
죽는다. 대부분은 `model:` 을 아예 안 쓰는 것이 맞다 — 런타임 기본값으로 간다.

## 개발 환경

런타임을 같이 고치는 흐름이므로 둘 다 로컬 편집 설치로 쓴다.

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -e ../whatap-ai-runtime
pip install -e .
```
