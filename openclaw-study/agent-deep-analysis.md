# OpenClaw agent 도메인 심화 분석 — 7개 결정 표면의 동작과 설계 판단

> **분석 글** (단순 설명이 아니라 동작 + 설계 평가). ultracode 멀티에이전트로 agent 도메인의 *빈틈/심화* 7영역을 병렬 매핑 → 정확성 비평 → 종합.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src/`를 변경하지 않아 working tree == 이 SHA). 핵심 파일만 링크, 나머지는 코드 스팬.
> 독자가 [agent-runtime-study.md](./agent-runtime-study.md)의 개요(2중 루프·하니스·툴·모델·컨텍스트·세션)를 안다고 가정하고, 그 위에 얹힌 *결정 표면*의 깊은 동작과 트레이드오프만 다룬다. 짝: [gateway-study.md](./gateway-study.md).

## 0. 이 글의 관점

agent runtime을 "모델을 부르는 루프"로만 보면 놓치는 게 있다. OpenClaw에서 흥미로운 결정은 대부분 **루프 바깥**, 즉 *언제 run을 시작하고, 어디서 줄을 세우고, 실패를 어느 단위로 격리하고, 결과를 정확히 한 번 배달하느냐*에서 일어난다. 이 글은 그 7개 표면을 본다.

관통하는 두 개의 테마:

- **거절보다 큐잉.** 동시성 상한, steering, 승인 — 어느 것도 기본적으로 요청을 버리지 않는다. 모두 "지금은 안 되니 줄을 서라"로 수렴하고, 그 줄을 어떤 자료구조와 lifecycle로 들고 있느냐가 설계의 본질이다.
- **격리 단위의 비대칭.** 같은 사건(예: rate limit 한 건)이 여러 backpressure 층을 *동시에* 건드리는데, 층마다 격리 단위(세션 / 공유 레인 / 프로필+모델)가 다르다. 이 비대칭이 blast radius를 결정한다.

---

## 1. Top-level run 동시성과 lane 스케줄링

### 동작

모든 top-level run은 [`src/process/command-queue.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/process/command-queue.ts)의 **프로세스 싱글톤 lane 레지스트리** 위에서 돈다. 레인 상태(큐·`activeTaskIds: Set<number>`·`maxConcurrent`·`draining`·`generation`)는 `globalThis`의 Symbol 싱글톤 Map에 저장돼 번들 청크가 분열돼도 같은 레인을 공유한다(`:107-161`). 세마포어는 별도 추상화가 아니라 그냥 `activeTaskIds.size < maxConcurrent` 비교다(`:341`). 새로 만들어지는 레인의 기본 상한은 1(`:188-195`).

`drainLane`의 pump는 `while (activeTaskIds.size < maxConcurrent && queue.length>0)`로 슬롯이 빌 때마다 큐에서 꺼내고, 태스크 완료 콜백이 다시 `pump()`를 재귀 호출한다(`:339-397`). **상한 도달 시 5번째 run은 거절되지 않고 무한 대기 큐에 들어간다.** 명시적 reject는 게이트웨이 드레이닝(`GatewayDrainingError`, `:429-431`)과 레인 clear(`CommandLaneClearedError`, `:518-520`) 두 경우뿐.

큐는 FIFO가 아니라 **priority-then-FIFO**다. enqueue 시 priority(foreground=1/normal=0/background=-1)와 단조 sequence로 정렬 삽입한다(`:258-271`). priority는 trigger에서 파생된다 — user/manual=foreground, cron/heartbeat/memory/overflow=background, 그 외 normal(`embedded-agent-runner/run.ts:326-341`). 따라서 사용자 run이 큐에서 cron run을 앞지른다.

run은 **2단 중첩 레인**으로 실행된다. `runEmbeddedAgent`는 먼저 `session:<key>` 레인(기본 maxConcurrent=1)으로 enqueue한 뒤, *그 안에서* 글로벌 레인(Main, =4)으로 다시 enqueue한다(`run.ts:514-515,573-575`). session 레인은 어디서도 동시성을 올리지 않으므로 같은 세션 run은 항상 직렬화되고, 서로 다른 세션끼리만 Main의 4슬롯을 두고 경쟁한다 — "세션당 1개 + 전체 4개"의 이중 상한.

레인별 동시성은 `applyGatewayLaneConcurrency`가 게이트웨이 시작(`server.impl.ts:942`)과 config 리로드(`server-reload-handlers.ts:505`)에서 push한다: Main=4, Subagent=8, Cron/CronNested=8(`server-lanes.ts:9-16`). 이 호출 전 또는 임의 이름의 동적 레인(`session:*`, `nested:*`)은 기본 1로 동작. cron run은 `resolveGlobalLane`이 cron→cron-nested로 remap해 self-deadlock을 피한다(`lanes.ts:11-18`).

쿼터 실패 시 동결은 `setCommandLaneConcurrency(lane, 0)`으로 한다(`session-suspension.ts:139-146`). 단 `setCommandLaneConcurrency`는 비-probe 레인에만 `minConcurrent=0`을 허용하고, probe 레인(`auth-probe:*`, `session:probe-*`)은 `minConcurrent=1`로 강제해 **동결 불가**다(`command-queue.ts:415-417`) — 회복용 probe 슬롯은 구조적으로 보호된다. 쿼터 동결의 호출처는 한 곳이 아니다: `run.ts:2544-2548/2977-2981`(top-level=`globalLane=Main`)뿐 아니라 `model-fallback.ts:605`(circuit_open, `laneId=attribution.lane`)·`:1395`에서도 호출돼 동결 레인이 호출처마다 다르다. SIGUSR1 in-process 재시작 후엔 `resetAllLanes`가 generation을 올려 stale active 완료를 무시하고 큐만 보존해 재펌프한다(`:576-593`).

### 설계 평가

**[강점] 세마포어를 정수 비교로 환원한 가벼움.** `Set` 크기 vs 정수 한 줄로 세마포어를 구현했기 때문에 스냅샷·리셋·generation 무효화가 전부 공짜다. in-process restart 내구성(priority/sequence 백필 마이그레이션, `:129-159`)까지 챙긴 건 운영 현실을 진지하게 본 결과다. 약점은 레인이 string 키로만 식별돼 전역 Map에 동적 레인(`session:*`, `nested:*`, `auth-probe:*`)이 쌓이는 것 — 정리가 `clearCommandLane`/`reset` 호출자 책임에 의존해 누수·계측이 거친 부분이다.

**[트레이드오프] "무한 대기 큐"의 양면과 격리 단위.** 상한 초과가 거절이 아니라 무한 대기라는 점은 UX상 친절하지만(요청이 버려지지 않음) backpressure가 큐 자체가 아니라 *소비자*에게만 있다 — heartbeat는 Main 큐 깊이>0이면 run을 스킵하고(`heartbeat-runner.ts:1336`), 나머지는 warnAfterMs 로그·taskTimeout으로만 표현돼 폭주 시 메모리·지연이 무한정 늘 수 있다. 더 날카로운 트레이드오프는 **격리 단위**다: top-level run의 쿼터 실패가 `laneId=Main`을 0슬롯으로 동결하면 한 세션의 rate limit이 *모든* top-level run을 (기본 30분) 묶는다. 호출처별 laneId가 달라 항상 Main을 묶는 건 아니지만, top-level 경로에 한해 공유 레인이 격리 단위라 blast radius가 세션 하나가 아니라 전체다. 한편 session 레인(직렬) 안에서 Main 레인(병렬)을 다시 잡는 구조는 "대화 순서 보장 + 처리량 4"를 동시에 달성하는 영리함이지만, Main이 가득 차면 session 슬롯을 점유한 채 글로벌 대기하는 holding-while-waiting 결합을 남긴다.

---

## 2. Steering — 진행 중 turn에 메시지 주입

### 동작

OpenClaw의 steering은 **하드 인터럽트가 아니라 turn 경계의 협조적 큐잉**이다. 그리고 "steering"이라는 한 단어가 사실 두 개의 별개 메커니즘을 가리킨다.

**(A) 라이브 steering.** [`packages/agent-core/src/agent-loop.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/agent-loop.ts)의 `runLoop`가 매 turn 시작 전(`:271`)과 turn_end 직후(`:373`) `getSteeringMessages()`를 폴링해 drain된 메시지를 다음 assistant 호출 앞에 user 메시지로 push한다(`:286-293`). 진행 중 LLM 스트림은 끊지 않는다 — 새 메시지는 현재 tool 배치/스트림이 끝나야 모델에 보인다. 진짜 중단은 `abort()`(`agent.ts:345`) 별개 경로다. `PendingMessageQueue.drain`은 mode='all'이면 전체를, 'one-at-a-time'이면 맨 앞 1건만 꺼낸다(`agent.ts:169-183`). 라이브 주입 호출자(`commands-steer.ts:144`, `sessions-send-tool.ts:240`)는 'all'을 강제한다.

활성 run은 `ACTIVE_EMBEDDED_RUNS` 프로세스 싱글톤(`run-state.ts:63-103`)으로 찾는다. 없으면 auto-reply의 `reply_run`으로 폴백, 그래도 없으면 `no_active_run` 등 닫힌 사유코드를 돌려준다(`runs.ts:369-432`). `waitForTranscriptCommit=true`면 steer 후 매칭 user `message_end`까지 resolve를 미루고, run이 먼저 끝나면 `cancelQueuedSteeringMessage`로 *그 텍스트만* 정확히 제거해 다음 turn으로 새지 않게 한다(`attempt.queue-message.ts:92-212`). `auto_retry_start`/`compaction_start`는 '아직 살아있다'는 신호라 terminal 취소를 한 tick 미룬다(`:188-206`).

**(B) 결과-배달(부모-자식).** 서브에이전트가 끝나면 delivery payload가 `subagentRuns`에 쌓이고, requester의 *다음* attempt가 시작될 때 `leasePendingAgentSteeringItems`가 pending을 in_progress로 lease하고 `buildMergedAgentSteeringPrompt`로 묶어 그 turn 프롬프트 앞에 prepend한다(`attempt.ts:3807-3837`). 라이브 steering과 달리 '진행 중 turn 인터럽트'가 아니라 '다음 turn 프롬프트 합성'이다. idempotency는 `leaseId=${runId}:agent-steering` 일치 가드로 — ack/release는 `delivery.steeringLeaseId === leaseId`일 때만 상태를 바꾸고(`agent-steering-queue.ts:225,254`), ack 시 payload를 비운다(`:235`). in_progress lease가 5분 넘으면 stale로 회수해 죽은 turn이 결과를 stranding하지 않게 한다(`:34-42`).

배달 전략 순서는 `runSubagentAnnounceDispatch`가 `expectsCompletionMessage`로 가른다: 일반이면 steer-primary→direct, completion handoff면 direct-primary→steer-fallback(`subagent-announce-dispatch.ts:96-131`). 유저 `/steer`는 활성 run이 없으면 `continueWithSteerFallback`으로 일반 prompt로 강등된다(`commands-steer.ts:125-163`).

### 설계 평가

**[강점] turn 경계로 drain을 가둔 것.** 진행 중 tool 배치/스트림을 중간에 끊으면 transcript 정합성과 prompt cache가 깨진다. drain을 turn_start/turn_end 두 지점으로만 제한해 그 위험을 *구조적으로* 제거했고, 진짜 중단은 `abort()`로 분리했다. transcript-commit 대기가 `auto_retry_start`/`compaction_start`를 생존 신호로 인식해 false-cancel 레이스를 막는 디테일(`attempt.queue-message.ts:188-206`)은 long-run에서만 드러나는 미묘한 버그를 미리 겨냥했고 코멘트로 invariant도 남겼다.

**[거친 부분] 용어 충돌과 dead-thread.** (A)와 (B)가 둘 다 "steering"으로 불리지만 메커니즘이 완전히 다르다(즉시 enqueue vs 다음 prompt 합성). 후자에만 lease-id idempotency·stale 회수가 필요하니 분리 자체는 정당하지만, 이름이 겹쳐 인지 비용이 크다. 또 `debounceMs`는 `subagent-announce-delivery.ts:630,1357-1358`에서 조건부로 전달되지만 최종 소비처(핸들 `attempt.ts:3411-3416`, 옵션 타입 `attempt.queue-message.ts:218`)에서 읽지 않아 **drop된다** — "정의·전달되지만 소비처에서 버려지는 dead-thread"(결과적 no-op).

---

## 3. 승인 및 exec 라이프사이클

### 동작

exec 툴이 정책상 승인이 필요하면 게이트웨이에 2단계로 등록한다. **여기서 "2단계"는 사용자 결정 단계가 아니라 register→waitDecision 두 RPC다.** 클라이언트가 `exec.approval.request`를 `twoPhase=true`로 호출하면 서버가 pending map에 ID를 동기 등록하고 `status:"accepted"`를 즉시 응답한 뒤(`approval-shared.ts:469-480`), 별도 `waitDecision`로 최종 결정을 받는다. 이건 exec가 approval-pending을 리턴하기 전에 ID가 서버에 존재함을 보장해 `/approve`가 레이스로 orphan되는 것을 막는다(`bash-tools.exec-approval-request.ts:135-154`).

상태는 `ExecApprovalManager`의 in-memory pending Map이 들고 있다가 resolve/expire 시 `resolvedAtMs`를 찍고 promise를 풀며 15초 grace 후 삭제한다(`exec-approval-manager.ts:134-178`). 이중 resolve는 `resolvedAtMs`로 차단. allow-once는 `consumeAllowOnce`가 `record.decision`을 undefined로 옮겨(`:191-205`) grace window 동안 같은 runId 재생을 원자적으로 막는다.

requester 가시성은 client의 connId/deviceId/clientId를 레코드에 박고(`approval-shared.ts:180-188`), ADMIN scope면 전부, 아니면 deviceId>connId 순으로 매칭해 남의 승인 replay를 막는다(`:122-156`). 배달 라우트(approval client / forwarder·iOS / turn-source 채널) 중 하나도 없으면 `"no-approval-route"`로 즉시 expire한다(`:430-467`) — 아무도 못 보는 승인이 타임아웃까지 매달리지 않게.

배달은 세 경로로 분리된다: forwarder(텍스트 미러, `exec-approval-forwarder.ts:495-532`), iOS APNs push(`exec-approval-ios-push.ts:360-414`), native channel runtime(render adapter). 채널 plugin이 native 승인 UX를 소유하면 `shouldSuppressForwardingFallback`로 generic 텍스트 fallback을 끈다.

async 승인의 결과는 **단발성 elevated handoff**로 되돌린다. 백그라운드 실행이 끝나면 승인-time 권한을 5분 TTL handoff로 등록(approvalId+sessionKey+idempotency 일치 시 단 1회 소비, `followup-state.ts:97-170`)하고, agent RPC로 같은 세션을 resume해 결과 프롬프트를 주입한다(`followup.ts:362-396`). 권한을 영속하지 않고 다음 1턴만 승격시키는 게 핵심. 승인 대기 중 `/new`/`/reset`로 sessionId가 rebind되면, on-disk 세션 스토어를 다시 읽어 expected와 다르면 followup을 드롭한다(`followup.ts:116-142`) — 옛 승인 결과가 새 세션으로 새는 것(#59349) 방지.

hardening: ask 발동 시 argv가 단일 세그먼트로 명확히 바운드된 경우에만 auto-reviewer를 호출하고, `security.audit.suppressions` 같은 민감 명령은 auto-review에서 제외해 무조건 인간 승인으로 강제한다(`exec-host-gateway.ts:449-519`). 결과 파서는 메타데이터가 `gateway id=`/`node=`로 시작하는지 검증해 stdout이 가짜 `Exec denied(...)`를 흉내내는 CWE-841 스푸핑을 차단한다(`exec-approval-result.ts:33-37`).

### 설계 평가

**[강점] "승인 1회"가 디스크에 남지 않는 권한 모델.** in-memory pending + durable allowlist(allow-always만) + 5분 TTL 단발성 handoff의 3층 분리로, async 권한 승격이 영속되지 않는다. `consumeAllowOnce`의 원자적 소비 + 15초 grace 정리는 resolved 잔여 엔트리 replay를 정밀하게 막는 좋은 동시성 설계다. 보안 hardening(출처 검증 정규식, deviceId 우선 바인딩, `plugin:` 접두 예약, 라우트 없으면 expire)이 모두 명시적 위협을 겨냥했고 코멘트로 근거가 남아 있다.

**[트레이드오프/거친 부분] 게이트웨이 라이프사이클 결합과 비대칭 rebind 가드.** pending 승인이 전적으로 게이트웨이 프로세스 메모리에 있어 재시작하면 전부 사라진다. 짧게 사는 운영자 결정엔 합리적이지만 장시간 백그라운드 승인은 프로세스 수명에 묶인다. 더 거친 건 followup 분기다: session-resume→direct fallback→denied 특례로 경로가 많고, 같은 'stale rebind 드롭' 검사가 두 군데서 중복 호출된다. gateway dispatch는 `expectedSessionId`로 거르지만 direct 경로는 그 가드를 못 봐서 별도 on-disk 재조회로 보완하는데, 이 비대칭(왜 한쪽만 스토어를 읽나)이 주석 없이는 직관적이지 않다.

---

## 4. 모델 provider 인증 생애 — cooldown / refresh

### 동작

실패는 **2단으로 분류·게이트**된다. raw 에러는 `classifyFailoverReason` + HTTP status로 13개 closed enum(`AuthProfileFailureReason`, `types.ts:77`)이 되고, `resolveAuthProfileFailureReason`(`auth-profile-failure-policy.ts:14`)이 게이트로 작동한다: `policy==='local'`·`format`·`empty_response`·`server_error`·`providerStarted!==true`인 timeout은 **null을 반환해 세션-로컬로 격리**한다 — format을 공유 프로필 cooldown으로 전파하면 같은 프로필을 쓰는 모든 세션이 죽는 회귀(#77228) 때문. 게이트 통과한 reason만 `markAuthProfileFailure`로 들어간다.

cooldown은 **두 레인**으로 갈린다(`computeNextProfileUsageStats`, `usage.ts:584`). billing/auth_permanent는 'disabled' 레인(지수 백오프, billing 5h~24h cap, auth_permanent 10m~60m cap). 나머지는 'cooldown' 레인(30s→1m→5m cap). errorCount가 강도를 결정한다. 해제 트리거는 3가지: (1) 성공 시 `resetSuccessfulUsageStats`가 모든 윈도우·카운터 전멸(`profiles.ts:284`), (2) 만료 후 `clearExpiredCooldowns`가 ordering 시점에 in-memory로 윈도우 제거 + errorCount=0(circuit half-open→closed), (3) `markAuthProfileFailure` 내부 stale 카운터 리셋. rate_limit/timeout만 `isModelScopedCooldownReason`(`usage-state.ts:20`)으로 **model-scoped** — 같은 프로필의 다른 모델은 bypass(`:82`)되지만, 그 모델이 또 실패하면 scope를 전체로 넓힌다(`usage.ts:659`).

WHAM 프로브(OpenAI OAuth)는 generic 429 backoff를 **provider가 알려준 실제 reset 시각으로 업그레이드**한다. provider==='openai' + reason이 rate_limit/empty/unclassified 등일 때만(`usage.ts:112`) wham/usage를 3s 타임아웃으로 쳐서, 401→12h, 403→24h, used_percent>=100이면 provider-reported reset_at까지 blockedUntil을 차등 적용한다(`usage.ts:172-213`). 프로브 throttle은 provider+agentDir 키별 in-process Map(30s 간격, 256 cap, 24h TTL, `model-fallback.ts:982`).

OAuth refresh는 만료 5분 전(`DEFAULT_OAUTH_REFRESH_MARGIN_MS`)에 트리거되고(`credential-state.ts:53`), provider+profileId 키 in-process queue로 직렬화 + global file-lock 안에서 main-store의 더 새 credential을 먼저 채택(중복 refresh 회피)한 뒤 실제 refresh, 실패 시 다단 fallback한다(`oauth-manager.ts:647-806`). 영속은 per-agent `openclaw-agent.sqlite`의 `auth_profile_store`/`auth_profile_state` 두 JSON blob이며, 읽기는 `coerceAuthProfileState`(`state.ts:156`)가 closed set으로 손상 행을 차단, 모든 mutation은 `updateAuthProfileStoreWithLock`(`store.ts:719`)의 write transaction 안에서 fresh 재로드→updater→저장으로 race를 막는다.

### 설계 평가

**[강점] mutation/predicate 분리 + closed enum.** 'mutation은 usage.ts, predicate는 usage-state.ts' 분리가 명확해 순수 함수가 ordering 핫패스에서 쓰이고 실제 쓰기는 lock 트랜잭션 오너 한 곳에 모인다. closed enum + 전수 coerce로 "impossible state를 unrepresentable하게" 만든 건 루트 AGENTS.md의 discriminated-union 지침과 정확히 일치. WHAM 프로브가 generic backoff를 provider 실제 reset으로 업그레이드하면서 단일-provider 케이스(#90702)를 rate_limit vs subscription_limit 구분으로 처리한 건 회복을 과도하게 늦추지 않는 정밀한 설계다.

**[거친 부분] 3개 병렬 윈도우와 중복 코드.** cooldown이 blockedUntil/cooldownUntil/disabledUntil 3개 병렬 nullable로 표현돼 `resolveProfileUnusableUntil`이 매번 max를 취하고 `clearExpiredCooldowns`가 셋을 독립 검사한다 — AGENTS.md가 경계하는 '병렬 nullable'에 가깝다(다만 세 윈도우가 서로 다른 lifecycle을 가져 단일 union으로 합치기 어려운 실제 도메인 제약이라 정당화 가능). 더 명백한 smell은 `markAuthProfileFailure`/`markAuthProfileBlockedUntil`이 'lock 성공'과 'lock 실패 fallback' 경로를 후크·로깅까지 거의 동일하게 두 번 반복하는 것(`usage.ts:718-828,853-937`) — 한쪽만 고쳐질 위험. throttle Map이 in-process라 다중 워커면 같은 provider를 30s보다 자주 프로브할 수 있다(영속 cooldown 윈도우 덕에 치명적이진 않음).

---

## 5. Provider transport 계층

### 동작

먼저 오해 하나를 제거한다: **`packages/llm-runtime`은 transport 코드가 아니라 레지스트리+facade다.** api id→provider 구현 Map(`api-registry.ts:50-114`)과 `model.api`로 조회해 위임하는 얇은 facade(`stream.ts:22-60`)뿐이고, SSE 파싱·델타 누적 등 실제 동작은 전부 [`src/llm/providers/*`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/llm/providers)에 있다.

**바이트→이벤트 변환은 provider마다 비대칭이다.** Anthropic만 손수 짠 SSE 디코더를 쓴다 — `.asResponse()`로 raw Response를 받아 `iterateSseMessages`가 `body.getReader()`+`TextDecoder`로 청크를 읽고 `consumeLine`(CRLF/CR/LF 모두)으로 분해, 빈 줄에서 flush한다(`anthropic.ts:386-443`). `message_stop` 누락은 에러로 강제(`:483-485`). 반면 OpenAI completions/responses는 SDK의 `.withResponse()`가 주는 async iterable을 `for await`로 돌 뿐, 바이트→이벤트 디코딩은 SDK가 담당한다(`openai-completions.ts:164-169`).

tool-call 델타 누적이 provider 모델 차이를 흡수한다. Anthropic은 `content_block_start`에서 `{partialJson:''}`를 만들고 델타마다 누적·재파싱, `content_block_stop`에서 최종 파싱(`anthropic.ts:642-737`). OpenAI completions는 청크에 명확한 블록 경계가 없어 `streamIndex`와 `id` 두 Map으로 동일 호출을 합친다(`:273-309`). `parseStreamingJson`은 정상파싱→partialParse→repair 후 partialParse 3단 폴백으로 미완성 JSON도 객체를 보장(`json-parse.ts:133-153`).

usage 캐시 토큰 회계가 provider 스펙 차이를 의식적으로 분기한다: Anthropic은 별도 필드라 4개 합산(`anthropic.ts:746-765`), OpenAI completions는 `cached_tokens`를 빼서 중복 계상 방지(`:1180-1202`), Responses는 input이 캐시를 포함하므로 `input - cached`로 분리(`openai-responses-shared.ts:816-823`). prompt 캐시도 두 패러다임을 네이티브하게: Anthropic은 4-breakpoint 예산에서 마지막 user 메시지 뒤로 역방향 탐색해 cache_control을 붙이고(`anthropic.ts:1310-1362`), OpenAI는 `prompt_cache_key`(64자 클램프)를 싣는다(`openai-prompt-cache.ts:5-13`). lazy import 어댑터는 빈 stream을 즉시 반환한 뒤 모듈 로드되면 `forwardStream`으로 펌프해 facade import가 무거운 SDK를 끌어오지 않게 한다(`register-builtins.ts:120-206`).

### 설계 평가

**[트레이드오프] 바이트 레이어 비대칭.** Anthropic 자작 디코더는 SSE 코멘트/CRLF/멀티라인 data를 정확히 처리하고 message_stop 누락을 에러로 강제하는 등 견고하지만, OpenAI 쪽(SDK iterator)과 코드 공유가 없어 SSE 처리 규칙이 한 곳에 모이지 않는다. usage 캐시 회계는 provider 스펙 차이를 주석으로 근거 달아 분기해 정확성에 신경 쓴 흔적이 강하다 — 이 비대칭은 "각 provider 스펙에 충실"이라는 의도적 선택이고, 공통화하면 오히려 한쪽 스펙을 깨뜨릴 위험이 있다.

**[거친 부분] 핫패스 O(n)과 보일러플레이트.** OpenAI `finishBlock`에서 `blocks.indexOf`로 contentIndex를 매번 재계산하는 O(n) 탐색이 스트리밍 핫패스에 남아 있다(`openai-completions.ts:191`). `createLazyStream`/`createLazySimpleStream`이 거의 동일 코드를 반복하고 8 provider×2 함수의 보일러플레이트가 길어 추상화 여지가 있다. 주의: partial 이벤트의 `partial`은 매번 동일 output 객체 참조라 consumer가 보관하면 이후 델타로 변형된다 — 중간 스냅샷이 필요하면 복사해야 한다.

---

## 6. 비-embedded 경로 — CLI 백엔드 vs embedded

### 동작

`isCliProvider` 분기가 외부 에이전트 CLI를 서브프로세스로 띄운다. 이 판정은 단순 provider 문자열이 아니라 **config cliBackends / 런타임 등록 / setup 레지스트리 3중 조회** 결과라(`model-selection-cli.ts:10-24`), 같은 모델이라도 등록 상태에 따라 embedded↔CLI 경로가 바뀐다. CLI면 `runCliAgent`, 아니면 `runEmbeddedAgent`([`attempt-execution.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/command/attempt-execution.ts)`:558-690`).

핵심 데이터 흐름: **한 턴 = 한 서브프로세스.** `prepareCliRunContext`가 백엔드를 resolve하고 bundleMcp loopback 서버를 ensure하며 자식에 주입할 `OPENCLAW_MCP_*` env(토큰/세션키/채널)를 구성한다(`prepare.ts:260-356`). `executePreparedCliRun`이 argv/env/stdin을 빌드해 `supervisor.spawn`으로 자식을 실행하고 stdout JSONL을 스트리밍 파싱한다(`execute.ts:278-891`).

**툴은 인자가 아니라 역방향 loopback MCP다.** 자식 CLI는 자기 네이티브 에이전트 루프를 그대로 돌리되, OpenClaw 툴을 쓸 땐 `127.0.0.1:port/mcp`로 JSON-RPC 콜백한다(`mcp-http.ts:138-306`). claude-cli 백엔드는 `--allowedTools mcp__openclaw__*` 화이트리스트로 OpenClaw 툴만 노출한다(`cli-backend.ts:48`). 즉 데이터 흐름이 반대다 — OpenClaw가 자식을 부르고, 자식이 툴을 위해 OpenClaw를 부른다.

종료 안전성: detached spawn 실패 시 no-detach fallback을 감지해 group-kill로 게이트웨이 자기 프로세스 그룹까지 죽이는 사고를 막고(`child.ts:355-372`, #71662), no-output 워치독이 stall을 시스템이벤트로 알리고 heartbeat를 깨운다(`execute.ts:748-774`). `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`는 backend.env로 들어와도 항상 강제 삭제된다 — 안 그러면 구독이 아닌 호스트-관리 과금 티어로 라우팅된다(`execute.ts:438-441`). input='stdin'이면 write 직후 end로 EOF를 닫아 자식이 인터랙티브 대기에 빠지지 않게 한다(`child.ts:91-98`).

### 설계 평가

**[강점] 두 루프를 한 결과 형태로 수렴.** 외부 CLI의 네이티브 루프를 건드리지 않으면서 OpenClaw 툴만 loopback MCP로 주입하고, embedded와 CLI를 같은 `EmbeddedAgentRunResult`로 수렴시킨 게 핵심이다. 툴을 인자/IPC가 아니라 HTTP brokering으로 노출한 결정 덕에 백엔드 동작이 거의 전부 선언적 config(args/resumeArgs/input/output/clearEnv)로 표현돼 anthropic·google이 같은 `execute.ts`를 공유한다. 종료 안전성 분기(#71662, no-output 워치독)는 운영 현실을 정직하게 반영한다.

**[거친 부분] 비대한 execute + 선언적 추상화의 구멍.** `execute.ts`가 891줄로 watchdog·세션재시도·env scrubbing·이미지·스트리밍이 한 함수에 몰려 useResume×jsonl×liveSession×stdin 조합이 추적하기 까다롭다. 더 본질적으로, claude-cli만 `isClaudeCliProvider`로 특수분기(transcript 존재 확인, binding flush 폴링 0/50/150ms, live-session stdio)가 곳곳에 하드코딩돼(`cli-runner.ts:65-67,100-108`) "선언적 백엔드"라는 추상화에 구멍이 있다. transcript 미존재 기반 세션 리셋이 파일시스템 타이밍에 의존하는 것도 거친 부분이다.

---

## 7. 툴 전체 + 서브에이전트 오케스트레이션

### 동작

**카탈로그와 실제 인스턴스가 분리된다.** `tool-catalog.ts`의 `CORE_TOOL_DEFINITIONS`는 ID·섹션·profiles·`includeInOpenClawGroup`만 담고 실행 로직이 전혀 없다(`:60-351`). 실제 모델 노출 툴은 `openclaw-tools.ts`의 `createOpenClawTools`가 옵션·config·채널 가용성에 따라 조립한다(`:246-534`). 따라서 "모델이 보는 툴" = 프로파일 allow/deny ∩ (조립된 인스턴스 + 플러그인 제공 툴). 4개 프로파일 minimal/coding/messaging/full 중 full은 `['*']`(`:363-376`). **카탈로그 등재 ≠ 코어 구현**: exec 구현은 `bash-tools.exec.ts`, `memory_*`/`wiki_*`는 코어가 아니라 memory-core/memory-wiki 플러그인 제공이다. 역할 분리도 명확하다: message=채널전송, sessions_send=보이는 세션에 메시지/임베디드 run, sessions_spawn=새 child run(target/transport/timeout 파라미터 의도적 거부, `sessions-spawn-tool.ts:284-301`).

서브에이전트는 **depth→role→권한**으로 게이트된다. `resolveSubagentCapabilities`가 depth로 role을 결정(0=main, 0<depth<maxSpawnDepth=orchestrator, 그 이상=leaf). leaf는 `controlScope='none'`·`canSpawn=false`(`subagent-capabilities.ts:161-193`). 기본 maxSpawnDepth=1이라 depth-1 자식은 leaf로 굳어 추가 nesting이 막힌다. `spawnSubagentDirect`에서 depth≥maxSpawnDepth거나 activeChildren≥maxChildren(기본5)이면 **즉시 forbidden**(큐 대기가 아님, `subagent-spawn.ts:1164,1174`).

완료 결과는 **SQLite 기반 durable outbox**다. `SubagentRunRecord.delivery`가 상태머신: not_required→pending→in_progress→delivered, 실패 시 failed/suspended, 만료 시 discarded(`subagent-registry.types.ts:47-85`). canonical 저장소는 SQLite(`loadSubagentRegistryFromSqlite`)이며 500ms TTL 캐시로 다른 worker가 active run을 관측한다. 배달은 **이중 경로로 정확히-한-번**을 보장한다: lifecycle 리스너(빠른 경로, `subagent-registry.ts:1091-1181`)가 phase별로 `completeSubagentRunWithRecovery`를 호출하고, 이벤트가 유실되면 60초 sweeper(느린 보정)가 stale active를 세션스토어 reconciliation으로 완료 추론한다(`:863-1058`). `subagent_ended` hook은 `endedHookEmittedAt` + in-flight guard로 run당 정확히 한 번 발화(`subagent-registry-completion.ts:79-139`). `sessions_yield`는 onYield 콜백에 의도만 기록하고 턴을 끝내, 결과가 announce 대신 다음 메시지로 도착하게 한다(`sessions-yield-tool.ts:24-37`).

(`subagent-registry.store.ts`의 JSON `runs.json` 로더는 독립 런타임 reader가 아니라 SQLite 레이어가 소비하는 마이그레이션/import 헬퍼 — SQLite canonical, JSON은 흡수되는 레거시. 단 stat 기반 read-cache로 reachable이라 "완전 dead"는 아닌 마이그레이션 부채.)

### 설계 평가

**[강점] 이중화 + 정확히-한-번.** lifecycle 이벤트(빠름)와 sweeper+reconciliation(느린 보정)의 이중화로 이벤트 유실에도 결과를 잃지 않고, `endedHookEmittedAt`+in-flight guard로 중복도 막는다. spawn 게이트(maxSpawnDepth=1로 자식 즉시 leaf, maxChildrenPerAgent·maxConcurrent 다층 팬아웃 제한)는 무한 nesting을 보수적으로 차단하는 견고한 설계다. 트레이드오프는 명확하다 — 완료 후 결과 가시성이 최대 60초 지연될 수 있다.

**[거친 부분] 표시-정합 갭과 best-effort 롤백.** full 프로파일이 `'*'`라 새 코어 툴이 카탈로그에서 누락돼도 노출은 되지만 group/profile/UI에서 빠지는 표시-정합 위험이 있다. role/scope가 세션 envelope(저장값)와 depth-derived fallback 두 출처에서 결합돼 ACP 재개 시 cross-store 추적이 복잡하다(`subagent-capabilities.ts:282-332`). spawn 실패 롤백은 단계마다 cleanupProvisionalSession/attachment rm/context-engine rollback으로 흩어진 best-effort라, 이미 시작된 run은 cleanup해도 백그라운드에서 끝날 수 있어 "등록 실패=완전 취소"가 보장되지 않는 본질적 거친 면이 있다.

---

## 8. 종합 평가 — 잘한 점 / 위험 / 숨은 가정

### 잘한 점

1. **거절보다 큐잉의 일관성.** 동시성·steering·승인이 모두 "지금 안 되면 줄을 선다"로 통일돼 사용자 요청이 함부로 버려지지 않는다. 그리고 그 줄을 들고 있는 자료구조가 각 도메인에 맞다 — lane은 정수 세마포어, steering은 turn 경계 drain, 승인은 in-memory pending+grace, 서브에이전트는 SQLite durable outbox.
2. **격리 단위를 의식적으로 고른 흔적.** auth cooldown의 model-scoped reason, `resolveAuthProfileFailureReason`의 null-격리(#77228), probe 레인을 freeze에서 보호하는 `minConcurrent=1`(`command-queue.ts:415-417`) — 전부 "이 실패가 누구까지 죽여야 하나"를 코드로 답한 결정이다.
3. **closed enum과 lock 트랜잭션으로 impossible state 차단.** AuthProfileFailureReason 13 enum + 전수 coerce, lane priority enum, delivery 상태머신, 승인 사유코드 — discriminated union으로 잘못된 조합을 표현 불가능하게 만들고 mutation을 lock 오너 한 곳에 모은 게 루트 AGENTS.md 지침과 일관된다.

### 위험

1. **공유-레인 격리의 blast radius.** top-level run의 쿼터 실패가 `laneId=Main`을 0슬롯으로 동결하면 한 세션의 rate limit이 전체 top-level run을 (기본) 30분 묶는다. 호출처마다 laneId가 달라 항상 Main을 묶는 건 아니지만(`model-fallback.ts:605,1395`는 attribution.lane), top-level 경로에 한해 격리 단위가 세션이 아니라 공유 레인이라는 건 변하지 않는다.
2. **무한 대기 큐의 backpressure 부재.** 상한 초과가 거절이 아니라 무한 큐잉이고 backpressure가 소비자(heartbeat 스킵)와 로그/타임아웃에만 있어 폭주 시 메모리·지연이 무한정 늘 수 있다.
3. **in-process 캐시의 다중 워커 비대응.** throttle Map(auth probe)이 프로세스 로컬이라 게이트웨이가 여러 워커로 뜨면 프로브가 30s보다 잦아질 수 있다(영속 cooldown 윈도우 덕에 치명적이진 않음).

### 숨은 가정 / 영역 간 연결

- **단일 게이트웨이 프로세스 가정이 곳곳에 박혀 있다.** lane 싱글톤·`ACTIVE_EMBEDDED_RUNS`·승인 pending Map·probe throttle Map이 전부 `globalThis`/in-process라 멀티-프로세스 게이트웨이에서는 약속이 약해진다. 영속이 필요한 것(서브에이전트 outbox, auth state)만 SQLite로 빠진 게 이 가정의 경계선이다.
- **하나의 실패가 두 격리층을 동시에 친다 (§1↔§4).** rate_limit 1건이 ① `session-suspension`으로 공유 Main 레인 동결(blast radius=전체 top-level)과 ② `auth-profiles/usage`로 프로필별 model-scoped cooldown(blast radius=해당 모델)을 *동시에* 건다. 두 backpressure의 격리 단위가 정반대(거친 공유-레인 vs 정밀 per-model)라, 어느 층에서 풀리느냐에 따라 회복 체감이 크게 달라진다 — 이 비대칭을 모르면 "왜 다른 모델까지 멈췄나" 오진이 나기 쉽다.
- **"steering"이라는 한 단어가 두 메커니즘을 덮는다 (§2↔§7).** 라이브 steering(turn 경계 drain)과 결과-배달 outbox(lease/ack 멱등성)는 같은 이름을 쓰지만 후자에만 lease-id 멱등성·5분 stale 회수가 필요하다 — 인지비용의 근원.
- **"subagent 동시성"도 두 층이다 (§6↔§1).** depth/children 게이트는 즉시 forbidden(거절), Subagent 레인(=8)은 큐 대기. 같은 단어가 거절 vs 대기로 갈리지만 모순은 없다 — 정책 게이트는 빨리 막고, 용량 게이트는 줄을 세운다는 일관된 분업.

**한 줄 결론:** OpenClaw agent 도메인의 진짜 설계는 모델 루프가 아니라 *큐·격리·정확히-한-번 배달*의 조합이고, 그 강점(일관된 큐잉, 의식적 격리 단위, closed-state)과 위험(공유-레인 blast radius, 무한 큐 backpressure, 단일-프로세스 가정)이 같은 뿌리 — "거절하지 않고 어딘가에 상태를 들고 있는다" — 에서 나온다.
