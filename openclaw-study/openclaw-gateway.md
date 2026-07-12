# OpenClaw Gateway 심층 분석 — 코드 레벨

> 대상: **openclaw/openclaw** `src/gateway/*` (524개 파일) — 코어 control plane.
> 방법: 로컬 소스 직접 정독 + 서브에이전트 4기 병렬 코드 매핑 + `/deep-research` 아키텍처 교차검증(부록 B, 주요 주장 3-0 confirm).
> 코드 링크는 **`openclaw@0fc5a57a`** 에 고정. 클릭하면 해당 파일/라인으로 이동한다(라인 번호는 핀 SHA 기준).
> 같은 디렉터리 [OpenClaw 전체 구조 분석](./openclaw-architecture.md) §3의 **확장판**이다.

---

## 0. TL;DR — 한 문단

게이트웨이는 **항상 켜져 있는 단일 장수 프로세스**로, OpenClaw의 *control plane*이자 *single source of truth*다. 채널·세션·라우팅·승인·노드를 전부 소유하고, 모든 외부 주체(앱·CLI·웹UI·원격 노드·**그리고 에이전트 자신**)는 **WebSocket**으로 여기에 붙는다. 통신은 `connect`/`hello-ok` 핸드셰이크로 시작해 **3종 프레임(`req`/`res`/`event`)** 으로 멀티플렉싱되며, 모든 요청은 **연결에 고정된 role+scope**로 **메서드마다 인가 검사**를 통과해야 실행된다(default-deny). 채널은 WS가 아니라 게이트웨이 **안의 in-process 플러그인**이다. 이 모든 선택은 JSON-RPC/LSP·AIP-151·MCP/OAuth 2.1 같은 외부 정설과 일치한다(부록 B).

---

## 1. Gateway는 무엇이고 왜 존재하나

`src/gateway/server.impl.ts` 헤더 그대로:

> *"Gateway server implementation builds runtime state, method registries, HTTP and WebSocket surfaces, config reload hooks, and graceful restart/shutdown."* — [`server.impl.ts:1`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server.impl.ts#L1)

**왜 단일 허브인가** — 별(star) 토폴로지로 한 프로세스가 모든 상태를 쥐면:
- 권한 검사(scope·승인)를 **한 곳에서 강제**,
- 채널/세션/라우팅 상태가 **한 정본**에 모임,
- 에이전트가 로컬이든 원격 노드든 **동일하게 control plane만 호출**.

```
   [외부 플랫폼: Telegram/Discord/Slack …]
        │ 각 플랫폼 고유 프로토콜
        ▼
  ┌─────────────────────────────────────────────┐
  │              게이트웨이 프로세스              │  ← always-on, SSOT
  │   채널 플러그인(in-process) · 세션 · 라우팅   │
  │   메서드 레지스트리 · broadcast · 승인 매니저 │
  └───────▲─────────────────▲───────────────────┘
          │ WebSocket        │ WebSocket
   ┌──────┴─────┐     ┌──────┴───────┐
   │ 앱/CLI/UI  │     │   에이전트    │ id=gateway-client, mode=backend
   │ 원격 노드  │     │  (tools 콜백) │
   └────────────┘     └──────────────┘
```

---

## 2. 프로세스 부팅 — 진입점부터 listen까지

진입 체인: CLI → 게이트웨이 lazy 파사드 → 구현체 11단계.

| 단계 | 위치 | 하는 일 |
|---|---|---|
| CLI 진입 | [`src/entry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/entry.ts) → [`cli/gateway-cli/run.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cli/gateway-cli/run.ts) → [`run-loop.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cli/gateway-cli/run-loop.ts) | 락 획득, 시그널 핸들러(SIGTERM/INT/USR1) 설치, 감독형 재시작 복구 |
| lazy 파사드 | [`server.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server.ts) | `server.impl`을 **동적 import** 뒤에 둠 → 타입/헬퍼만 쓰는 가벼운 콜러가 전체 startup 의존성 그래프 비용을 안 냄 |
| 구현체 | [`server.impl.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server.impl.ts) `startGatewayServer()` | 아래 11단계 |

**startup 11단계** (모두 `createGatewayStartupTrace()`로 계측):

1. **config.snapshot** — `~/.openclaw/openclaw.json` + 플러그인 메타 로드
2. **config.auth / final-snapshot** — auth 부트스트랩, last-known-good 승격
3. **plugins.bootstrap** — [`server-startup-plugins.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-startup-plugins.ts) 번들 플러그인 + 메서드 레지스트리
4. **runtime.config** — bind host/port, auth mode, TLS, control UI root 해석
5. **tls.runtime** — TLS 설정 시 인증서 로드 ([`infra/tls/gateway.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/tls/gateway.ts))
6. **runtime.state** — `createGatewayRuntimeState()`: HTTP/HTTPS 서버 + WS 서버 + pre-auth 예산 + client registry + broadcast + chat run state
7. **runtime.early** — [`server-startup-early.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-startup-early.ts) 플러그인 discovery, remote skills, 유지보수 타이머
8. **runtime.services** — [`server-runtime-startup-services.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-runtime-startup-services.ts) 채널 health monitor, heartbeat
9. **gateway.handlers / ws-attach** — 코어+플러그인 메서드 descriptor, WS 연결 핸들러 부착
10. **http.listen** — HTTP+WS 서버를 포트에 바인드
11. **runtime.post-attach** — [`server-startup-post-attach.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-startup-post-attach.ts) Tailscale 노출, update 체크, 세션 전송 복구, 채널·cron 시작 → `mark("ready")`

**기본 바인드**: `127.0.0.1:18789` — [`config/paths.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/config/paths.ts) `DEFAULT_GATEWAY_PORT=18789`. bind 모드 loopback/LAN(`0.0.0.0`)/tailscale/auto.

---

## 3. 누가 WS로 붙나 — 클라이언트 vs in-process 채널

핸드셰이크에 등록된 클라이언트 종류 ([`packages/gateway-protocol/.../client-info.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/client-info.ts)):

| 주체 | client id | mode |
|---|---|---|
| **에이전트(툴 콜백)** | `gateway-client` | `backend` |
| 원격 노드 | `node-host` | `node` |
| 사람용 클라 | `cli`/`tui`/`openclaw-macos`/`-ios`/`-android`/`webchat-ui`… | `cli`/`ui`/`webchat` |

**`channel`은 목록에 없다.** 채널은 게이트웨이 *안에서* 플러그인 런타임으로 start/stop된다 — [`server-channels.ts:1`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-channels.ts#L1) *"Gateway channel manager. Starts, stops, restarts, and snapshots plugin channel account runtimes."* 즉 **채널 ↔ 게이트웨이 = in-process 함수 호출**, 외부 통신은 각 플랫폼 고유 프로토콜. (자세히는 [전체 구조 분석 §4](./openclaw-architecture.md).)

---

## 4. 와이어 프로토콜 — 핸드셰이크 + 3프레임

스키마 정본: [`packages/gateway-protocol/src/schema/frames.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts).

**핸드셰이크**: 클라가 `connect`(ConnectParams) → 서버가 `hello-ok`.
- **ConnectParams** ([`:30`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L30)): `minProtocol`/`maxProtocol`, `client{id,version,platform,mode,instanceId}`, `role`, `scopes`, `device{id,publicKey,signature,signedAt,nonce}`, `auth{token,bootstrapToken,deviceToken,password,approvalRuntimeToken}`.
- **HelloOk** ([`:84`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L84)): 협상된 `protocol`, `server{version,connId}`, `features{methods,events}`, `snapshot`(초기 상태), `auth{role,scopes,deviceToken}`, `policy{maxPayload,maxBufferedBytes,tickIntervalMs}`.

**연결 수명 동안 흐르는 3종 프레임** (`type`으로 discriminate):

| 프레임 | 방향 | 필드 | 의미 |
|---|---|---|---|
| `req` ([`:151`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L151)) | 클라→서버 | `id`, `method`, `params` | 요청 |
| `res` ([`:162`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L162)) | 서버→클라 | `id`, `ok`, `payload`, `error` | **요청 id에 짝지어진 응답** |
| `event` ([`:174`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts#L174)) | 서버→클라 | `event`, `payload`, `seq`, `stateVersion` | **id 없는 일방 push** |

추가로 `tick`(하트비트), `shutdown`(종료 공지) 이벤트. `seq`는 단조 증가로 **누락 감지**용, `stateVersion`은 스냅샷 일관성용.
> 외부 대조: 이 req/res/event 3종 = JSON-RPC 2.0/LSP의 request·response·**notification**과 정확히 대응(부록 B).

---

## 5. 요청 처리 파이프라인

진입: [`server-methods.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods.ts) `handleGatewayRequest()`.

```
req 도착
 → authorizeGatewayMethod(method, client, params)   // §8 인가 (server-methods.ts:221)
 → consumeControlPlaneWriteBudget()                 // 쓰기 메서드 레이트리밋(기기/IP당 3/60s)
 → methodRegistry.getHandler(method)                // lazy-load 핸들러 패밀리
 → withPluginRuntimeGatewayRequestScope(...)        // caller 신원을 핸들러로 운반
 → handler 실행 → respond(ok, payload, error)
```

- **lazy 핸들러 패밀리** — `createLazyCoreHandlers()`가 패밀리별 모듈을 **첫 호출 때 로드**(dedupe 캐시). startup 비용/메모리 절감. 광고된 메서드가 모듈에 없으면 에러.
- **request scope** — [`plugins/runtime/gateway-request-scope.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/plugins/runtime/gateway-request-scope.ts) `AsyncLocalStorage`로 `client`/`context`/`pluginId`를 플러그인 핸들러에 전달.
- **control-plane 레이트리밋** — [`control-plane-rate-limit.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/control-plane-rate-limit.ts) sliding-window, `deviceId|clientIp` 키, 버킷 최대 10k(유니크키 DoS 방지).

---

## 6. 2단계 응답(accepted→final) + 이벤트 스트림

오래 걸리는 작업은 `res`가 **두 번** 온다. 클라 처리: [`packages/gateway-client/src/client.ts:1330`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-client/src/client.ts#L1330).

```
res 도착 → pending(그 id) 찾음
  if (expectFinal && payload.status === "accepted") {
      onAccepted(payload); return;   // ★ Promise 안 풀고 대기 유지 (ack)
  }
  pending.delete(id); ok? resolve : reject   // 두 번째 res = 최종 결과
```

서버 쪽 ack 발신은 `twoPhase: true` 핸들러가 `respond(true, {status:"accepted", id, expiresAtMs})` 먼저, 나중에 최종 payload. 대표적 long-running: `agent` 런, `exec.approvals.set`(승인 대기), `plugin.approval.set`.

**독립 event 스트림** ([`client.ts:1314`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-client/src/client.ts#L1314)): `seq`로 gap 감지(`onGap`), `tick`으로 생존 확인, `stateVersion`으로 스냅샷 동기.

```
client                         gateway
  │── req(id=7) ──────────────▶│
  │◀── res(id=7, accepted) ────│   ack → onAccepted(), 대기
  │◀── event(seq=42) ──────────│   id 없는 상태 push
  │◀── res(id=7, ok, final) ───│   최종 → resolve
  │◀── event(seq=43, tick) ────│   하트비트
```

→ **한 소켓 위 이중 멀티플렉싱**(id 기반 2단계 응답 + seq 기반 일방 이벤트)이 HTTP 요청-응답 1:1로 표현 불가 → WS 채택의 근본 이유.
> 외부 대조: accepted→final = Google AIP-151 long-running Operation(=HTTP 202+polling) 패턴(부록 B).

---

## 7. 인증 (Authentication) — 연결 신원 확정

연결 시점에 자격증명을 검증해 **granted role+scopes**를 박는다.

- **auth 해석** — [`auth-resolve.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/auth-resolve.ts) `resolveGatewayAuth()`: config+override+env → mode(token|password|none). [`auth-mode-policy.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/auth-mode-policy.ts)는 token+password 동시 설정 같은 모호함을 **fail-closed** 거부.
- **자격증명 종류** — shared secret(token/password), **device token**(verifyDeviceToken: 발급자·role·scope 일치 확인), **bootstrap token**(페어링·만료·프로필 확인).
- **device identity** — [`infra/device-identity.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/device-identity.ts) Ed25519 키쌍(`device.json`). connect의 `device.signature`를 publicKey SHA256 fingerprint로 deviceId 유도 + 서명 시각(clock skew)·nonce 신선도 검증. 페어링+승인 후 device token 발급/회전.
- **레이트리밋** — [`auth-rate-limit.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/auth-rate-limit.ts) sliding-window(기본 10회/60s/300s 잠금), loopback 면제. bootstrap verify는 추가로 mutex+fs 직렬화(정상 온보딩을 공격 큐 뒤로 밀리지 않게).

**granted scopes의 출처**: 기본 빈 배열(**default-deny**). 페어링된 device, device token, 또는 plugin 등록에 바인딩될 때만 채워진다.

---

## 8. 인가 (Authorization) — 메서드마다 강제

권한은 **연결에 고정**(`client.connect.role`/`scopes`)되고, 게이트웨이가 **매 요청 디스패치 직전 검사**한다.

**role 모델** ([`role-policy.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/role-policy.ts)): `operator`(기본, operator RPC) vs `node`(노드 발신 메서드). `parseGatewayRole()`이 untrusted 입력을 닫힌 집합으로 좁힘.

**operator scope** ([`operator-scopes.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/operator-scopes.ts)): `operator.admin` / `.read` / `.write` / `.approvals` / `.pairing` / `.talk.secrets`. `isOperatorScope()`로 닫힌 집합 narrow.

**메서드→스코프** ([`method-scopes.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/method-scopes.ts)):
- `resolveLeastPrivilegeOperatorScopesForMethod()` ([`:149`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/method-scopes.ts#L149)) — 그 메서드에 필요한 **최소** 스코프. 미분류 메서드는 `[]` 반환 → *"Default-deny for unclassified methods."* ([`:160`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/method-scopes.ts#L160))
- `authorizeOperatorScopesForMethod()` ([`:165`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/method-scopes.ts#L165)) — granted ⊇ required? admin은 통과, READ는 WRITE로 승급. 미분류는 `ADMIN_SCOPE` 요구(fail-closed).

**서버 강제 지점** ([`server-methods.ts:221`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods.ts#L221) `authorizeGatewayMethod`, 디스패치 전 매번 호출):

```
if (!client.connect) → health 등 pre-connect만 통과
role = parseGatewayRole(connect.role ?? "operator")  → 실패 시 "unauthorized role"
if (!isRoleAuthorizedForMethod(role, method))         → "unauthorized role"
if (role === "node") return ok                         // 노드 전용 경로
if (scopes.includes(ADMIN_SCOPE)) return ok
scopeAuth = authorizeOperatorScopesForMethod(...)      → 실패 시 "missing scope: X"
```

→ **유일 통로 + 연결 바인딩 + 매호출 검사 + default-deny** 네 겹이라야 "강제"가 성립. in-process였다면 넷 다 자동으로 안 걸린다.
> 외부 대조: scope를 handshake에 바인딩 + per-method 검사 + default-deny + 부족 시 거부 = MCP/OAuth 2.1 모델(`insufficient_scope`)과 동형(부록 B).

---

## 9. 에이전트 = backend WS 클라이언트

에이전트의 내장 툴이 게이트웨이를 역호출하는 어댑터: [`src/agents/tools/gateway.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/tools/gateway.ts).

- `callGatewayTool(method, opts, params, {expectFinal, scopes})` → URL/토큰/스코프 결정 후 [`gateway/call.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/call.ts) `callGateway`로 위임. `clientName=gateway-client`, `mode=backend`, `displayName="agent"`.
- 행동마다 `resolveLeastPrivilegeOperatorScopesForMethod`로 **딱 필요한 최소 스코프만** 든 일회성 연결.
- **보안 검증**(이 파일의 절반):
  - URL 허용목록 — loopback 또는 설정된 `gateway.remote.url`만. URL 내 자격증명·path·query 차단.
  - local/remote 타깃 구분 — env 선택 URL은 remote로 강등(loopback이라도 local 승인 권한 불부여).
  - 승인 메서드(`exec.approval.*`/`plugin.approval.*`)는 local일 때만 승인 런타임 토큰, remote일 때만 영속 device identity 부착.

**왜 embedded인데도 loopback WS?** — 에이전트는 모델 출력으로 동작(인젝션 표면)하고 위치(embedded/원격 node)가 바뀌며 죽을 수 있는 행위자라, 직렬화 비용을 감수하고 **인증·최소권한·끊김감지 되는 동일 경계** 뒤에 둔다.
> 외부 대조: 프롬프트 인젝션은 모델 탐지로 못 막으므로 **구조적 least-privilege·격리**로 제약해야 한다는 게 학계 합의(Beurer-Kellner 2506.08837, PFI 2503.15547) — 부록 B.

---

## 10. 메서드 서피스 (패밀리)

코어 핸들러는 [`server-methods/`](https://github.com/openclaw/openclaw/tree/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods) 디렉터리에 패밀리별로. 주요 패밀리:

| 패밀리 | 대표 메서드 | 용도 |
|---|---|---|
| connect / health | `connect`, `health`, `status` | 핸드셰이크, 상태·채널 health |
| channels | `channels.{status,start,stop,logout}` | 채널 플러그인 생명주기 |
| chat / agent | `chat.{send,history,abort}`, `agent`, `agent.wait` | 메시지 전송·전사, 에이전트 런 |
| sessions | `sessions.{list,send,steer,abort,reset,delete,compact}` | 세션 스토어·디스패치 |
| send | `send`, `message.action`, `poll` | 원시 메시지 전송(outbound) |
| cron | `cron.{list,add,update,remove,run}`, `wake` | 스케줄 작업 |
| node / device | `node.{pair,list,invoke,event}`, `device.{pair,token}` | 원격 노드·기기 페어링/RPC |
| exec.approvals / plugin.approval | `exec.approvals.{get,set}`, `plugin.approval.{get,set}` | 명령/플러그인 승인 (2단계 응답) |
| models / config / tools | `models.list`, `config.{get,set,patch}`, `tools.{catalog,invoke}` | 모델 카탈로그·설정·툴 |
| talk / tts / voicewake | `talk.*`, `tts.*`, `voicewake.*` | 음성 세션·TTS·웨이크워드 |
| system / logs / update / restart | `system-presence`, `logs.tail`, `update.run`, `gateway.restart.request` | 운영 |

**플러그인 소유 메서드**는 `activeRegistry.gatewayMethodDescriptors`로 등록되고 `normalizePluginGatewayMethodScope()`로 admin 사칭을 막는다(코어 scope 가드).

---

## 11. 이벤트 / 구독 / 스냅샷

**broadcast** ([`server-broadcast.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-broadcast.ts)):
- **per-client 단조 seq** — 비-타깃 broadcast마다 증가, 타깃 이벤트는 seq 미증가(거짓 gap 방지).
- **이벤트 scope 가드** — `EVENT_SCOPE_GUARDS`가 이벤트별 필요 스코프 선언 → role 기반 필터(권한 없는 클라엔 안 보냄).
- **느린 소비자** — ws 버퍼가 `MAX_BUFFERED_BYTES` 초과 시 drop 또는 연결 종료.

**구독** — 세션/노드별 fanout 레지스트리([`server-node-subscriptions.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-node-subscriptions.ts), `server-chat-state.ts`). 양방향 인덱스(session↔connId, node↔session)를 원자적으로 정리해 stale 참조 방지.

**스냅샷** ([`schema/snapshot.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/snapshot.ts)): hello-ok에 **초기 상태 1회** 전달(presence 목록 + health + `stateVersion{presence,health}` 카운터), 이후 event의 `stateVersion` 힌트로 클라가 캐시 무효화·재조회.

---

## 12. 세션 & 노드

**세션 스트리밍** ([`server-chat.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-chat.ts), [`server-chat-state.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-chat-state.ts)):
- `ChatRunState`가 run별 버퍼(final text/raw/delta)를 들고, **이미 broadcast한 텍스트 길이**를 추적해 중복 flush를 막음.
- 에이전트 이벤트(delta/final/tool/lifecycle)를 chat·session 스트림으로 투영 → `broadcastToConnIds("chat", …)`.
- `ToolEventRecipientRegistry` — 어떤 connId가 그 run의 tool 이벤트를 받을지(10분 TTL + 30초 final grace).

**노드(원격 백엔드)** ([`node-registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/node-registry.ts), [`server-node-events.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-node-events.ts)):
- `node-host`(mode=node)가 caps/commands/permissions를 선언하며 연결 → `NodeRegistry`에 NodeSession 등록.
- 노드 발신 이벤트(음성 전사, exec 완료, presence)는 dedupe(전사 eventId 1.5s 창, exec.finished runId 10분 창) 후 세션 구독자로 라우팅.
- 연결 해제 시 pending invoke 거부 + 인덱스 정리.

**에이전트 런 종료 상태** ([`agents/agent-run-terminal-outcome.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/agent-run-terminal-outcome.ts)): `hard_timeout`·`cancelled`는 **sticky**(나중 관측이 못 덮음), timeout/cancel/complete 우선순위를 한 곳에서 정규화.

---

## 13. 생명주기 — health / restart / shutdown

- **채널 health monitor** ([`channel-health-monitor.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/channel-health-monitor.ts)): 기본 5분 주기, stale 30분/connect grace 2분 임계로 running·stuck·stale socket·reconnect 평가.
- **restart** ([`infra/restart.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/restart.ts)): SIGUSR1 정책 + **deferral**(pending 작업 drain까지 재시작 연기). [`server-restart-sentinel.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-restart-sentinel.ts)가 재시작 후 pending continuation·outbound 전송 재개.
- **graceful shutdown** ([`server-close.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-close.ts)): discovery→tailscale→채널 stop→세션 drain([`active-sessions-shutdown-tracker.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/active-sessions-shutdown-tracker.ts))→cron→heartbeat→broadcast 정리→WS/HTTP 종료. 각 단계 timeout race + 경고 수집.

---

## 14. 관통하는 설계 원칙 (왜 이렇게)

1. **단일 control plane / SSOT** — 상태 분산 금지. 모든 행동이 한 프로세스의 메서드 호출로 수렴.
2. **신뢰 경계로 나눔** — 신뢰되는 채널은 in-process, 모델이 모는 에이전트는 인증된 WS 뒤.
3. **인가는 항상 서버·항상 메서드 단위·항상 default-deny** — 클라의 최소권한 요청은 보너스, 강제는 서버.
4. **연결에 권한 고정** — 호출마다 권한 위조 불가.
5. **이중 스트림 멀티플렉싱** — id 2단계 응답 + seq 이벤트를 한 소켓에. 그래서 HTTP 아닌 WS.
6. **lazy 부팅** — 파사드 동적 import + 패밀리별 핸들러 지연 로드로 startup 비용 절감.
7. **process-stable 메타 + 핫패스 freshness 폴링 금지** — 설치/매니페스트/discovery는 재시작/doctor로만 갱신.

---

## 부록 A. 핵심 파일 지도

| 관심사 | 파일 |
|---|---|
| 진입/부팅 | [`cli/gateway-cli/run-loop.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/cli/gateway-cli/run-loop.ts) · [`server.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server.ts) · [`server.impl.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server.impl.ts) |
| 프로토콜 | [`schema/frames.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/frames.ts) · [`client-info.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/client-info.ts) · [`schema/snapshot.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-protocol/src/schema/snapshot.ts) |
| 클라이언트 | [`gateway-client/src/client.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/gateway-client/src/client.ts) · [`gateway/call.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/call.ts) · [`agents/tools/gateway.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/tools/gateway.ts) |
| 디스패치/메서드 | [`server-methods.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods.ts) · [`server-methods/`](https://github.com/openclaw/openclaw/tree/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods) |
| 인증/인가 | [`auth-resolve.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/auth-resolve.ts) · [`method-scopes.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/method-scopes.ts) · [`role-policy.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/role-policy.ts) · [`operator-scopes.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/operator-scopes.ts) |
| 이벤트/세션/노드 | [`server-broadcast.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-broadcast.ts) · [`server-chat.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-chat.ts) · [`node-registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/node-registry.ts) |
| 생명주기 | [`channel-health-monitor.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/channel-health-monitor.ts) · [`server-close.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-close.ts) · [`infra/restart.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/restart.ts) |

## 부록 B. 교차검증 (deep-research)

`/deep-research`로 게이트웨이의 아키텍처 선택을 외부 분산시스템·보안 정설과 대조했다. **핵심 주장 대부분 3-0 confirm** — 즉 이 게이트웨이는 새 발명이 아니라 확립된 패턴의 조합이다.

| 우리 코드의 설계 | 외부 정설 | 출처 | 판정 |
|---|---|---|---|
| WS 한 연결로 RPC + 서버 push 멀티플렉싱; req/res/event 3종 | JSON-RPC 2.0 / LSP의 request·response·**notification(event)**; notification은 id 없고 응답 없음, request는 반드시 응답 | [jsonrpc.org/specification](https://www.jsonrpc.org/specification) · [LSP 3.17](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/) | ✅ 3-0 |
| `id`로 요청-응답 상관 | JSON-RPC: response는 요청 `id`를 그대로 echo해 상관(병렬/순서무관 처리 허용) | jsonrpc.org | ✅ 3-0 |
| `res`는 ok/error 중 하나 | JSON-RPC: result/error 정확히 하나만 포함 | jsonrpc.org | ✅ 3-0 |
| accepted→final 2단계 응답 | Google AIP-151 long-running Operation(=future/promise 핸들 + 폴링, HTTP 202+polling 등가) | [google.aip.dev/151](https://google.aip.dev/151) | ✅ 3-0 |
| handshake에 scope 바인딩 + per-method 검사 + default-deny + 부족 시 거부 | MCP/OAuth 2.1 least-privilege; 스코프 메타데이터 광고, route/tool별 검증, `403 insufficient_scope` | [MCP authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization) · [MCP security](https://modelcontextprotocol.io/docs/tutorials/security/authorization) | ✅ 3-0 |
| embedded 에이전트도 인가 경계(loopback WS) 경유 | 프롬프트 인젝션은 모델 탐지로 신뢰 불가 → **구조적 least-privilege·default-deny·trusted/untrusted 격리**로 제약해야 한다는 학계 합의 | Beurer-Kellner et al. arXiv:2506.08837 · PFI arXiv:2503.15547 | ✅ |
| WS heartbeat(`tick`)로 생존/presence | WebSocket Ping/Pong이 장수 연결의 표준 하트비트 | RFC 6455 | ✅ |

> 요지: OpenClaw 게이트웨이의 "WS 컨트롤플레인 + 3프레임 + 2단계 ack + handshake-bound scope + 구조적 인가 경계"는 각각 LSP/JSON-RPC, AIP-151, MCP/OAuth 2.1, 인젝션 보안 연구의 정설과 1:1로 대응한다.
