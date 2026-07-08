# main 에이전트의 하루 — OpenClaw 제품-어시스턴트 층

> ultracode 멀티에이전트(서브에이전트 Opus 4.8 × 9)로 7영역 병렬 매핑 → 정확성 비평 → 종합.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src`/`packages`를 변경하지 않아 working tree == 이 SHA로 검증됨).
> 컨셉: 인프라 편들([gateway](./gateway-study.md)·[러너](./agent-runtime-study.md)·[agent-core](./agent-core-study.md)) 위에 얹힌 **"인격을 가진 어시스턴트" 층**을 하루의 흐름으로 읽는다 — 부팅 → 인격 장착 → 응대 → 스스로 깨어남 → 기억 → 세션 수명.

---

## 0. 이 문서의 위치 — 기계 위의 인격

OpenClaw는 3층으로 읽을 수 있다: (a) **인프라 층** — 게이트웨이·러너·agent-core 루프·채널, (b) **제품-어시스턴트 층**(이 문서) — 부트스트랩 파일·SOUL/IDENTITY·auto-reply 정책·heartbeat·memory·세션 수명, (c) **사용자 데이터** — 사용자가 편집하는 SOUL.md/USER.md/MEMORY.md.

**핵심 명제:** "어시스턴트다움"의 대부분은 **코드가 아니라 워크스페이스의 마크다운 파일(프롬프트)** 에 산다. 코드는 뼈대만 제공한다 — 파일을 스캐폴딩하고, 세션별로 필터링하고, 예산 안에서 트렁케이션하고, 결정적 순서로 시스템 프롬프트에 주입하고, 침묵/억제 규약을 강제한다.

---

## 1. 워크스페이스와 부트스트랩 파일 — 인격의 물리적 몸

**코드가 인식하는 부트스트랩 파일은 8종**: `AGENTS.md · SOUL.md · TOOLS.md · IDENTITY.md · USER.md · HEARTBEAT.md · BOOTSTRAP.md · MEMORY.md` ([`src/agents/workspace.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/workspace.ts)`:31-38`, `VALID_BOOTSTRAP_NAMES :194-203`). **BOOT.md는 이 집합에 없는 별개 시작-훅 파일**(§3). CLAUDE.md는 AGENTS.md 심볼릭 링크.

- **경로**: `OPENCLAw_WORKSPACE_DIR` > `~/.openclaw/workspace[-<profile>]`(`workspace-default.ts:11-30`). `openRootFile` 경계로 심볼릭/탈출 방지.
- **온보딩 스캐폴딩**: 템플릿 원본은 레포의 `docs/reference/templates/`(HEARTBEAT만 `src/agents/templates`). `writeFileIfMissing(flag:"wx")`로 **없을 때만** 기록(`workspace.ts:911-934`). AGENTS/TOOLS 필수, SOUL/IDENTITY/USER/HEARTBEAT는 optional.
- **vanish 가드**: `.openclaw/workspace-state.json`(bootstrapSeededAt/setupCompletedAt) + sha256 attestation — 최근 초기화 후 워크스페이스가 사라지면 `WorkspaceVanishedError`로 재시딩 거부(사용자 데이터 덮어쓰기 방지, `workspace.ts:461-515`).
- **세션종류 필터**(`:1083-1107`): 서브에이전트 = `{AGENTS, TOOLS}`만, 크론 = `+{SOUL, IDENTITY, USER}`. **MEMORY.md는 메인 세션 전용** — 프라이버시를 프롬프트 지침이 아니라 **로더 코드로** 강제. ⚠️ 단 `!sessionKey`면 전체 반환(`:1096-1098`) — 방어가 완전히 닫혀 있진 않다.
- **예산·트렁케이션**: 파일당 20k자·총 60k자(`embedded-agent-helpers/bootstrap.ts:91-92`), head 0.75/tail 0.25, **AGENTS.md만 policy-digest 전용 트렁케이션**(정책 라인 우선 보존, `:221-269`). 트렁케이션 시 raw↔injected 비교 경고 블록(`bootstrap-budget.ts:135-233`).
- **주입 순서**: `CONTEXT_FILE_ORDER` agents(10)→soul(20)→identity(30)→user(40)→tools(50)→bootstrap(60)→memory(70)(`system-prompt.ts:69-77`). **HEARTBEAT.md만 dynamic** → 캐시경계 아래(`:191-238`).
- **BOOTSTRAP.md = 일회성 출생증명서**: pending이면 `## Bootstrap Pending` 배너("첫 답변은 인사가 아니라 BOOTSTRAP.md를 따르라", `:320-330`). 완료 시 파일 삭제 — 이름/성격/이모지 결정→IDENTITY·USER·SOUL 갱신하는 자기소개 의식 대본.

> **[설계 평가]** 인격·기억·툴노트·주기작업을 개별 파일로 분리해 독립 진화·트렁케이션·세션필터가 가능. MEMORY 세션 제외·attestation 가드가 코드 레벨 방어의 모범. 거친 부분: "메인 세션 여부"가 별도 플래그가 아니라 "subagent/cron이 아님"이라는 **부정 판정**.

---

## 2. SOUL·IDENTITY — 인격이 코드로 표현되는 방식

**인격 = 2층 구조**: 코드에 박힌 고정 앵커 1줄 + 편집 가능한 워크스페이스 파일.

- **고정 앵커**: `"You are a personal assistant running inside OpenClaw."` — 코드상 "정체성"은 사실상 이 한 줄뿐(`system-prompt.ts:966-970`). 파일이 없어도 최소 정체성은 코드가 보장, **SOUL.md가 없으면 인격 지시문 자체가 프롬프트에서 사라진다.**
- **SOUL.md = 성격/톤, 원문 그대로 주입**: "You're becoming someone"·"Have opinions"([`docs/reference/templates/SOUL.md`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/docs/reference/templates/SOUL.md)`:10-35`). 앞에 자동 라벨("persona/tone. Follow it…") 프리펜드(`system-prompt.ts:217-225`).
- **IDENTITY.md는 예외적으로 파싱**: `parseIdentityMarkdown`이 Name/Emoji/Creature/Vibe/Avatar 필드 추출(템플릿 플레이스홀더 배제, `identity-file.ts:30-100`) → 게이트웨이 표시용 identity로 병합.
- **표시 identity precedence**: `resolveAssistantIdentity` = 기본 에이전트면 `ui ?? agent ?? file`, 비기본이면 `agent ?? file ?? ui`(`assistant-identity.ts:104-119`) — "기본이냐"에 따라 우선순위가 뒤집히는 제품 판단. DEFAULT는 name="Assistant"/avatar="A".
- config per-agent: 이름 → `[name]` 메시지 프리픽스, emoji → ack 리액션(기본 👀).
- realtime(음성)은 IDENTITY→USER→SOUL 서브셋 12k자 예산 + "do not mention unless asked".

> **[설계 평가]** 인격이 코드가 아니라 **편집 가능한 데이터**에 산다는 게 설계 축. SOUL(원문 주입=성격)과 IDENTITY(필드 파싱=기계적 메타데이터)의 처리 분기가 명확 — **성격은 데이터로 남고, 정체성 메타데이터만 코드로 승격된다.**

---

## 3. BOOT.md — 게이트웨이 부팅 시 자가 점검

게이트웨이 startup 후반, 워크스페이스별 `BOOT.md`를 **격리된 boot 세션**에서 1회 실행.

- **계약**: 루트 BOOT.md를 읽어 trim — 없으면/공백이면 스킵, 내용 있으면 "Follow BOOT.md instructions exactly" 프롬프트로 주입([`src/gateway/boot.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/boot.ts)`:62-102`).
- **4중 격리**: 전용 세션키 `agent:{id}:boot` + 매번 새 sessionId + `deliver:false` + 세션 매핑 `structuredClone` 스냅샷→finally 원복(`:104-231`). 응답은 `NO_REPLY`만 하도록 지시.
- **시점**: post-attach 사이드카 이후 250ms 지연 발화(`server-startup-post-attach.ts:845-859`). 실패는 best-effort warn — 부팅을 막지 않음.
- **echo 이중 억제**: internal-runtime-context 델리미터 strip + 세션키별 프롬프트 등록 → 아웃바운드에서 **80자 이상 연속 substring 매칭**으로 제거(`boot-echo-guard.ts:13,50-51`) — 약한 모델의 내부 프롬프트 유출(#53732) 방어.
- ⚠️ task dedup 키가 agentId가 아니라 **workspaceDir** — 워크스페이스 공유 시 첫 agentId로 1회만(`hooks/bundled/boot-md/handler.ts:32-44`).

> **[설계 평가]** "메인 세션에 개입하지 않는 격리 실행" 4중 방어가 핵심. 실패가 조용한 warn 로그뿐이라 관측성이 약한 게 거친 부분.

---

## 4. auto-reply — 메시지를 어시스턴트답게 응대

경로: `provider-dispatcher` → `dispatch`(포그라운드 펜스·훅·타이핑) → `dispatch-from-config`(입장정책·라우팅) → `get-reply-run`(큐/steer) → reply run.

- **침묵 규약 `SILENT_REPLY_TOKEN = "NO_REPLY"`**(`tokens.ts:7`): 그룹 프롬프트에 "응답 불필요 시 정확히 NO_REPLY만" 지시 → 디스패처가 토큰-only·**JSON envelope·reasoning-prefixed 변형까지** 침묵 판정(`tokens.ts:60-198`).
- **그룹 mention 요구**: `resolveGroupRequireMention`(채널→config 폴백) — 미언급이면 markIdle·무응답, elevated/exec 디렉티브도 무시(`dispatch-from-config.ts:1958-1970`).
- **타이핑 컨트롤러**: runComplete+dispatchIdle **둘 다** 충족돼야 정지하는 "seal"(`typing.ts:182-255`) — 늦은 콜백이 타이핑을 영원히 재시작시키는 버그를 구조 차단. NO_REPLY 텍스트면 시작도 안 함. heartbeat/internal은 suppress.
- **블록 스트리밍**: minChars 800/maxChars 1200/idleMs 1000(`block-streaming.ts:110-249`) — 단 이 파일은 **렌더링이 아니라 chunk/coalesce 설정 해소**(실제 send+edit은 `src/channels/streaming.ts`).
- **reply run↔세션**: `replyRunRegistry`(sessionKey↔sessionId↔operation), 큐 모드로 run-now/interrupt/followup 결정. `/steer`는 활성 run에 주입, 없으면 일반 프롬프트 강등.
- **포그라운드 펜스**: 세대 카운터 — 새 세대가 배달 완료하면 이전 세대 배달 취소(`dispatch.ts:105-175`). shutdown drain은 `getTotalPendingReplies`.

> **[설계 평가]** 침묵 판정의 다층 방어(토큰·JSON·reasoning·streaming 조각)는 견고하나 정규식 유지비가 높은 과설계 경향. 활성 상태가 **3중 표현**(replyRunRegistry/embeddedRuntime/session)이라 busy 판정이 복잡한 게 가장 거친 부분.

---

## 5. heartbeat — 스스로 깨어남

- **스킵 게이트 ≥7층** (`heartbeat-runner.ts:1319-1440`, 위→아래 선형): disabled → interval 0 → quiet hours → **Main lane 큐>0** → cron/nested lane → busy lane → 활성 reply run defer → pending delivery → 세션 lane busy. 별도 비용 게이트는 없음 — **empty-skip·isolatedSession·lightContext 3가지 구조적 선택**으로 절감.
- **HEARTBEAT.md empty 게이팅**: 기본 템플릿은 주석뿐 → `isHeartbeatContentEffectivelyEmpty`면 **API 호출 자체를 skip**(`:1044-1055`). ⚠️ 반대 동작 주의: **파일 없음(ENOENT)=런 유지, 비어 있음=skip**(`:1062-1068`).
- **캐시 친화**: `heartbeat.md`만 `DYNAMIC_CONTEXT_FILE_BASENAMES` → 캐시경계 뒤(§1).
- **우선순위**: trigger='heartbeat' → **queue priority = background**(`run.ts:326-341`) — 사용자 run이 항상 앞선다.
- **응답 억제 3겹**: ① `HEARTBEAT_OK` 토큰+300자 이하면 skip ② `heartbeat_respond` 툴 notify=false ③ 채널 showOk 기본 false + 다음 턴 트랜스크립트 필터로 히스토리 청소(`heartbeat-filter.ts:452-482`).
- **wake 큐·쿨다운**: coalesce 250ms, priority ACTION>DEFAULT>INTERVAL>RETRY, 30s min-spacing + 60s/5회 flood 가드(`heartbeat-cooldown.ts:85-145`).
- 게이트웨이 RPC: `last-heartbeat`·`set-heartbeats`(전역 토글).

> **[설계 평가]** wake 억제(디스패치 전)와 실행 스킵(직전)의 이중 방해 방지 + "조용한 heartbeat는 API 호출 0"이 구조로 달성됨. 거친 부분: 억제 규약이 토큰·툴·config·필터 네 곳에 흩어져 "왜 조용했나" 추적이 어렵다.

---

## 6. MEMORY.md / memory — 장기기억 유지보수

장기기억은 **세 갈래**로 유지된다:

1. **pre-compaction flush (trigger='memory')**: 컨텍스트가 임계값에 근접하면 압축 **직전** durable 메모리를 디스크에 flush하는 별도 런(`agent-runner-memory.ts:1280-1291`) — "정리 런"이 아니라 "압축 전 저장 런". ★ **두 축 구분(비평 정정)**: 이 런은 **lane=Main이면서 queue priority=background** — lane(실행 레인)과 priority(스케줄 순위)는 별개 축이다(`run.ts:326-340` vs `agent-runner-memory.ts:1249`).
2. **memory-core 인덱싱/검색**: 인덱스 대상 = 심링크 아닌 `MEMORY.md` + `memory/**.md` 재귀. `memory_search`/`memory_get`. (`wiki_search`/`wiki_get`은 memory-core가 아니라 **memory-wiki 플러그인** 소유 — 비평 정정.)
3. **dreaming = 스케줄된 승격**: memory-core가 managed cron 잡으로 단기→장기 승격·예산 압축(`dreaming.ts:44-72`) — flush(세션 임계값)와 dreaming(cron)의 두 축 완전 분리.

- **flush 툴 안전 설계**: read + **append-only write 2개만**, `memory/YYYY-MM-DD.md` 한 파일 한정, MEMORY.md/SOUL.md는 read-only(`memory-core/flush-plan.ts:19-34`) — 자동 쓰기가 큐레이션을 오염 못 함. 실패 3회면 포기, 압축 사이클당 1회.
- **MEMORY.md 이중 정체성**: 부트스트랩 파일(항상 프롬프트 주입) ∩ 검색 인덱스 엔트리.
- **코어/플러그인 경계**: 코어는 `MemoryPluginCapability` 슬롯만, flush plan/prompt/검색은 memory-core 플러그인이 주입 — **플러그인 교체로 메모리 정책을 통째 바꿀 수 있다.**
- ⚠️ session-memory 훅(/new·/reset 시)은 `memory/YYYY-MM-DD-<slug>.md` — flush의 `YYYY-MM-DD.md`와 **다른 경로 규약**.

> **[설계 평가]** MEMORY.md(큐레이션 요약, dreaming이 관리) vs memory/일자.md(에이전트가 append하는 원장)의 역할 분리 + flush 툴을 2개로 좁힌 안전 설계가 정점. lane vs priority 두 축을 뭉뚱그리면 오독한다.

---

## 7. 메인 세션의 수명 — 하루의 연속성

- **키**: `agent:<agentId>:<mainKey>` = 기본 **`agent:main:main`**(`routing/session-key.ts:27-28`). `canonicalizeMainSessionAlias`가 레거시/별칭 흡수.
- **수렴(dmScope=main)**: direct 대화는 peerId를 무시하고 메인 키로 접힘 — **TUI·앱·채널 DM이 같은 세션**으로 수렴(`session-key.ts:222-254`). per-peer 모드면 identityLinks로 cross-channel 재통합.
- **/new·/reset**: 기본 트리거 `['/new','/reset']` — 새 sessionId 발급, override(model/verbose) 보존. soft reset은 세션 유지한 채 바인딩/스냅샷만 클리어.
- **daily/idle 프레시니스**: `reset.mode` daily(기본 atHour=4) 또는 idle(idleMinutes). ⚠️ **daily 리셋 크론/타이머는 없다 — 다음 인바운드 턴에서 lazy 평가**(`session.ts:447-491`). daily는 `sessionStartedAt`, idle은 `lastInteractionAt` 기준 — **서로 다른 타임스탬프**라 이어진 세션은 원본이 오래돼도 idle-fresh.
- **sessions.json 유지보수**: pruneAfter 30일·maxEntries 500. ⚠️ **비대칭**: thread/group/channel 세션은 protected인데 **direct(메인)는 protected가 아님**(`store-maintenance.ts:294-313`) — 30일 미접촉 메인 세션은 prune 대상.

> **[설계 평가]** "단일 canonical 키 수렴 + alias 흡수"가 축이고, 명시/암묵 리셋이 같은 롤오버 경로로 수렴해 아카이브·override 보존이 일관. "사람 대화 surface는 protected, direct는 disposable"이라는 비대칭과 lazy daily 평가가 비직관 포인트.

---

## 8. 종합 — "어시스턴트다움"은 어디까지 코드이고 어디부터 마크다운인가

**코드가 소유(뼈대)**: 앵커 1줄 · 파일 목록/세션 필터/예산 · 주입 순서 · 캐시경계 분리 · NO_REPLY/HEARTBEAT_OK 침묵 규약 · heartbeat 스킵 게이트 · flush 툴 제약 · 세션 canonicalize/프레시니스. → **"언제·어떤 순서로·무엇을 억제하며 주입/실행하는가"**

**마크다운이 소유(인격)**: SOUL(성격) · IDENTITY(이름/이모지) · USER · TOOLS · MEMORY · HEARTBEAT(주기작업) · BOOTSTRAP(출생 의식) · BOOT(부팅 스크립트). → **"실제로 어떤 존재이고 무엇을 하는가"** — 파일이 없으면 그 인격/행동이 프롬프트에서 사라진다.

**경계선**: IDENTITY.md만 예외적으로 코드가 파싱해 기계적 용도(UI/프리픽스/리액션)로 승격 — **성격은 데이터로 남고, 정체성 메타데이터만 코드로 넘어온다.**

**강점**: 파일 단위 인격 분리 / 프라이버시·안전·격리를 코드 레벨 강제 / 토큰 비용의 구조적 절감 / 메모리 정책의 플러그인 교체 가능성.

**거친 부분**: 활성 상태 3중 표현 / 억제 규약 4곳 산재 / 부정 판정 의존(`!sessionKey` 공백) / **lane vs queue priority 두 축 혼동 위험**(비평이 잡은 실질 오류) / boot 실패 관측성.

---

### 부록. 검증 메모
- 핀 유효성: `0fc5a57a..HEAD`는 study 문서만 변경 → `src`/`packages` byte-동일(메인 세션 검증).
- 비평 정정 반영: memory flush의 **lane=Main + priority=background 두 축 구분**(이전 "background 분류 못 찾음"은 오류 — `run.ts:326-340`이 명시 분류) / `wiki_*` 툴은 memory-wiki 소유 / `!sessionKey` 시 전체 파일 반환 방어 공백 / MEMORY.md만 MISSING 마커 없이 완전 배제 / identity precedence 코드 일치 확인.
