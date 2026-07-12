# Gateway Q&A — 메서드·에이전트·저장

> 게이트웨이 학습 중 나온 질문 묶음. 본문은 [gateway-study.md](./gateway-study.md). 코드 링크는 `openclaw@0fc5a57a` 핀.

---

## Q1. 왜 "패밀리"가 문자열이야? (method가 왜 문자열?)

와이어 프로토콜에서 **`req.method`가 문자열**이라서 ([`frames.ts:155`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L155) `method: NonEmptyString`).

**왜 문자열인가** — 원격 호출은 **함수 자체를 소켓으로 못 보낸다**(함수는 한 프로세스 메모리에만 존재). 그래서 함수의 *이름*을 보내고 받는 쪽이 이름표로 찾아 실행한다(=RPC의 본질). 그 이름은:
- **직렬화 가능** — req는 WS 위로 JSON(텍스트)로 흐름
- **언어 중립** — 클라가 TS·Swift·Kotlin 제각각이라 공통 계약 필요
- **동적 바인딩** — 런타임에 `이름→핸들러` 맵에서 찾음(lazy 로드·플러그인 확장 가능)
- **확장·가독성** — `sessions.list`·`myplugin.foo`, 접두사 정책(`config.*`→admin)

→ enum/숫자였다면 확장·언어중립·가독성을 잃음. 그래서 **문자열**.

**"패밀리"** = 점(`.`) 앞 이름으로 묶은 메서드 그룹(`sessions.*` → `server-methods/sessions.ts` 한 모듈). 문자열 네임스페이스라 그룹핑·접두사 정책이 가능.

위에는 이제 Claude 답변이긴 한데, prefix로 강제하는 정책상으로 문자열을 채택하지 않았나 하는 생각입니다.

---

## Q2. embedded 에이전트가 따로 있어?

이 질문은 제가 질문 자체를 헷갈린 거 같습니다. ㅎㅎ 모델을 말씀하시는 줄 알고 없다고 말씀드렸는데,
Opencode Primary 에이전트랑 같은 역할을 하는 Embedded가 있다고 합니다. 다만 용어 상으로 좀 헷갈릴 수 있는 부분이
Embedded 에이전트가 아닌 "mode"라고 합니다. Embedded는 방식이지 주체가 아니라고 합니다.

"이 agentId를 가진 에이전트를 embedded 방식으로 돌려"라고 말할 수 있을 것 같습니다.

*default인 main agent는 embedded가 기본 동작 방식이라고 합니다. + 사용자가 외부 하네스를 입혀서도 구동이 가능하다고도 합니다.

`src/agents/embedded-agent-runner/` — 게이트웨이와 **같은 프로세스 안에서** 에이전트 run을 돌리는 러너 ([`run.ts:2`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/embedded-agent-runner/run.ts#L2) *"Top-level embedded-agent run orchestration entrypoint"*).

Agent 런타임 쪽을 봐바야 이 부분은 자세히 알 것 같습니다,

---

## Q3. 에이전트는 몇 대야? 한 대에 여러 모델이 붙어?

- **여러 대 가능** — config `agents.list?: AgentConfig[]`([`types.agents.ts:162`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/config/types.agents.ts#L162)), 각자 `agentId`(`:56`). 설정 없으면 기본 **1대 `main`**.
- **한 에이전트 = 모델 1개(+백업)** — `model?: AgentModelConfig`(`:88`) = `{primary, fallbacks}`. 동시에 여러 모델이 붙는 게 아니라 **활성 primary 1개**로 돌고 실패 시 fallbacks로 대체. (+ per-model 오버라이드 `models?: Record<...>` `:95`, 서브에이전트용 기본 모델 별도 `:138`.)

→ **에이전트 N대(각자 agentId·전용 SQLite·전용 모델 설정) / 각 에이전트는 모델 1개로 동작.**

---

## Q4. 대화 내용은 어디에 저장돼?

**JSONL 파일** — SQLite 아님. `src/transcripts/store.ts:12` *"File-backed transcript session store"* → `transcript.jsonl`. docs도 *"session transcripts stay JSONL"*(docs/logging.md). ([`transcripts/store.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/transcripts/store.ts))

---

## Q5. 세션 정보는 어디에 저장돼?

세션의 라우팅 정보는 => SQLite
세션의 대화 내용 => JSON

- **세션 인덱스 = JSON 파일**: `agents/<agentId>/sessions/sessions.json` ([`config/sessions/paths.ts:37`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/config/sessions/paths.ts#L37), docs/openclaw-agent-runtime.md:69).
- 세션별 transcript는 `sessions/` 디렉토리의 파일.
- **세션 관련 일부 메타만 SQLite**에: `current_conversation_bindings`(라우팅 연속성)·`acp_sessions`·`tui_last_sessions`.

---

## Q6. 그럼 SQLite엔 어떤 데이터(row)가 저장돼?

**운영·제어·인증 같은 "상태/메타"** 

**전역 `state/openclaw.sqlite` (64개 테이블)**:

| 분류 | 테이블 예 |
|---|---|
| 인증·기기 | `device_auth_tokens`·`device_pairing_paired/pending`·`device_identities`·`auth_profile_stores` |
| 노드 | `node_pairing_*`·`node_host_config` |
| 승인 | `exec_approvals_config`·`plugin_binding_approvals` |
| cron·작업 | `cron_jobs`·`cron_run_logs`·`task_runs`·`subagent_runs`·`flow_runs`·`commitments` |
| 전송 큐 | `delivery_queue_entries`·`channel_ingress_events`·`task_delivery_state` |
| 플러그인 | `plugin_state_entries`(KV)·`plugin_blob_entries`·`installed_plugin_index` |
| 미디어 | `media_blobs`·`capture_*` |
| 알림·음성·모델 | `apns_registrations`·`web_push_subscriptions`·`voicewake_*`·`model_capability_cache` |
| 게이트웨이 운영 | `gateway_restart_*`·`state_leases`·`migration_*`·`backup_runs` |
| 라우팅 | `current_conversation_bindings` |

**에이전트별 `openclaw-agent.sqlite` (4개)**: `auth_profile_store`/`auth_profile_state`(모델 인증 프로필)·`cache_entries`(캐시)·`schema_meta`(스키마 버전).
