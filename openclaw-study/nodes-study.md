# OpenClaw 노드 시스템 — 게이트웨이에 손발을 빌려주는 기기들

> ultracode 멀티에이전트(서브에이전트 Opus 4.8 × 9)로 7영역 병렬 매핑 → 정확성 비평 → 종합.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src`/`packages`/`apps`/`extensions`를 변경하지 않아 working tree == 이 SHA로 검증됨).
> 선행 전제: [게이트웨이](./gateway-study.md)(WS·인가·registry 기초)와 [에이전트](./agent-runtime-study.md)(nodes-tool·exec 분기)는 안다고 가정.

---

## 0. 노드란 무엇인가 — 재정의

"노드"는 에이전트가 아니라, 게이트웨이에 `role=node`로 붙어 **자기 기기의 능력(caps/commands)을 빌려주는 종속 클라이언트**다. 에이전트가 "머리"라면 노드는 "손발" — 카메라·화면·셸·위치를 게이트웨이 요청에 따라 실행하고 결과만 돌려준다.

**핵심 재정의 3가지:**
- 노드는 **두 얼굴** — 헤드리스 TS 데몬(`openclaw node run`)과 네이티브 앱 내장 모드(macOS/iOS/Android). **동일 프로토콜**(`node.invoke.request/result`·`node.event`)을 공유하나 **구현은 완전 분리**(TS vs Swift vs Kotlin).
- 노드의 신뢰는 에이전트와 다르다 — 에이전트가 게이트웨이 안 "준비된 신뢰"라면, 노드는 **외부에서 붙는 미신뢰 기기**라 페어링·인가·표면 교집합이라는 별도 관문을 거친다(§8).
- 노드가 노출하는 능력은 **설치된 플러그인 + 플랫폼에 따라 동적** — 정적 목록이 아니다.

---

## 1. node-host 런타임 — 헤드리스 노드의 탄생

`openclaw node run`은 별도 프로세스를 spawn하지 않고 **현재 프로세스가 곧 노드**인 포그라운드 데몬이다.

- **부팅**: `ensureNodeHostConfig`(node.json) → displayName(opts>config>머신명) → 플러그인 레지스트리에서 caps/commands 수집 → GatewayClient 생성 → **`await new Promise(()=>{})`로 이벤트루프 붙잡기**([`src/node-host/runner.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/node-host/runner.ts)`:234-326`).
- **선언 표면**: `caps=['system', ...plugin]`, `commands=[system.run 3종 + execApprovals 2종 + plugin]`, **`scopes:[]`**(`runner.ts:280-289`, 상수 `src/infra/node-commands.ts:2-14`) — 이 `scopes:[]`가 §2 자동승인의 열쇠.
- **auth-fatal만 exit(1)**: 재접속 백오프는 GatewayClient에 위임하고, `AUTH_TOKEN_MISSING/MISMATCH`·`BOOTSTRAP_INVALID`·`PASSWORD_*`·`CLIENT_VERSION_MISMATCH`(6종)면 `process.exit(1)`로 슈퍼바이저에 넘긴다(`runner.ts:75-115`).
- **상태 파일**: `node.json`(stateDir, mode 0600) — nodeId(자동 UUID)/token/displayName/gateway(`config.ts:14-77`).
- **데몬화**: `node install`이 launchd/systemd/schtasks에 `['node','run',…]` argv를 등록(`daemon.ts:92-178`).

> **[설계 평가]** 재접속은 클라이언트 라이브러리, fatal은 슈퍼바이저 — 책임을 깔끔히 나눔. 반면 macOS 앱은 자체 while+지수백오프(1s→10s)라 "동일 프로토콜·다른 수명주기"의 비대칭이 있다.

---

## 2. 페어링·인증 — 신뢰를 얻는 두 겹의 관문

페어링은 **두 계층으로 분리**된다: **연결 인증**(누가 붙어도 되나) = device-pairing, **능력-표면 승인**(무슨 능력을 쓰나) = node-pairing.

- device-pairing = publicKey + 역할별 bearer 토큰(회전/폐기까지) 무거운 인증([`src/infra/device-pairing.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/device-pairing.ts)`:614-1055`). node-pairing = caps/commands/permissions 표면 + 단일 token의 가벼운 능력-승인(`node-pairing.ts:86-390`).
- **pending은 노드가 아니라 게이트웨이가 생성**: `node.pair.*` 6종 전부 `operator.pairing` 스코프라 노드는 호출 불가. connect 중 `reconcileNodePairingOnConnect`가 pending을 만든다(`node-connect-reconcile.ts:112-225`) — 페어링 스팸을 구조적으로 차단.
- 전이: pending(TTL 5분) → operator 승인(`approveNodePairing`, macOS NSAlert 또는 `openclaw nodes pairing approve`) → paired → 라이브 세션 `updateSurface`.
- **자동승인 2종**: silent local(로컬+공유시크릿+비브라우저) / trusted CIDR(role=node·미페어링·**scopes 비어 있음**·비브라우저, `node-pairing-auto-approve.ts:18-79`).
- **재접속 재인증은 토큰이 아니라 교집합**: device 자격 + `승인분 ∩ 선언분`만 effective — 재연결로 몰래 능력을 넓히는 것을 차단(`node-connect-reconcile.ts:178-224`).

> **[설계 평가]** 인증과 인가-표면을 두 파일로 분리하고 `node.pair.request`까지 operator 전용으로 못박은 fail-closed 설계.
>
> **반직관 3가지**: (a) `node_pairing_*` **SQLite 테이블은 스키마에만 있고 런타임은 JSON 파일**(nodes/pending.json·paired.json) 사용 — vestigial. (b) node-pairing 토큰은 재연결 인증에 안 쓰이고 오직 `node.pair.verify` RPC용. (c) `node.pair.request`가 operator 스코프.

---

## 3. invoke 프로토콜 · 명령 카탈로그

게이트웨이 `node.invoke`는 requestId 기반 pending-invoke(기본 30s)로 `node.invoke.request`를 보내고 `node.invoke.result`로 resolve한다.

- **프로토콜**: `NodeRegistry.invoke` — randomUUID requestId, `{id,nodeId,command,paramsJSON,timeoutMs,idempotencyKey}`([`node-registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/node-registry.ts)`:426-491`). 응답은 **nodeId AND connId 이중검증** 후 resolve, late result 무시(`:642-674`).
- **거절 경로**: 미연결→APNs wake 2단계 후 `NOT_CONNECTED` / allowlist·declared 미포함→`INVALID_REQUEST` / `execApprovals.*`·persistent browser 변이→하드 차단 / iOS 포그라운드 전용→`QUEUED_UNTIL_FOREGROUND` 큐잉.
- **카탈로그** — 코어: `system.run.prepare/run/which`·`system.notify`·`browser.proxy`·`execApprovals.get/set`. 기기 능력: `PLATFORM_DEFAULTS`(camera/screen/location/notifications/contacts/calendar/photos/motion/sms…, `node-command-policy.ts:15-129`).
- **정정**: `talk.ptt.*`는 플러그인 경유가 아니라 **코어 정적 상수 `TALK_PTT_COMMANDS`**(`node-command-policy.ts:50`, `hasTalkSurface`면 합류). **canvas만** `nodeInvokePolicies`(플러그인 등록) 경유(`:239-268`).
- **이중 허용 검사**: 게이트웨이 allowlist ∩ 노드 declaredCommands, 그 위 네이티브 가용성 게이트(CAMERA_DISABLED 등).

> **[설계 평가]** 방어는 계층적으로 견고하나, command 목록이 **코어·Android·iOS·앱 상수 4곳에 분산**돼 표류 위험. gotcha: `system.run`은 §5의 별도 채널로 처리 — 데스크톱 노드(system.*+플러그인)와 모바일 앱(기기 능력)은 서로 다른 디스패처라 상대 명령엔 UNAVAILABLE.

---

## 4. 기기 능력 파이프라인 — 노드의 감각

능력은 방향에 따라 **3종류**로 갈린다:

| 방향 | 능력 | 경로 |
|---|---|---|
| **동기 요청-응답** | camera·location·canvas·talk-ptt | 에이전트 툴 → `node.invoke` → 네이티브 핸들러 → payload 반환 |
| **게이트웨이→노드 브로드캐스트** | voicewake | `voicewake.set` → settings 저장 → `voicewake.changed` 전 클라 push (**node.invoke 없음**) |
| **인바운드→전사** | audio | node.invoke가 아니라 **인바운드 첨부 media-understanding** |

- camera: 노드가 **5MB 이하 재압축 base64 jpg** 반환 → 에이전트가 temp 파일화 + `modelHasVision`일 때만 image 주입(`nodes-tool-media.ts:143-183`).
- canvas: **양방향** — 노드 화면 제어 후 `snapshot` 렌더 결과를 이미지로 역주입(`extensions/canvas/index.ts:13-137`).

> **[설계 평가]** 새 능력 추가 = ①게이트웨이 정책 ②에이전트 툴 ③네이티브 핸들러 3곳 동기화. base64 폭증은 재압축+raw 차단+temp 파일화+vision 게이팅 다층 방어. "노드 능력"이 문서상 한 축처럼 보이지만 실제 방향이 셋으로 갈리는 비대칭이 핵심.

---

## 5. exec host=node — 노드에서 셸을 부리는 법

에이전트 exec가 `host=node`면 게이트웨이가 `node.invoke system.run`으로 포워딩하고, **노드가 자체 정책·승인을 재평가**한 뒤 spawn한다.

- 진입: `executeNodeHostCommand`(`bash-tools.exec.ts:1640`). 노드 지정 = `params.node` 또는 `defaults.node`(bound), 불일치 거부.
- 셸: POSIX `["/bin/sh","-lc",cmd]`, Windows `["cmd.exe","/d","/s","/c",cmd]`(`node-shell.ts:9-12`).
- **3중 게이트**: ① caller 사전판단(`system.run.prepare` + 원격 approvals 스냅샷, 원격이 더 엄격하면 auto-review 차단) ② 게이트웨이가 approved 플래그를 안 믿고 **runId↔exec.approval 레코드 재검증**(`node-invoke-system-run-approval.ts:239-320`) ③ 노드측 parse→policy→execute 재평가 + cwd/스크립트 drift 재검증 + env PATH 차단([`invoke-system-run.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/node-host/invoke-system-run.ts)`:376-928`).
- **정정**: cmd.exe 래퍼 차단(`windowsShellWrapperBlocked`)은 "항상"이 아니라 **`security=allowlist` 모드 한정**(`exec-policy.ts:71-76`) — full 모드면 통과.
- 실행·회신: spawn(stdin ignore, 출력 200KB cap, 타임아웃 SIGKILL) → 동기 result + 비동기 `exec.finished`(tail 20KB). **실패 invoke는 게이트웨이가 승인창 즉시 회수**(`node-registry.ts:660-666`).
- macOS 분기: `OPENCLAW_NODE_EXEC_HOST=app`이면 유닉스 소켓으로 companion 앱에 위임(승인 UI 바인딩), 불가 시 로컬 spawn 폴백.

> **[설계 평가]** caller·게이트웨이·노드 3자가 독립 재평가하는 신뢰 경계가 계층마다 fail-closed. POSIX sh는 transport wrapper로 보고 inner payload로 판정하되 cmd.exe는 semantics가 달라 allowlist 모드에서 별도 게이트 — 셸 신뢰도 차이를 정책에 반영. gotcha: 타임아웃 3겹(노드 SIGKILL / invoke `max(10s, runTimeout+5s)` / registry 30s), 출력 캡 3겹.

---

## 6. 게이트웨이측 심화 — 대기 큐·승인창·fanout·정리

- **대기 큐**: `pendingInvokes`가 nodeId AND connId 이중검증으로 재연결·id 충돌 오염 차단.
- **capability-binding 승인창**: system.run invoke 시점에 `(nodeId,connId,sessionKey,runId)` 키로 창을 열고(`timeoutMs+5분 grace`), 노드의 exec.* 이벤트를 이 창과 대조 인증 — **게이트웨이가 요청한 적 없는 exec 알림 주입을 차단**. terminal이면 닫고 실패 invoke는 즉시 회수(`node-registry.ts:157-172,493-563,660-666`).
- **fanout**: node↔session 양방향 Set, 실제 흐르는 건 사실상 **chat 계열뿐**(chat.subscribe 구독 세션 한정). 슬로우 컨슈머는 1008 종료.
- **이벤트 8종**(`server-node-events.ts:389-896`): voice.transcript(1.5s dedupe) / agent.request / notifications.changed / chat.(un)subscribe / exec.started·finished·denied(10분 dedupe) / push.apns.register / node.presence.alive(SQLite lastSeen 60s throttle).
- **disconnect 정리**: unregister→pending reject→승인창 삭제→구독 해제→wake state 클리어→presence 재브로드캐스트(`ws-connection.ts:477-494`).

> **[설계 평가]** 승인창의 수명을 요청 수명에 묶은(terminal 닫기·실패 회수) capability-binding이 정교하다.
>
> **정정(dead code 확정)**: `nodePresenceTimers` Map은 생성·정리만 있고 **`.set()` 호출부가 소스 전체에 없다** — 미사용. voicewake.changed는 fanout이 아니라 connect 직후 개별 노드 직접 push.

---

## 7. 네이티브 앱 접점 — 기기가 노드가 될 때

- **공유 vs 독립**: Apple 두 플랫폼은 `apps/shared` OpenClawKit의 `GatewayNodeSession` actor 재사용, **Android는 완전 독립 Kotlin 구현**(프로토콜 id를 손으로 미러링).
- **caps 차이**: macOS 최소셋(canvas/screen+조건부 browser/camera/location) < iOS 대폭 확장(+talk/photos/contacts/calendar/motion/voiceWake) / Android(+sms/callLog). **exec(system.run)는 데스크톱 전용** — iOS·Android엔 없다.
- **macOS 이중 노드**: 앱 자체가 role=node(canvas/screen을 Swift로) + 별도 headless TS node-host 서비스 병행, system.run은 UDS로 앱에 되위임(`docs/platforms/mac/xpc.md:10-38`) — TCC 안정성(같은 서명 번들)과 원격 자동화를 동시에 얻는 절충.
- **cap vs permission 분리**: caps는 "기능 광고", permissions는 "실제 OS 권한(TCC)" — `PermissionManager`가 TCC 상태를 딕셔너리로 동봉. **권한 없어도 cap은 광고 가능**(실패 판단은 호출 시점).
- **백그라운드 제약**: Android `requiresForeground`→`NODE_BACKGROUND_UNAVAILABLE`, FGS CONNECTED_DEVICE. iOS는 background 진입 시 presence 비콘.
- invoke 타임아웃 latch: 권한 프롬프트가 무한 블록해도 타임아웃이 이기도록 NSLock latch(`GatewayNodeSession.swift:66-140`).

> **[설계 평가]** Apple은 공유물로 표류를 막지만 Android는 3중 유지보수 부담(언어 경계상 불가피). gotcha: macOS 파일명 'Node'가 정반대 두 계열 혼재 — `MacNode*`(앱=노드) vs `NodePairingApprovalPrompter`(게이트웨이 host측).

---

## 8. 종합 — 노드 시스템의 신뢰 모델

**에이전트-준비신뢰 vs 노드-미신뢰 관문:**
- 에이전트 = 게이트웨이 내부의 준비된 신뢰. 노드 = 외부에서 붙는 **미신뢰 주변기기**로, **4겹 관문**을 거친다:
  ① device-pairing 연결 인증 → ② node-pairing 능력-표면 승인(operator 전용) → ③ invoke마다 이중 allowlist(declared ∩ policy) → ④ exec는 노드 자체 재평가까지. 재연결 시 교집합으로 능력 확장 차단.

**강점**
- 신뢰 경계가 계층마다 fail-closed — approved 플래그조차 게이트웨이가 재검증하고 노드가 또 재평가.
- capability-binding 승인창 — 노드의 임의 exec 알림 주입 차단, 창 수명=요청 수명.
- 인증(device)과 인가-표면(node)의 파일 단위 분리.

**거친 부분**
- **command 목록 4중 분산**(코어·Android·iOS·앱) → 표류 위험.
- **auto-review 위임 경로** — 모델 판단이 allowlist-miss를 allow-once로 승격할 수 있는 틈.
- **best-effort 전송** — exec 이벤트/결과가 catch 무시라 단절 시 유실 가능(dedupe는 수신측 책임).
- **레거시 잔재** — `node_pairing_*` SQLite 테이블(런타임 미사용), `nodePresenceTimers`(dead code), macOS runId 폴백.
- **능력 방향 비대칭** — 동기 invoke / 게이트웨이 push / 인바운드 전사가 "노드 능력" 한 이름 아래 섞여 있다.

---

### 부록. 검증 메모
- 핀 유효성: `0fc5a57a..HEAD`는 study 문서만 변경 → `src`/`packages`/`apps`/`extensions` byte-동일(메인 세션 검증).
- 비평 정정 반영: cmd.exe 차단은 **allowlist 모드 한정** / `talk.ptt.*`는 코어 정적 상수(플러그인 경유 아님, canvas만 경유) / `nodePresenceTimers` **미사용 확정** / `scopes:[]` 선언↔자동승인 조건 연결 / 실패 invoke 승인창 즉시 회수 / nodeId+connId 이중검증·이중 타임아웃 교차 확인.
