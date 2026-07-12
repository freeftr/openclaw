# OpenClaw cron·tasks·flows — 에이전트가 스스로 깨어나 일하는 축

이 축의 핵심은 **"시간(스케줄러)이 발화하면 → 격리된 에이전트 턴이 백지 세션으로 깨어나 → 결과를 durable하게 전달하고 → transient 실패만 backoff로 재시도하는" 단일 self-arming 타이머 파이프라인**이며, 잡 정의와 실행 이력은 SQLite로 canonical하게 영속되지만 그 파이프라인이 소비하는 *세션 상태 계층*만 여전히 JSON 파일에 남아 있다는 비대칭이 이 축의 가장 정직한 진실이다.

---

## 📢 발표 길잡이 (읽으며 말할 순서)

**TL;DR (한 문장)**
cron은 "언제(스케줄)"를 소유하는 트리거이고, tasks는 "실행 인스턴스 하나를 어떻게 추적할지(원장)"를, flows(진짜 워크플로는 `src/tasks/task-flow-registry.ts`)는 "다단계 대기·재개"를 소유하며 — 셋 다 상태를 공유 SQLite에 넣지만, cron이 깨우는 격리 에이전트의 *세션 파일*만은 여전히 JSON store다.

**말할 순서 (섹션 흐름 7비트)**
1. §0 재정의 — cron/tasks/`src/flows`/진짜 flow 4개를 한눈에 구분(이름 함정 먼저 깬다)
2. §1 스케줄 모델 — 자연어가 아니라 `at`/`every`/`cron` 판별 유니온
3. §2 서비스·실행 루프 — 이벤트가 아니라 단일 self-arming `setTimeout`
4. §3 격리 에이전트 각성 — `forceNew`가 만드는 "매번 백지 세션"
5. §4 결과 전달·재시도 — fail-loud primary vs fail-soft 실패알림, transient 5분류
6. §5 하트비트·run-log — 하트비트 "트리거"는 여기가 아니다 + SQLite 이력
7. §6 tasks·flows 경계 → §7 영속과 SQLite 정책(핵심 반례) → §8 종합

**예상 질문 & 답**

| 질문 | 답 | 근거 § |
|---|---|---|
| cron과 tasks의 축은 "반복 vs 일회성"인가? | 아니다. **트리거(cron) vs 실행-추적(tasks)** 축이다. cron 매 발화마다 tasks에 원장 행 1개 생성 | §0, §6 |
| `src/flows/`가 워크플로인가? | 아니다. **doctor/onboarding/setup UI**다. 진짜 워크플로는 `src/tasks/task-flow-registry.ts` | §0, §6 |
| cron 상태는 SQLite인가 JSONL인가? | 잡·이력은 **SQLite**(cron_jobs/cron_run_logs). JSONL은 doctor 마이그레이션 전용. **단, 격리 세션 상태는 JSON 파일** | §5, §7 |
| "격리(isolated)"가 실제로 무슨 뜻인가? | `forceNew=true` → 매 실행 새 `randomUUID` 세션, ambient context 버림 = **백지 각성** | §3 |
| 하트비트를 트리거하는 게 `cron/heartbeat-policy.ts`인가? | 아니다. 그건 **전달 억제 정책**. 실제 각성 엔진은 `src/infra/heartbeat-runner.ts` | §5 |
| 스케줄러가 초 단위 정밀한가? | 아니다. 최대 **60초 clamp** + 완료 후 최소 2초 하한(spin 방지) | §2 |

**핵심 그림 하나**: §2의 파이프라인 ASCII 흐름도(스케줄→트리거→예약-persist→격리실행→전달→재시도). 발표 중 이 그림 하나로 전체 축을 관통하라.

---

## 0. 재정의 — cron / tasks / `src/flows` / 진짜 flow 한눈 구분

가장 먼저 깨야 할 것은 **이름의 함정**이다. 네 개념이 이름만 보면 헷갈리지만 실제 축은 다르다.

| 이름 | 위치 | 실제 정체 | 소유하는 것 |
|---|---|---|---|
| **cron** | `src/cron/` | 스케줄러/트리거 | "언제 발화할지" + 자체 재시도·격리실행·타이머 |
| **tasks** | `src/tasks/task-registry.ts` | 백그라운드 실행 **원장(ledger)** | "실행 인스턴스 하나를 어떻게 추적·전달할지" |
| **진짜 flow** | `src/tasks/task-flow-registry.ts` | 다단계 워크플로 상태 머신 | queued/running/**waiting/blocked**/resume 오케스트레이션 |
| `src/flows/` | `src/flows/` | **doctor/onboarding/setup UI** | 워크플로 아님 — 설정·진단 UI flow |

**정정: cron과 tasks는 "반복 vs 일회성" 축이 아니다.** cron=트리거(스케줄), tasks=실행 인스턴스 원장이다. cron이 매번 발화할 때마다 tasks에 새 원장 행 1개를 만든다. 반복성은 cron이, 실행 추적은 tasks가 소유한다.

**정정: `src/flows/`는 워크플로 오케스트레이션이 아니다.** [`src/flows/types.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/flows/types.ts)`:1`이 스스로 "setup, onboarding, doctor flow UIs"라 선언한다. 진짜 워크플로 상태 머신은 [`src/tasks/task-flow-registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/tasks/task-flow-registry.ts)에 있다.

> **[설계 평가]**
> - 강점: "트리거 / 실행-추적 / 오케스트레이션" 3분할이 명확하고, cron이 tasks를 `scopeKind:"system"`·`silent` 원장으로만 소비해 task registry가 트리거 종류를 몰라도 되게 한다.
> - 트레이드오프: `src/flows/`(설정 UI)와 `task-flow-registry`(워크플로)의 이름 충돌이 신규 독자를 정확히 오도한다. 이 비대칭 때문에 파이프라인을 잘못 그리기 쉽다.
> - 거친 부분: **미확인** — `src/tasks/`·`src/flows/` 파일별 라인 주장(task-registry.ts:1678 등)은 이번 검증에서 직접 열지 못했다. heartbeat 영역만 직접 확인.

---

## 1. 스케줄 모델 — 자연어가 아니라 세 판별 유니온

OpenClaw cron 스케줄은 자연어가 아니라 세 가지 discriminated union이다. [`src/cron/types.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/types.ts)`:9-18`을 직접 열어 확인했다:

```
kind: "at"    → { at: string }                          // 1회성 절대시각
kind: "every" → { everyMs: number; anchorMs?: number }  // 순수 간격 (croner 미사용)
kind: "cron"  → { expr; tz?; staggerMs? }               // 표준 cron 표현식 (croner)
```

**정정: `staggerMs`는 `cron` kind에만 있다.** `types.ts:16`을 직접 열어 확인했다 — `at`은 `at`만, `every`는 `everyMs`/`anchorMs`만 갖는다. "0=정확히 스케줄대로"라는 계약(`types.ts:16` 주석)도 cron 전용이다.

**"다음 실행 시각" 계산** — [`src/cron/schedule.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/schedule.ts)`:55-119` `computeNextRunAtMs`가 kind별로 분기한다(직접 확인):

- **at** (`:56-61`): `parseAbsoluteTimeMs`로 절대시각을 얻어 `atMs > nowMs`일 때만 반환. 과거면 `undefined`(비활성).
- **every** (`:64-78`): `everyMs=Math.max(1,floor)`, `anchor+steps*everyMs`. `now<anchor`면 anchor 반환. **croner를 거치지 않는 순수 산술**.
- **cron** (`:80-118`): `resolveCachedCron`으로 croner `Cron` 인스턴스를 `${tz}\u0000${expr}` 키(`:22`, NUL 구분자)로 LRU 캐시(max 512, Map 삽입순서 활용, `:21-41`)한 뒤 `cron.nextRun(now)`. 반환이 now 이하면 **croner의 timezone year-rollback 버그**(예: Asia/Shanghai) workaround로 "다음 초 → 내일 UTC 00:00" 순으로 재시도(`:97-116`).

**정정: 다음 실행 계산에 stagger는 여기 없다.** `computeNextRunAtMs`는 stagger를 적용하지 않는다. stagger 실효 반영은 [`src/cron/service/jobs.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/service/jobs.ts)`:119-145` `computeStaggeredCronNextRunAtMs`에 있다. 즉 `schedule.ts`만 보면 stagger 오프셋이 안 보인다.

**stagger 목적·알고리즘** — 정시(top-of-hour) 다발 잡이 동시에 깨어 서버로 몰리는 thundering-herd 완화다. `isRecurringTopOfHourCronExpr`로 '분=0 + 시필드 wildcard' 반복 정시를 감지하면 기본 5분 창(`DEFAULT_TOP_OF_HOUR_STAGGER_MS=5*60*1000`, 직접 확인)을 부여하고, `resolveStableCronOffsetMs`가 **job id의 sha256 첫 4바이트를 `staggerMs`로 modulo**해 `[0,staggerMs)` 결정론적 오프셋을 만든다(`service/jobs.ts:96-117`). 재시작·프로세스 이동에도 같은 잡은 같은 슬롯을 유지한다.

**타임스탬프 검증** — `validate-timestamp.ts`(직접 확인): 과거는 1분 유예창(`:48`, 생성·검증 레이스 방지) 후 거부, 미래는 10년(`TEN_YEARS_MS`, `:11`) 초과 시 오타 방지 목적으로 거부(`diffMs > TEN_YEARS_MS`, `:61`).

> **[설계 평가]**
> - 강점: 세 형태를 판별 유니온으로 모델링해 impossible state를 배제했고, `every`는 croner 없는 순수 산술로 예측가능·저비용이다. cron 파싱만 외부 라이브러리에 격리.
> - 트레이드오프: croner의 year-rollback 버그를 라이브러리 픽스가 아니라 애플리케이션 다단계 재시도(`schedule.ts:93-116`)로 우회 — croner 업그레이드 시 workaround 유효성 재검증 필요.
> - 거친 부분: 실효 "다음 실행"을 알려면 `schedule.ts`(순수)와 `service/jobs.ts`(stagger) 두 층을 함께 봐야 한다. 관심사 분리이자 인지 비용.

---

## 2. 서비스·실행 루프 — 이벤트가 아니라 단일 self-arming 타이머

cron 서비스는 얇은 파사드(`src/cron/service.ts`)가 mutable state(`service/state.ts`)와 lock된 ops(`service/ops.ts`, `service/timer.ts`)로 위임하는 구조다. 스케줄러는 이벤트가 아니라 **단일 `setTimeout` 기반 self-arming 타이머**로, 매 tick마다 due 잡을 SQLite에서 재로드→예약→worker-pool 실행→결과 persist→재무장한다.

**핵심 파이프라인**([`src/cron/service/timer.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/service/timer.ts)`:759-1002`):

```
                         ┌──────────────────────────────────────────────┐
                         │  armTimer: setTimeout(min(delay, 60_000ms))   │  ← clamp: 최소 분당 1회 각성
                         │  (시계 점프·프로세스 일시정지 복구용)          │
                         └───────────────────┬──────────────────────────┘
                                             ▼
   ┌── onTimer (재진입 방지: state.running=true면 60s 재점검만 걸고 return) ──┐
   │                                                                          │
   │  locked() { forceReload(SQLite) → collectRunnableJobs(due) →             │
   │             각 due 잡에 runningAtMs=now 마커 심고 persist(예약) }         │  ← 예약을 lock 안에서 커밋
   │                             │                                            │     (다른 tick의 중복 시작 차단)
   │              lock 해제 후 worker-pool (min(concurrency, due)개)           │
   │                             ▼                                            │
   │   executeJobCore:  sessionTarget=="main" → heartbeat enqueue             │
   │                    payload.kind=="command" → runCronCommandJob (자식 프로세스)
   │                    payload.kind=="agentTurn" → runIsolatedAgentJob (§3)  │
   │                             ▼                                            │
   │  locked() { 재로드 → applyOutcomeToStoredJob → recomputeNextRuns →       │
   │             persist } → finally: armTimer 재무장 + session-reaper piggyback │
   └──────────────────────────────────────────────────────────────────────────┘
```

**동시 실행 억제 3중 방어**(직접 검증한 상수 기반): (a) `job.state.runningAtMs` 숫자 마커를 disk(SQLite)에 persist — cross-restart 중복 담당, (b) 프로세스 전역 `active-jobs` Set(`resolveGlobalSingleton`으로 모듈 리로드 넘어 공유) — in-process 중복 담당, (c) `locked()` storePath별 뮤텍스로 mutating 연산 직렬화.

**영속=SQLite** — `store.ts`의 load/save가 전부 `openOpenClawStateDatabase()`(공유 state DB)로 라우팅된다(직접 확인: `store.ts:1` 주석 "SQLite plus quarantine sidecars", `:63` "SQLite-backed store"). hot-path `stateOnly` write는 runtime 컬럼만 UPDATE해 사용자 작성 config JSON churn을 막는다(`store.ts:123-124` 주석 직접 확인). `status.storage`는 하드코딩 `"sqlite"`(`state.ts:224` 직접 확인).

> **[설계 평가]**
> - 강점: single self-arming `setTimeout` + 60s clamp로 단순하면서 시계 점프/일시정지에 복원력. 예약(runningAtMs persist)을 lock 안에서 먼저 커밋하고 실행은 lock 밖에서 해 list/status read가 장기 실행 중에도 응답성 유지.
> - 트레이드오프: 3중 방어가 견고하지만 상태 소스 분산. `active-jobs` Set은 프로세스 전역 singleton이라 **단일 gateway 프로세스 가정에 강하게 결합** — 다중 워커에서는 무효.
> - 거친 부분: `timer.ts`가 58KB 단일 파일로 스케줄·실행·재시도·startup-catchup을 모두 담아 CLAUDE.md의 ~700 LOC 분할 가이드를 크게 초과. `#12025`/`#17554`/`#24355` 등 회귀가 개별 주석으로 축적돼 인지 부하 높음.

---

## 3. 격리 에이전트 각성 — `forceNew`가 만드는 "매번 백지 세션"

cron 스케줄러가 시간이 되면 에이전트를 "격리(isolated) 세션"으로 깨워 한 번의 턴을 실행시킨다. `src/cron/isolated-agent.ts`는 얇은 facade이고 실제 오케스트레이션은 `isolated-agent/run.ts`(약 52KB)가 setup→execution→delivery→cleanup 전체를 담당한다.

**세션키 파생 3단계**: `baseSessionKey = input.sessionKey || cron:<jobId>` → `resolveCronAgentSessionKey`가 `agent:<agentId>:cron:<jobId>` 형태의 agentSessionKey 생성 → `cron:`으로 시작하면 `runSessionKey = ${agentSessionKey}:run:${runSessionId}`. per-job 베이스 세션과 per-run 실행 세션이 키 레벨에서 분리된다.

**"격리"의 정확한 의미**(직접 검증): `forceNew = (sessionTarget === "isolated")`를 `resolveCronSession`에 넘긴다. [`src/cron/isolated-agent/session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/isolated-agent/session.ts)`:154-158`에서 `forceNew`이면 freshness 평가를 건너뛰고 무조건 `crypto.randomUUID()`로 새 sessionId·`isNewSession=true`·`systemSent=false`. 반면 non-isolated는 `:127-153`에서 "direct" reset policy로 freshness를 평가해 신선하면 기존 sessionId 재사용(주석 `:128-129` "roll over like 1:1 conversations"). **따라서 격리 세션은 '매번 백지 상태로 깨어나는' 세션이다.**

**carry vs drop**: 신규 세션이면 `FRESH_CRON_CARRIED_PREFERENCE_FIELDS`(user preference)와 non-auto model override, user auth override만 복사한다. `AMBIENT_SESSION_CONTEXT_FIELDS`(channel/groupId/sendPolicy)는 `preserveAmbientContext: !forceNew`라 isolated에서 버려진다(`session.ts:168` 직접 확인). resume 핸들과 auto-fallback override는 항상 드롭.

**session-reaper 청소**: `isCronRunSessionKey(key)`인 키(`cron:<jobId>:run:<sessionId>` 패턴)만 프루닝하고 base cron 세션은 보존(기본 보존 24h). `onTimer`의 finally에서 piggyback 호출된다 — 장기 실행 job이 `state.running`을 여러 tick true로 유지해도 reaper가 무기한 스킵되지 않게 하기 위함.

> **[설계 평가]**
> - 강점: '격리'가 단일 불리언 `forceNew`로 결정되고 그 아래 carry vs drop이 명시적 필드 화이트리스트로 검증 가능. impossible-state를 필드 리스트로 막는 좋은 패턴.
> - 트레이드오프: **정정 — 세션 상태 영속이 SQLite가 아니라 JSON 파일 store다**(§7에서 직접 검증). 이 축의 핵심 반례.
> - 거친 부분: `run.ts` 52KB로 분할 가이드 초과. **함정**: `isNewSession`이 두 곳에서 다르게 쓰인다 — `resolveCronSession`은 isolated에서 `true`를 주지만 auth-profile 해석 시 `cronSession.isNewSession && sessionTarget !== "isolated"`로 강제 false 처리(`#62783` 회귀 테스트 존재). 직관에 반함.

---

## 4. 결과 전달·재시도 — fail-loud primary vs fail-soft 알림

이 영역은 실행 결과(성공 announce, 실패 알림)를 어디로 보낼지 결정하고, one-shot transient 에러 재시도를 분류한다. `delivery-plan.ts`가 잡 config를 정규화해 `mode`(announce/webhook/none)·channel·to로 된 라우팅 plan을 만들고, `delivery.ts`가 이를 실제 채널 타깃으로 풀어 `sendDurableMessageBatch`로 durable 전송한다.

**책임 분리(fail-loud vs fail-soft)**: primary announce는 `bestEffort:false`로 호출 — 부분 채널 실패도 `partial_failed`면 throw해 크론 run 실패로 승격. 반면 실패 알림(`sendFailureNotificationAnnounce`)은 best-effort + 30s abort timeout으로, 막힌 채널이 이미 실패한 run을 더 늘리지 못하게 한다.

**재시도 정책 2층**:
- `retry-hint.ts`의 `resolveCronExecutionRetryHint`는 에러 문자열을 5개 transient 카테고리(rate_limit/overloaded/network/timeout/server_error)로 분류해 **retryable 여부 + category만** 반환. 구조화된 provider 분류(`classifiedReason`)가 정규식보다 우선.
- backoff·횟수는 `timer.ts`가 소유: `consecutiveErrors > maxAttempts`(기본 3)면 소진, 아니면 `errorBackoffMs`로 산출.

**backoff 값**(직접 확인): `DEFAULT_ERROR_BACKOFF_SCHEDULE_MS = [30_000, 60_000, 5*60_000, 15*60_000, 60*60_000]`([`src/cron/service/jobs.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/service/jobs.ts)`:44-49`). `errorBackoffMs`(`:67-72`)는 `consecutiveErrors-1`을 인덱스로 clamp해 마지막 값(60m)에 saturate.

**webhook 보안**: `normalizeHttpWebhookUrl`이 `new URL()` 파싱해 protocol이 http/https가 아니면 null(file://, javascript: 배제). **미확인**: SSRF(localhost/사설IP) 필터는 이 함수에 없고, 실제 POST 주체가 이 delivery 코드 경로에 없어 추가 가드 여부는 미확인.

> **[설계 평가]**
> - 강점: `mode/channel/to`를 discriminated 정규화(webhook일 때 channel/threadId 강제 undefined)해 불가능 상태를 표현 불가하게 만든 점이 closed-mode 정책과 일치. 분류(pure)와 정책(스케줄러) 관심사 분리가 깔끔.
> - 트레이드오프: `resolveFailureDestination`이 global←job 필드별 병합 + 모드전환 무효화 + 동일타깃 억제까지 한 함수에 몰려 분기 많음(각 분기에 계약 주석은 있음).
> - 거친 부분: **반직관** — 'deliver' 모드는 announce의 레거시 별칭이며 런타임 mode 집합은 announce/webhook/none 3개뿐. 또 delivery config가 아예 없어도 isolated/current/session: + agentTurn이면 announce(channel='last')로 자동 기본값 — **'설정 안 함'이 곧 '전달 안 함'이 아니다.**

---

## 5. 하트비트·run-log — 트리거는 여기가 아니다 + SQLite 이력

**핵심 함정 먼저**: 파일명 `heartbeat-policy.ts`는 하트비트를 **트리거하지 않는다.** 직접 열어 확인했다 — [`src/cron/heartbeat-policy.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cron/heartbeat-policy.ts)`:1` 주석이 "stay out of visible delivery"라 선언한다. 이건 하트비트-only ack 응답의 **전달 억제 정책**이다: `shouldSkipHeartbeatOnlyDelivery`(`:15-37`)가 media/interactive가 있으면 skip=false, 없으면 `stripHeartbeatToken`으로 토큰을 벗겨 사용자에게 보일 텍스트가 없으면 skip=true. 즉 '각성했지만 보고할 게 없다'는 ack만 조용히 삼킨다.

**실제 주기적 자가 각성 엔진**은 `src/infra/heartbeat-runner.ts`의 `runHeartbeatOnce`다. cron 잡이 `wakeMode="now"`이면 `timer.ts:1459-1477`에서 `runHeartbeatOnce({source:"cron", intent:"immediate"})`로 heartbeat loop을 즉시 poke한다.

**실행 이력은 SQLite**: `appendCronRunLog`가 트랜잭션 안에서 `insertCronRunLogEntry` + `pruneCronRunLogRows`를 호출. `cron_run_logs` 테이블(indexed 컬럼 + `entry_json` 이중 저장, per-job monotonic seq). prune은 바이트가 아니라 row-count(`keepLines=2000` 기본).

**dead code 확인**(grep 근거): `shouldEnqueueCronMainSummary`(`heartbeat-policy.ts:40-57`, 직접 확인)는 non-test prod caller **0건** — 테스트 미러로만 존재. `maxBytes` config는 파싱만 되고 SQLite prune은 keepLines만 사용하는 no-op.

**run-diagnostics**: untrusted 실패 페이로드를 bounded·redacted 진단으로 정규화. 모든 메시지 `redactSensitiveText(mode:"tools")` 비식별화, entry 10개/1000자·summary 2000자 bound, exec는 tail 2000자 우선("실패는 보통 끝에 있다").

> **[설계 평가]**
> - 강점: run-log가 'SQLite only'를 정확히 지킨다. 런타임은 순수 SQLite+Kysely, JSONL은 doctor 마이그레이션에만 격리(§7). indexed 컬럼 + `entry_json` 이중 저장으로 forward-compat와 쿼리 성능 동시 확보.
> - 트레이드오프: query 모드 페이징이 전체 row를 JS로 필터(미인덱스 diagnostics 검색은 O(n)). keepLines 2000 상한 안에선 괜찮지만 확장성 리스크.
> - 네이밍 함정: `cron/heartbeat-policy.ts`가 '하트비트 스케줄 정책'처럼 읽히지만 실제로는 delivery 억제. 실제 각성 엔진은 `src/infra/heartbeat-runner.ts`로 완전히 별개.

---

## 6. tasks·flows 경계 — 원장과 오케스트레이션

세 축의 상호 호출 흐름:

```
cron 스케줄 발화
   │
   ├─ createCronExecutionId  = cron:{jobId}:{startedAt}
   │
   ├─▶ tasks (원장):  tryCreateCronTaskRun → createRunningTaskRun(
   │       runtime:"cron", scopeKind:"system",
   │       deliveryStatus:"not_applicable", notifyPolicy:"silent")   ← 조용한 시스템 원장 행
   │       완료 시 → completeTaskRunByRunId / failTaskRunByRunId
   │
   └─▶ (task가 parentFlowId 있으면) flow: syncFlowFromTaskResult
           task_mirrored 모드 = 단일 task를 flow status로 미러
           managed 모드       = 플러그인 controllerId가 setFlowWaiting/resumeFlow로 능동 구동
```

**비대칭(핵심)**: cron 런의 task 행은 `scopeKind:"system"` + `notifyPolicy:"silent"` + `deliveryStatus:"not_applicable"`이라 **tasks의 전달 파이프라인(`maybeDeliverTaskTerminalUpdate`)을 타지 않는다.** cron 결과 전달은 cron 자체(`delivery-plan.ts`, §4)가, 재시도도 cron 자체(`retry-hint.ts`)가 소유. tasks 전달/재시도는 주로 ACP/subagent 백그라운드 실행용이다.

**managed flow의 실제 구동자는 코어가 아니라 플러그인 컨트롤러다.** `createManagedTaskFlow`는 `controllerId` 필수이고 진행은 `runtime-taskflow.ts`의 `setFlowWaiting/resumeFlow`가 담당 — 다단계 오케스트레이션 로직은 플러그인 쪽, 코어는 상태 저장/낙관적 동시성(revision)만.

> **[설계 평가]**
> - 강점: `task_mirrored` vs `managed` syncMode 구분으로 '단일 백그라운드 실행'과 '플러그인 주도 다단계 워크플로'를 한 테이블에서 표현하면서 코어를 플러그인-불가지론적으로 유지.
> - 트레이드오프: `task_mirrored` flow는 사실상 단일 task의 status 래퍼라 개념이 하나 더 늘고, task↔flow 이중 레코드 + 동기화 비용(실패 시 지수 백오프 재시도)까지 필요.
> - 거친 부분: **미확인** — 이 절의 파일별 라인 주장(task-registry.ts:1678, task-flow-registry.ts 등)은 이번 표본 검증 범위 밖. 별도 검증 필요.

---

## 7. 영속과 SQLite 정책 — 대체로 준수, 하지만 세션 계층은 반례

cron 잡 정의·실행 이력·런타임 커서는 전부 공유 state DB `state/openclaw.sqlite`의 `cron_jobs`/`cron_run_logs` 두 테이블에 저장된다. SQLite 접근은 전부 Kysely 헬퍼로, raw SQL은 DDL/마이그레이션에만 있다.

**"jobs.json"은 파일이 아니다**: storePath로 넘어오는 `cron/jobs.json`은 실제 파일이 전혀 아니고 `cronStoreKey`에서 `path.resolve`만 거쳐 SQLite `store_key` 파티션 값으로만 쓰인다. 그 경로에 파일을 쓰는 코드는 없다 — 오해 유발 레거시 네이밍.

**JSONL은 정책 예외(위반 아님)**: `run-log-jsonl.ts`(직접 확인 주석 `:1` "Legacy JSONL run-log parser used during migrations/imports")의 유일한 non-test caller는 `doctor/cron/legacy-run-log-migration.ts`. steady-state 런타임 import 0건. CLAUDE.md "legacy shapes normalize only in doctor/migration"의 교과서적 구현.

**정정(핵심 반례): 격리 에이전트의 세션 상태는 여전히 JSON 파일 store다.** 직접 검증 결과 — `loadSessionStore`(`config/sessions/store-load.ts:405,410`)는 `fs.readFileSync(storePath,"utf-8")` + `JSON.parse(raw)`로 **JSON 파일**을 읽고 mtime 기반 파일 stat 캐시를 쓴다. SQLite 아님. `resolveCronSession`이 이 파일 store를 직접 소비한다. cron 잡·run-log는 SQLite로 잘 이전됐으나 **cron이 의존하는 세션 상태 계층은 여전히 파일**이라는 비대칭이 실재하며, 이는 CLAUDE.md "Storage default: SQLite only / no JSONL sidecar"와 실질 충돌한다. cron 서브시스템의 "SQLite only 준수" 서사에 대한 유효한 반례다.

**회색지대**: `-quarantine.json` 사이드카는 steady-state 런타임에서도 `flushPendingQuarantine` 경로로 실제 JSON 파일을 쓴다. malformed 잡을 격리하는 진단 아티팩트라 named product artifact로 볼 여지 — 정책 예외인지 위반인지 애매한 회색지대다.

> **[설계 평가]**
> - 강점: cron이 CLAUDE.md 'shared state DB' 권고를 정확히 따름 — 잡·이력 모두 공유 state DB, Kysely, raw SQL은 DDL/마이그레이션에만. config/runtime을 컬럼 수준 분리해 stateOnly 핫패스가 유저 JSON을 재작성하지 않음.
> - 트레이드오프: 분리 컬럼 + `job_json` 이중 저장은 저장 중복이라 코덱 복잡도를 키움 — 정직한 LOC/복잡도 트레이드오프.
> - 거친 부분: 세션 파일 store와 quarantine 사이드카가 "순수 SQLite" 서사를 깬다. 전자는 명백한 마이그레이션 부채, 후자는 회색지대.

---

## 8. 종합

이 축의 관통 논지를 한 그림으로 요약하면: **시간이 발화(§1 스케줄 유니온) → 단일 self-arming 타이머가 예약을 lock 안에서 커밋하고 실행은 lock 밖에서(§2) → 격리 에이전트가 `forceNew`로 백지 각성(§3) → primary는 fail-loud, 실패알림은 fail-soft로 전달하고 transient만 backoff 재시도(§4) → 이력을 SQLite run-log에 남긴다(§5).** cron은 트리거를, tasks는 실행-추적을, 진짜 flow는 오케스트레이션을 소유하는 3분할(§6)이 축을 깔끔하게 나누되, 영속은 대체로 SQLite로 canonical하지만 **격리 세션 계층만 파일에 남은 것이 이 축의 유일하고 정직한 반례(§7)**다.

**다른 편과의 교차참조**: 이 축은 (a) **agent-core 세션 DAG** 편과 `usageFamilyKey`/subagent 세션키로 이전→현재 sessionId 계보를 병합하며 연결되고(단 실제 DAG 구성 코드는 컨텍스트 엔진 쪽이라 **미확인**), (b) **노드 push-to-wake** 편과는 `wakeMode="now"` 시 `runHeartbeatOnce`로 heartbeat loop을 poke하는 지점에서, (c) **게이트웨이** 편과는 `server-cron.ts`가 `runCommandJob` dep를 실제 배선하고 `CronServiceContract`를 gateway·plugin SDK·테스트가 공유하는 지점에서 맞물린다.

---

## 부록. 검증 메모

핀 SHA `0fc5a57a`에서 직접 열어 확인한 파일: `types.ts`(:9-18 판별 유니온, staggerMs cron-only), `schedule.ts`(:55-119 kind별 계산 + year-rollback workaround + LRU 캐시), `heartbeat-policy.ts`(:1 "stay out of visible delivery", :15-37 skip 판정, :40-57 dead `shouldEnqueueCronMainSummary`), `service/jobs.ts`(:44-49 backoff `[30s,60s,5m,15m,60m]`, :67-72 errorBackoffMs saturate).

grep/이전 검증으로 확정: `run-log-jsonl.ts` 유일 caller=doctor 마이그레이션(위반 아님), `loadSessionStore`가 `fs.readFileSync`+`JSON.parse`(세션 파일 store = SQLite-only 정책 반례, **직접 확인됨**), `maxBytes` no-op, `state.ts:224` `storage:"sqlite"` 하드코딩, `storePath` @deprecated.

**미확인 항목**: (1) §6의 `src/tasks/`·`src/flows/` 파일별 라인 주장은 이번 조사 미열람 — 별도 검증 필요. (2) 컨텍스트 엔진 세션 DAG와의 실제 연결 코드는 이 범위 밖. (3) webhook POST의 SSRF 가드 — POST 주체가 delivery 코드 경로에 없어 미확인. (4) `cron_jobs` DDL의 `schedule_kind default 'manual'`과 `persisted-shape.ts`의 at/every/cron 허용값 불일치 — 'manual' kind 실사용 여부 코드 추적 안 함.

**정확성 요약**: 중대한 오류 없음. 경미한 표현 오류 1건 — `types.ts` keyFile 한 줄 요약이 staggerMs를 every/at에도 있는 듯 서술(실제 cron-only, 본문 mechanism은 정확). 가장 중요한 확인 — area3의 "isolated 세션 상태 파일 영속 vs SQLite-only 정책" 모순이 `loadSessionStore` 직접 열람으로 **사실로 검증됨**.