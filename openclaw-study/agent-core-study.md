# OpenClaw 내장 에이전트 루프 (agent-core) — 심장부 편: `prompt()` 안쪽

> ultracode 멀티에이전트(서브에이전트 Opus 4.8 × 9)로 7영역 병렬 매핑 → 정확성 비평 → 종합.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src`/`packages`를 변경하지 않아 working tree == 이 SHA로 검증됨).
> [agent-runtime-study.md](./agent-runtime-study.md)(러너 편)의 속편 — 러너 편이 "실제 turn 반복은 `activeSession.prompt()` 안쪽에서 돈다"에서 멈춘 그 **gotcha를 여는 문서**. 러너 바깥(재시도 루프·하니스·툴정책·컨텍스트 조립)은 안다고 가정.

---

## 0. 이 문서의 위치 — "그 gotcha를 여는 문서"

**핵심 정정 하나부터:** "안쪽 turn 루프"는 `attempt.ts`에도 `activeSession`에도 없다. 진짜 본체는 [`packages/agent-core/src/agent-loop.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/agent-loop.ts)의 **`runLoop`** 이며, 두 진입 경로가 이것을 공유한다:

- `Agent.prompt()` → `runAgentLoop` (agent.ts) — emit sink = `processEvents`
- `AgentHarness`(=`activeSession`) → `runAgentLoop` (agent-harness.ts) — emit sink = `handleAgentEvent`

즉 두 경로는 **같은 `runLoop`을 emit sink만 바꿔** 호출한다. 러너 편의 "`activeSession.prompt()` 안쪽"은 엄밀히는 AgentHarness 경로다.

> **[설계 평가]** 진입 분기를 emit sink 교체 하나로 흡수 — 루프 알고리즘은 순수 함수로 격리되고 "누가 이벤트를 소비하고 어디에 커밋하나"만 주입점으로 남는다.

---

## 1. 한 turn의 해부 — `runLoop` 제어 흐름

`runLoop`(`agent-loop.ts:258-389`)은 **이중 while**이다.

```
바깥 while(true) (:274)                    ← follow-up 큐 있으면 재진입, 없으면 break (:376-385)
└ 안쪽 while(hasMoreToolCalls || pendingMessages.length>0) (:278)
     한 바퀴 = 정확히 1 turn:
     (1) turn_start (:279-283, 최초 turn은 래퍼가 선발행 → 스킵)
     (2) pendingMessages(steering/follow-up) 주입 (:286-293)
     (3) streamAssistantResponse — 모델 1회 호출 (:296)
     (4) 툴콜 → executeToolCalls → 결과 context 재투입 (:313-332)
     (5) turn_end (:334)
```

- **"결과 재투입 → 재호출" = 안쪽 while이 다시 도는 것.** 지속 조건은 `hasMoreToolCalls = !executedToolBatch.terminate`(`:326`). 툴콜 없으면 초기 `false` 그대로 자연 종료.
- `shouldTerminateToolBatch`(`:694-699`)는 **배치 전원이 `terminate===true`일 때만** true — 하나라도 일반 툴이면 계속 돈다.
- **하드코딩 max-turn 상한 없음**(grep `maxTurns`/`maxIterations` 무매치). 종료는 3경로: (a) `stopReason==='error'||'aborted'` 즉시 return(`:306-310`), (b) `shouldStopAfterTurn` 훅 true(`:361-371`), (c) 안팎 소진 후 말미 `agent_end`(`:388`).
- **stopReason은 루프가 만들지 않는다** — LLM 스트림 최종 AssistantMessage에 실려 온다. 정상 `end_turn`은 조기 탈출이 아니라 "툴콜 없음 → 자연 종료"로 처리.
- 스트림이 throw하면 runLoop이 아니라 상위 `.catch`/`pushLoopFailure`(`:157,242-253`)가 종료 이벤트를 합성 — **runLoop 코드만 봐선 이 탈출 경로가 안 보인다.**

> **[설계 평가]** turn 경계가 `turn_start/end`와 정확히 일치하고 "재투입=재순회" 단일 메커니즘으로 툴 루프를 표현. **상한 책임 주체(정정):** 무한 루프 방지는 agent-core가 아니라 **`agent-session.ts`의 `runAgentPrompt` while(§7)** 과 `shouldCompact`가 진다 — attempt.ts도 아니다. agent-core는 정책 중립 메커니즘(훅·terminate·abort)만 제공.

---

## 2. `Agent` / `prompt()` API와 큐

`Agent`([`agent.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/agent.ts)`:202-616`)는 **얇은 상태 래퍼**다.

- **`prompt()`는 turn을 반복하지 않는다** — 입력 정규화 후 `runWithLifecycle` 안에서 `runAgentLoop`을 **단 1회** await(`:434-448`). 반복은 runLoop 내부.
- **두 개의 `PendingMessageQueue`** (둘 다 기본 `'one-at-a-time'`, `:255-256`):
  - **steering** — 진입 직전(`agent-loop.ts:271`) + 매 turn 끝(`:373`) drain → 다음 모델 호출 **앞**에 주입
  - **followUp** — 안쪽 루프 소진 후 바깥에서 drain(`:376`) → `continue`로 재진입
- **동시성 엄격**: `activeRun` 있으면 `prompt()`/`continue()` 즉시 throw(`:376-380`). 병렬성은 run 내부 툴 병렬뿐.
- **`abort()`** = `activeRun.abortController.abort()` 하나(`:345`) — signal이 stream·각 `tool.execute`로 전파.
- **`idle`의 정의** = `agent_end` 이벤트가 아니라 **listener까지 settle된 시점**(`waitForIdle` = `activeRun.promise`).
- `continue()` 함정: 마지막 메시지가 assistant면 steering→followUp drain으로 우회, 둘 다 비면 throw(`:396-410`). 이중 drain 방지 `skipInitialSteeringPoll` 플래그.

> **[설계 평가]** "상태·이벤트·큐는 agent.ts, 알고리즘은 agent-loop.ts 순수 함수" 분리가 명확. steering/followUp를 **주입 시점**으로 구분한 것이 우아하다. 함정: `idle`을 "agent_end = idle"로 오독하기 쉽다(구독자 후처리가 run 수명에 포함됨).

---

## 3. 세션 트리 — append-only JSONL DAG · compaction replay

- **파일당 트리 하나**: 첫 줄 `{type:"session",version:3}` 헤더, 이후 매 줄 `SessionTreeEntry`(**10종 판별 유니온**: message/thinking_level_change/model_change/compaction/branch_summary/custom/custom_message/label/session_info/leaf, `types.ts:454-464`). 쓰기는 append뿐 — 수정/삭제 없음.
- **단일 branch 재구성**: `leafId→root` parentId walk(`storage-base.ts:131-152`). 유효 leaf는 마지막 엔트리가 아니라 **leaf 엔트리의 `targetId` override**(`:42-44`) — undo/navigate가 leaf 엔트리를 append하는 방식이라서.
- **compaction replay**(`session.ts` `buildSessionContext`): ① 최신 compaction 마커 → **합성 summary** push(`:68-77`) ② 이전 구간은 `firstKeptEntryId` 만난 시점부터 replay(`:83-92`) ③ **이후 신규 branch 엔트리 전부 replay(`:93-95`)** ← 정정: 범위는 `:83-95`.
- ⚠️ **silent context loss(비평 발굴)**: `foundFirstKept`가 끝까지 false면 compaction 이전 tail이 **통째로 드롭**되고 요약만 남는다 — fork/navigate 엣지에서 조용한 컨텍스트 손실 가능.
- **undo/branch = 엔트리 추가**: `navigateTree()→moveTo(newLeafId)`가 leaf 엔트리 append(+선택적 `branch_summary`). `fork()`는 별도 파일 복사 + `parentSession` 지시.
- **transcript commit 시점**: LLM `message_end`에서 곧바로 `session.appendMessage`(§6). 하니스발 마커(model/thinking/leaf)는 turn 도중이면 `pendingSessionWrites`에 쌓였다 turn 경계에 flush(`agent-harness.ts:552-579`).
- model 추적은 `model_change` 마커 없이 assistant 메시지의 provider/model에서도 갱신되며 **1패스 전역 스캔이라 compaction 경계와 무관**(`:33-43`).

> **[설계 평가]** undo/branch/compaction 전부 "새 엔트리 추가"로 표현되는 비파괴 모델 — 파일이 항상 재현 가능하고 compaction도 원문을 안 지운다. 트레이드오프: 파일 단조 증가 + open마다 전체 재파싱. (참고: `session-manager`는 코어에 없다 — src/agents 상위 래퍼. 코어의 레포 역할은 `JsonlSessionRepo`.)

---

## 4. 시스템 프롬프트 조립

**범위 정정:** 조립 본체는 agent-core가 아니라 [`src/agents/system-prompt.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/system-prompt.ts)의 `buildAgentSystemPrompt` — 캐시 경계가 루프 성능에 직결돼 포함.

- **캐시 경계 마커** `<!-- OPENCLAW_CACHE_BOUNDARY -->`(`:1250`) 기준 이분:
  - **위 = 안정 프리픽스**(`:1020-1250`): 정체성 → Tooling(고정 `toolOrder`) → Sub-Agent → Safety → Skills/Memory → Workspace → Bootstrap → Project Context
  - **아래 = 동적 서픽스**(`:1254-1329`): 채널/세션별 컨텍스트·Messaging·Voice·Heartbeat·Runtime
- 프로바이더 KV 캐시는 **바이트 동일 프리픽스에만 히트** → 결정성 3중 보강: `CONTEXT_FILE_ORDER`·고정 toolOrder·CRLF/공백 정규화(`prompt-cache-stability.ts`). `stablePrefix`는 sha256 키로 최대 64개 LRU 메모이즈.
- **`HEARTBEAT.md`만 dynamic으로 분류**돼 경계 아래 렌더 — 자주 바뀌는 파일을 프리픽스에서 배제.
- 오버라이드 3축: per-agent config / provider `promptContribution`(stablePrefix·dynamicSuffix·sectionOverrides) / 채널·세션 분기.

> **[설계 평가]** 안정/동적 분리를 "마커 문자열" 단일 앵커로 구현한 실용성. 거친 부분: 해시 입력 목록(`:979-1019`)이 프리픽스 실내용과 **수동 동기화** — 새 안정 섹션 추가 시 키 누락 → stale 캐시 위험. 1300줄 단일 함수.

---

## 5. 툴 실행 생애

**prepare → execute → finalize 3단계:**
- `prepareToolCall`(`agent-loop.ts:715~`): 툴 조회·인자 검증·`beforeToolCall` 훅. 실패는 `kind:"immediate"` 에러 result.
- `executePreparedToolCall`(`:789~`): `tool.execute(signal, onUpdate)`. **throw하면 catch가 `createErrorToolResult`로 감싸** `{result, isError:true}` 반환(`:809-815`) — 개별 툴 실패가 루프를 죽이지 않는다.
- `finalizeExecutedToolCall`(`:829~`): `afterToolCall` 훅이 content/**terminate**/isError 덮어쓰기 가능.

**병렬 vs 순차:**
- 기본 **parallel**(`agent.ts:261`). `toolExecution==="sequential"`이거나 툴 하나라도 `executionMode==="sequential"`이면 순차 강등(`:505-517`).
- 병렬이어도 `tool_execution_start`는 준비 루프에서 **순서대로 전부 선발행**, execute만 동시 → 결과 순서는 `Promise.all` 배열 순서로 보존 = **결정적 트랜스크립트**.
- **abort 비대칭**: 순차는 매 툴 후 `signal.aborted`면 break(`:584-586`). **병렬은 이미 배열에 담긴 클로저를 `Promise.all`로 완주**(`:628-654`) — 취소 신속성은 각 툴이 signal을 존중하는지에 달림.

**terminate 트리거 실전:** 단일 툴로 turn을 끝내려면 `afterToolCall`에서 `terminate:true` 주입 — 실례 `message-tool-terminal.ts`(훅 체이닝 monkey-patch). 배치가 그 툴 하나면 전원-AND 충족 → 종료.

(`runToolLifecycle`(embedded-agent-subscribe)은 루프 **밖** 툴용 이벤트 재현 래퍼 — 루프 내부 훅과 혼동 금지.)

> **[설계 평가]** 병렬 기본 + "결과 순서 = 호출 순서" 보장으로 동시성과 결정적 트랜스크립트를 동시에 취함. 네 실패 경로가 전부 `createErrorToolResult`로 수렴하는 일관성.

---

## 6. 이벤트 스트림 → 게이트웨이 투영

- `streamAssistantResponse`가 LLM `AssistantMessageEvent`(start/text_delta/toolcall/done)를 for-await로 돌며 `message_start`/`message_update`(원본 델타 동봉)/`message_end`로 변환(`:434-491`).
- **배송 3단**: ① emit sink 팬아웃(Agent=`processEvents` 리듀스, Harness=`handleAgentEvent`→`session.appendMessage`→`emitAny`) ② `embedded-agent-subscribe`가 `pendingEventChain`으로 순서 보존, 델타 누적→`stripBlockTags`→`assistantTexts`/`toolMetas` ③ 게이트웨이 이중 경로(runId별 seq 버스 `emitAgentEvent` + 콜백 `onAgentEvent`) → `server-chat` 머지·브로드캐스트.
- **transcript commit = `message_end` 그 순간**(`agent-harness.ts:583`) — 별도 commit 이벤트가 없다. steering의 `steerAndWaitForTranscriptCommit`은 자신이 큐잉한 user 텍스트와 일치하는 `message_end`를 관찰해 커밋 확인. 대기 중 `agent_end`가 와도 즉시 실패시키지 않고 **한 tick defer** — `auto_retry_start`/`compaction_start`(재개 증거)가 뒤따를 수 있어서.

> **[설계 평가]** `message_end`를 commit 신호로 겸용 + 재개 케이스 defer-cancel 흡수가 우아하다. 부담: 같은 논리 이벤트가 두 채널(seq 버스+콜백)로 나가는 이중 배송. **잠재 함정(영역 교차):** §3의 하니스발 마커 지연 커밋(`pendingSessionWrites`)과 steering의 `message_end` 대기가 순서 경합할 수 있다 — 어느 영역도 단독으로 명시하지 않는 지점.

---

## 7. harness · compaction 계약면

- **의존 역전**: agent-core는 provider를 import하지 않고 `AgentCoreRuntimeDeps{streamSimple, completeSimple}`를 **주입**받는다(`runtime-deps.ts:4-45`). concrete 묶음은 `src/agents/runtime/index.ts`의 `openClawAgentCoreRuntime` — 방향은 **src → agent-core**.
- **바깥이 안쪽을 반복 구동하는 계약**(§1 상한의 실체): `AgentSession.runAgentPrompt` = `agent.prompt()` 후 `while(handlePostAgentRun()){ agent.continue() }`(`agent-session.ts:1055-1088`). `handlePostAgentRun`이 retry(`prepareRetry`)와 `checkCompaction`을 게이트. **turn 루프는 코어 안, retry·compaction 재개는 세션이 바깥에서.**
- **compaction 3단 계약**: `prepareCompaction`(cut point) → `compact`(요약 생성) → `appendCompaction`(요약 엔트리 + `agent.state.messages` 재빌드). 기본 `reserveTokens=16384`/`keepRecentTokens=20000`, `shouldCompact` = `contextTokens > contextWindow - reserveTokens`. `findCutPoint`는 뒤에서부터 keepRecent를 채우고, split-turn이면 turn prefix 별도 요약(요약 0.8×/prefix 0.5× reserveTokens 캡). 코어는 순수 Result 반환, src쪽은 throwing 브리지.
- (미검증: context-engine host capability `"compact"`의 프로덕션 소비 지점 — 이번 범위에서 테스트 호출만 확인.)

> **[설계 평가]** turn 루프(순수 메커니즘)와 retry·compaction 재개(세션 정책)의 책임 분리가 명확해 테스트/재사용성이 높고, 의존 역전으로 코어가 provider·정책에서 격리.

---

## 8. 종합 — 강점 / 거친 부분 + 전체 그림 한 바퀴

**강점**
- **단일 메커니즘의 다중 재사용**: "재투입=재순회"(툴 루프) · "message_end=commit"(steering 접점) · "엔트리 추가=undo/branch/compaction"(비파괴 저장).
- **정책 중립성**: 코어는 max-turn·provider·세션 정책을 모른다 — 전부 세션과 훅으로 위임(의존 역전).
- **결정성 우선**: 병렬 툴 결과 순서 보존, 프롬프트 캐시 3중 정규화+경계 분리.

**거친 부분**
- `firstTurn` 비대칭(래퍼 선발행+루프 스킵), 프롬프트 해시 키 수동 동기화, 1300줄 조립 함수.
- 이벤트 이중 배송 순서 정합 + §3 지연 커밋 ↔ §6 message_end 대기의 잠재 경합.
- `firstKeptEntryId` 미발견 시 tail 통째 드롭(조용한 손실), 병렬 abort의 완주 — "정상 경로에선 안 보이는" 엣지들.

**전체 그림 — 게이트웨이 → 러너 → 루프 → 메모리 한 바퀴**
```
① 게이트웨이: 채널 메시지 → server-chat → 러너에 프롬프트
② 러너(attempt.ts+agent-session): 컨텍스트 조립(§4 프롬프트+§3 branch replay)
   → activeSession.prompt() = AgentHarness 경로 진입
③ 루프(runLoop): 모델 스트림(§7 주입) → 툴 3단계(§5) → 재투입
   → steering/followUp drain(§2) — 매 이벤트 §6으로 투영
④ 메모리: message_end마다 JSONL 커밋(§3). 컨텍스트 차면 세션이
   handlePostAgentRun→compaction(§7) 게이트 후 agent.continue()로 재개
⑤ agent_end → §6 이중 배송 → server-chat → 클라이언트 브로드캐스트 — 한 바퀴 완성
```

**핵심 통찰:** `agent-core`는 "turn을 어떻게 도는가"(메커니즘)만 알고, "언제 멈추고·재시도하고·압축하는가"(정책)는 전부 세션이 바깥에서 감싼다. **`prompt()` 괄호 안은 정책 없는 순수 실행 엔진이고, 그 바깥 한 겹(agent-session)이 지능을 담당한다.**

---

### 부록. 검증 메모
- 핀 유효성: `0fc5a57a..HEAD`는 study 문서만 변경 → `src`/`packages` byte-동일(메인 세션 검증).
- 비평 정정 반영: compaction replay 범위 `:83-95`(신규 branch 포함) / `foundFirstKept` 미발견 시 tail 드롭 발굴 / 무한루프 상한 주체 = `agent-session.runAgentPrompt`(attempt.ts 아님) / `shouldTerminateToolBatch` 전원-AND 확증 / session-manager는 코어에 없음(상위 래퍼).
