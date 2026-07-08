# OpenClaw 채널 플러그인 심층 분석 — 골격 안쪽 구현 (시스템 편 속편)

> ultracode 멀티에이전트(서브에이전트 Opus 4.8)로 개별 채널 플러그인 구현 7영역을 병렬 매핑 → 정확성 비평 → 종합.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src`/`packages`/`extensions`를 변경하지 않아 working tree == 이 SHA로 검증됨).
> 선행 문서: [channels-plugins-study.md](./channels-plugins-study.md)(시스템 편 — 골격·로더·경계). 이 속편은 그 골격 함수가 호출된 **안쪽** — 각 채널이 transport를 어떻게 조립하고, inbound를 facts로 바꾸며, outbound를 플랫폼 제약에 맞춰 내보내는가를 본다.
> **시스템 편 정정 포함**: "단일 골격, 예외 없음" 주장은 과장 — qqbot이 골격 *함수* 미경유(계약 *타입*은 준수) 반증 존재(§5).

---

## 1. Telegram — 골격 안쪽 구현

### 조립 체인: channel → monitor → grammY Bot
- [`extensions/telegram/src/channel.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/extensions/telegram/src/channel.ts)`:686-1177`: `createChatChannelPlugin` 인스턴스화. `startAccount`에서 `getMe` probe → botInfo **24h 캐시**(실패 시 캐시 폴백, 401이면 캐시 삭제+throw) → `monitorTelegramProvider`.
- `bot-core.ts:108-423`: 미들웨어 **순서대로** use — account throttler(그룹 fair-queue, `account-throttler.ts:18-159`) → update-dedupe → sequentialize(`sequential-key.ts` — control/btw/approval/topic/chat 레인 분리) → raw-update-log.

### 폴링 이중 모드 + isolated worker ingress
- `monitor.ts:169-205,301-302`: `useWebhook` 분기로 webhook/polling 런타임 지연 import. 폴링 기본값은 **isolated ingress**(grammY runner가 기본이 아님).
- `telegram-ingress-worker.ts:87-99`: `getUpdates`를 **worker_thread**에서 실행 → 스풀 디렉터리 기록 → 부모가 lane별 drain해 `bot.handleUpdate`(`polling-session.ts:781-1076`). classic runner는 fallback(`:1078-1260`).
- → **에이전트 처리 지연이 폴링을 막지 않게 근본 분리.** dead-letter·stale-claim 회복까지(`:618-779`).

### 재시도·stall 복구
- 지수 backoff(30s~600s, jitter 0.2, `polling-session.ts:53-58`) + stop-timeout burst 쿨다운. 30s watchdog가 stall 감지 → **transport dirty**(keep-alive 소켓 폐기) → 재시작(`:989-1005`). 409 conflict → webhookCleared 리셋 + 소켓 교체 재시도(`:1018-1021`, #69787).

### inbound → facts
- `bot-handlers.runtime.ts:479-965`: debounce(포워드 버스트 80ms 별도 레인)·media-group 버퍼·text-fragment 재조립 → reply-chain·prompt-context → `processMessage`.
- `bot-message-context.ts:162-233,498`: forum flag·topic name 캐시·DM/reply thread 해석. topic thread id는 코어엔 `chatId:topicId` canonical, Telegram send 시에만 numeric 환원.

### outbound: HTML 변환·청크·버튼
- `send.ts:671-776`: 마크다운→HTML → `splitTelegramHtmlChunks(4000)` — **열린 태그를 청크 끝에서 닫고 다음 청크 앞에서 다시 열어** 마크업 보존(`format.ts:645-757`). HTML parse 실패 시 plain fallback.
- **버튼 콜백 인코딩 비대칭(정밀화)**: `native-command-callback-data.ts` — `tgcmd:` native 명령은 checksum **없이 `/` 프리픽스 검사만**(`:18`), `tgcb1:` opaque value만 fnv1a 5자 checksum 보호(`:35-52`). "checksum이 모든 버튼을 방어" ✗ — **opaque value만**.

### native 명령·권한·doctor
- `bot-native-commands.ts:724-834`: nativeSkills면 agent route 바인딩 후 skill 명령 등록, 총량 초과 시 per-skill 빼고 `/skill`만 유지. `bot-native-command-menu.ts:492-549`: 스코프별 delete→set, 해시 동일 시 429 회피 skip.
- allow-from: `telegram:`/`tg:` 프리픽스 제거 후 **숫자 sender id만**(`allow-from.ts:2-18`). DM은 pairing challenge(`dm-access.ts:80-131`), 그룹은 base+policy+topic override 이중 검사.
- doctor(`doctor.ts:146-604`): @username→numeric id 리페어 등. `state-migrations.ts:238-605`: legacy state 감지→import→삭제. `update-offset-store.ts:100-160`: botId/token 지문으로 rotation 감지, stale offset 폐기.

> **[설계 평가]** 4채널 중 운영 견고성이 가장 두껍다. 강점: worker-thread ingress + 파일 스풀 + lane 복구의 근본 분리, HTML 청크의 태그 보존. 트레이드오프: 409/stall마다 transport 통째 재구축은 실증적 해법이나 실패 분류·backoff·쿨다운·타이머가 얽힌 상태기계 복잡도가 상당. 거친 부분: `bot-handlers.runtime.ts` 3000줄 단일 클로저에 debounce·media-group·fragment·권한 공존 — dispatch dedupe claim/commit/release 누수 표면이 넓다.

---

## 2. Discord — 자체 게이트웨이/REST/voice

### discord.js 없는 자체 WS 게이트웨이
- `internal/gateway.ts:233-295,384-407`: `ws` + `discord-api-types`(타입 전용)로 hello/heartbeat/identify/**resume**/invalid-session opcode 직접 처리. Ready에서 `session_id`/`resume_gateway_url` 저장. close code fatal/non-resumable 분류(`gateway-close-codes.ts:20-26`).
- **런타임 discord.js 의존 없음(확증)**: deps는 `@discordjs/voice`(음성)·`discord-api-types`(타입)뿐.

### 삼중 rate-limit 방어
| 층 | 위치 | 방식 |
|---|---|---|
| 게이트웨이 send | `gateway-rate-limit.ts:19-26` | 120/60s 슬라이딩, heartbeat/resume critical bypass |
| identify | `gateway-identify-limiter.ts:16-49` | 버킷별 5s 게이트 — **모듈 싱글턴**(전 계정/샤드 공유) |
| REST | `rest-scheduler.ts:305-550` | `X-RateLimit-Bucket`별 동시성/reset + 3레인 가중 RR + 429 requeue |

### 멱등 slash 배포 + CJK 정규화
- `command-deploy.ts:143-185`: 명령셋 stable 해시를 **앱ID 스코프** 캐시에 저장, 변경분만 배포(#77359). `:410-421` **Discord 서버측 CJK/공백 정규화를 로컬 비교에 역이식**해 매 시작 spurious PATCH 429 억제.

### voice = 위임 + 브릿지
- `voice.ts:28-49`: `@discordjs/voice`의 `adapterCreator.sendPayload`를 자체 게이트웨이 `send(critical)`로 브릿지 — voice 프로토콜은 자체 구현 아님. `manager.ts`(~1900줄): join/leave·autoJoin·followUsers(REST 예산 reconcile)·DAVE 암호화·realtime vs stt-tts.

### inbound/outbound + registerFull
- `message-handler.context.ts:121-430`: DM/길드/스레드/포럼 → facts. runtime 함수 필드는 payload와 분리(`inbound-job.ts:46-98`, 큐 직렬화 안전).
- `chunk.ts:158-259`: 2000자/17줄 **코드펜스 밸런싱** 청커 + webhook 페르소나 전송(실패 시 일반 폴백).
- `index.ts:23-27`: registerFull은 subagent thread-binding 훅 + voice transcript provider **둘만** — 얇은 표면.

> **[설계 평가]** discord.js를 버린 건 rate-limit·startup stagger·멱등 배포·intents 프로빙을 세밀하게 쥐기 위한 의도 — 429가 3계층+해시 캐시+stagger로 구조 방어된다. CJK 정규화 역이식은 실 429 리포트에서 역산한 강점이자 정규식 2회 replace라는 취약한 휴리스틱. 대가는 `internal/` 유지보수 표면 최대. inbound/outbound는 골격 최대 재사용 + discord 고유만 얹은 깔끔한 경계.

---

## 3. WhatsApp — Baileys 얇은 래핑 + 다층 방어

### 세션 수명
- `session.ts:158-228`: `useMultiFileAuthState`→`makeWASocket`. browser `['openclaw','cli',VERSION]`, `syncFullHistory=false`, `markOnlineOnConnect=false` 고정.
- **`session.runtime.ts:1-9`: 모든 Baileys 심볼 단일 재수출 배럴** — 의존 경계 한 곳.
- `connection-controller.ts:216-296,460-724`: 515(post-pairing)·408(timeout) 각 1회 소켓 재생성, 401은 `logoutWeb`+relink. heartbeat/watchdog 두 타이머, 무활동 시 499 forceClose. **`LOGGED_OUT`만 enum 폴백(`?? 401`), 408/515/499는 하드코딩**.

### auth 영속·손상 복구
- `auth-store.ts:43-45,205-216`: `<oauthDir>/whatsapp/<accountId>`에 creds.json + baileys 키 파일들. `creds-persistence.ts:13-63`: authDir별 직렬 큐 + `BufferJSON` + **0o600 원자 교체** + `.bak` 백업/복원.

### QR 페어링·미디어·degrade
- `login-qr.ts:243-576`: activeLogins 버전 관리 + QR/연결/실패 3중 race + `waitForWebLogin` 폴링(TTL 3분).
- **audio-decode는 간접 의존(확증)**: src import 0회 — Baileys optional peer로서 **outbound PTT 보이스노트 파형 생성을 Baileys가 수행**할 때 쓰임.
- **outbound degrade는 이중 조건 게이트(정밀화)**: `outbound-base.ts:234-256` — `!text && !media`이면서 `interactive||presentation||channelData` 있을 때**만** throw. text+버튼 혼합은 통과(text만 전송). "구조화 페이로드 무시" ✗.
- `pluginHooks.messageReceived` opt-in(account>channel>false), 훅 실패는 `fireAndForgetBoundedHook` 격리. doctor는 의존성 점검이 아니라 **legacy 설정 마이그레이션 전용**(`doctor.ts:8-57`).

> **[설계 평가]** 비공식 rc 라이브러리(Baileys) 리스크를 세 겹으로 방어 — ① 전 심볼 배럴 격리(교체 지점 단일화) ② `typeof===function` 폴백으로 API 드리프트 흡수 ③ creds 백업·원자 저장으로 라이브러리 밖 상태 손상 방어. 트레이드오프: 수명 로직이 status 매직넘버(401/408/515/499)에 강결합인데 폴백은 401 하나뿐 — Baileys가 코드 의미를 바꾸면 재시작/로그아웃 분기가 조용히 틀어질 수 있다.

---

## 4. Slack · Signal — 무게가 극단적으로 다른 두 채널

### Slack
- **Socket vs HTTP 단일 코드경로**: `provider-support.ts:325-331` — mode(기본 socket)로 `SocketModeReceiver`(appToken) vs `HTTPReceiver`(signingSecret+webhookPath). socket은 재접속 while 루프 상주, http는 abort 대기만 하고 라우팅은 코어 HTTP 서버(`provider.ts:546-652`).
- **HTTP 라우팅**: registerFull이 계정별 webhookPath를 `api.registerHttpRoute(auth:'plugin')` 등록(`plugin-routes.ts:23-31`) → **글로벌 Symbol Map**에서 Bolt requestListener로 위임(`registry.ts:46-57`). block action은 최우선 `await ack()`(3초 제한).
- block kit: `app.action(/.+/)` 전 액션 포착 → `summarizeAction` 정규화 → 서명 검증 userId actor-binding, exec-approval 우선(`interactions.block-actions.ts:186-284,886-970`).
- **Bolt 내부 재접속 Reflect 몽키패치**(`provider-support.ts:56-114`) — ping 타임아웃 누수 방지, 단 SDK 업그레이드 강결합.

### Signal
- signal-cli **자식 프로세스 spawn**(`daemon.ts:89-119`) + JSON-RPC POST/`/api/v1/events` SSE 수동 파싱(`client.ts:196-435`).
- **native vs container 자동 감지(확증)**: `client-adapter.ts:135-171` — `apiMode='auto'`면 Promise.any 프로브, **native 50ms grace 우선**, 30s TTL 캐시. container=WebSocket / native=SSE. `autoStart && container`는 상호배타 예외.

> **[설계 평가]** 같은 골격 위에서 무게가 극단 분화 — Slack 200+ 파일(Bolt 이벤트/interaction/modal 인하우스) vs Signal ~40파일(프로토콜을 signal-cli에 위임, 대신 외부 바이너리·프로세스 수명 부담). Slack 글로벌 Symbol Map은 control-plane과 runtime의 로드 타이밍 차이 때문이나 중복 등록 시 조용한 no-op이라 멀티계정이 path를 안 나누면 한 계정만 산다(거친 부분).

---

## 5. 채널 스펙트럼 전수 (bundled 19개)

bundled 판정은 package.json 플래그가 아니라 **catalog `origin:"bundled"`**(`src/channels/plugins/bundled-ids.ts:14-42`).

### transport 5계열
| 계열 | 채널 |
|---|---|
| 웹훅 inbound | line · sms(Twilio) · synology-chat · nextcloud-talk(HMAC) · googlechat · msteams · zalo |
| WebSocket | mattermost · clickclack · qqbot · twitch(@twurple) |
| 폴링/SDK-sync | matrix(matrix-js-sdk) · **imessage(chat.db SQLite 폴링 + osascript)** |
| 서브프로세스 | signal(signal-cli) |
| raw 소켓 | irc(net/tls 6667/6697) |
| 이중 | feishu(websocket 기본/webhook) |

### 크기 30배 편차 (비-test LOC)
- 최소: clickclack 1.4k < sms 2.2k < synology 2.9k < irc 3.5k < twitch 3.6k …
- 최대: imessage 14.7k < msteams 19.7k < **qqbot 24.5k** < feishu 28.6k < **matrix 40.6k**(WASM crypto).
- 얇은 쪽 = 3rd-party SDK 위임/단순 웹훅, 두꺼운 쪽 = 자체 엔진(qqbot engine/stages 파이프라인, matrix crypto).

### 골격 채택 — 정정된 규칙 ★
- **18개는 `createChatChannelPlugin` 함수 경유. qqbot은 예외** — 골격 함수를 한 번도 호출하지 않고 `ChannelPlugin<ResolvedQQBotAccount>` 객체를 **직접 리터럴 구성**(`extensions/qqbot/src/channel.ts:206`, grep 무결과 확증). 단 계약 타입은 준수하므로 `gateway.startAccount` 균질화는 유지.
- → 정확한 표현: **"계약 타입은 필수, 골격 함수는 권장"**. 시스템 편의 "예외 없는 규칙"은 과장.

### plugin-sdk 헬퍼 재사용 밀도 (import 빈도)
string-coerce 226 · config-contracts 182 · channel-outbound 106 · ssrf-runtime 86 · number-runtime 85 · channel-contract 66 · reply-runtime 57 · channel-inbound 49.

> **[설계 평가]** 19개를 계약 타입 + 헬퍼로 수렴시켜 transport 제각각이어도 매니저 관점은 균질(재사용 밀도가 정량 뒷받침). 그러나 ① 골격 안쪽 크기가 30배 벌어져 단일 골격이 구현 복잡도까지 평준화하진 못하고 ② imessage(Full Disk Access)·signal(외부 데몬)·zalouser(역공학 클라이언트) 같은 OS/프로세스 제약은 골격이 감추지 못한다.

---

## 6. setup / doctor / config 계약 (manifest-first)

- **매니페스트-우선 선언**: telegram `package.json:17-25` — `setupEntry` + `setupFeatures{configPromotion, legacyStateMigrations}`. 코어는 런타임 로드 전 이 **순수 데이터 플래그**로 게이팅(`src/plugins/manifest.ts:1958-1990`).
- **경량 setup 엔트리**: `defineBundledChannelSetupEntry`(`channel-entry-contract.ts:569-627`)가 채널 전체 import 없이 lazy 로더만 전달.
- **채널-소유 doctor 수리 = 2심볼 공개 표면**: `telegram/doctor-contract-api.ts:1-2` — `normalizeCompatibilityConfig`·`legacyConfigRules`만 re-export. 코어는 이 좁은 아티팩트만 로드(`src/channels/plugins/doctor-contract-api.ts:34-51`) — 루트 AGENTS.md의 "plugin-owned config repairs" 실체.
- **병합 체인**: contract-api → bootstrap plugin.doctor → 외부 registry 폴백(`legacy-config.ts:94-122`), 4소스 `mergeDoctorAdapters`("먼저 온 것 승").
- **doctor는 두 종류**: config 수리(파일 변형)와 **state 수리(on-disk 변형, `legacyStateMigrations`)** — 서로 다른 계약·로더. 내부 로직은 채널 고유(discord는 top-level `bindings[]` 생성, telegram은 안 함).
- `channelEnvVars`(telegram→`TELEGRAM_BOT_TOKEN`)는 `--use-env` 온보딩 경로가 소비(`channel-env-vars.ts:41-62`).

> **[설계 평가]** "코어가 무거운 채널 런타임을 로드하지 않고 config를 고친다"를 2심볼 최소 표면으로 강제한 것이 핵심 — control-plane/runtime-plane 분리와 정확히 일치, cold doctor/setup 경로가 가볍다. 트레이드오프: 3소스 로드·병합 + "먼저 온 것 승" + 규칙 dedupe가 얽혀 "어느 소스가 이겼나" 추적이 어렵다.

---

## 7. 채널 관통 미디어·음성 파이프라인

- **경계**: 채널은 "바이트→로컬 경로"까지만(whatsapp `saveMediaStream` 50MB, telegram SDK 재노출), 전사·설명·크기게이트·body 주입은 전부 코어 media-understanding. **iMessage만 예외** — 네이티브 `~/Library/Messages/Attachments` in-place 읽기(다운로드 없음), `resolveInboundAttachmentRoots` 구현체는 사실상 iMessage 하나를 위해 존재.
- **크기 상한 삼중 계층(확증)**: media store 5MB / whatsapp inbound 50MB / **오디오 이해 20MB**(`packages/media-understanding-common/src/defaults.ts:22`) — 큰 파일은 저장되나 전사에서 skip 가능.
- **전사 파이프라인**: image→audio→video 순 `runCapability`(`apply.ts:52,556-573`), auto-detect는 active model→로컬 CLI(sherpa/whisper)→provider auth. 성공 시 Body를 `[Audio]` 치환 + `{{Transcript}}`/CommandBody 세팅(슬래시 명령 유지).
- **voice note mention preflight**: 그룹 requireMention 통과용 첫 오디오 선전사(`audio-preflight.ts:18-73`).
- **outbound**: `resolveMaxBytes` 계정→에이전트 폴백. WhatsApp은 ffmpeg opus/ogg(48kHz mono) 트랜스코드, Telegram은 voice 비호환 시 audio 파일 폴백.
- **Discord realtime 음성은 독립 서브시스템**: wav 세그먼트→전사→에이전트 턴→TTS→opus 재생 큐 — "첨부→이해" 모델과 다른 축, 재사용은 `transcribeAudioFile`/TTS seam 수준.

> **[설계 평가]** 채널이 서로 다른 전송 API를 가져도 이해 파이프라인은 코어 하나로 수렴하는 깔끔한 경계. 가독성 함정: 오디오 상한이 계층마다 다른 의미(5/50/20MB)라 "왜 전사가 안 되지"의 오진을 부르기 쉽다.

---

## 8. 종합 — 채널 구현들이 보여주는 패턴

### 코어로 흡수된 것 (수렴)
계약 타입+gateway 수명주기(19/19) · inbound facts 조립 · typed presentation+청킹 · 미디어 이해 전체 · doctor 최소 표면 로딩 · 방어 헬퍼(ssrf/string-coerce).

### 플랫폼에 남은 것 (고유)
transport 프로토콜(롱폴/자체WS/Baileys/Bolt/데몬/SQLite폴링/raw소켓) · 플랫폼 제약 정면 처리(TG HTML 청크, Discord 버킷·CJK, WA creds 원자성, Signal 모드 감지) · 채널-소유 doctor 로직 · 골격 밖 서브시스템(Discord realtime voice, qqbot 엔진, matrix crypto).

### 강점
- **경계 규율** — control-plane(2심볼 doctor/setup)과 runtime-plane 분리가 실제로 지켜져 cold 경로가 가볍다.
- **의존 리스크 격리** — WA baileys 배럴+폴백, Discord 자체 소유 — 외부 라이브러리 드리프트 흡수층이 명확.
- **재사용 밀도** — 헬퍼 import 빈도가 골격 실효성을 정량 뒷받침.

### 위험
- **골격 프레이밍 과신** — qqbot 반증: 계약 타입은 필수, 골격 함수는 권장일 뿐.
- **상태기계 복잡도** — TG 폴링 dirty-transport, WA status 매직넘버, Slack Bolt 몽키패치는 견고하나 회귀/업그레이드 취약 표면.
- **doctor 병합 추적성** — 3소스 폴백 + "먼저 온 것 승"의 승자 추적 어려움.
- **계층 분산 가독성** — 미디어 삼중 상한, 채널별 doctor 편차.
- **골격이 못 감추는 운영 제약** — imessage FDA, signal 외부 데몬, zalouser 역공학.

---

### 부록. 검증 메모
- 핀 유효성: `0fc5a57a..HEAD`(=`7e06dd8b`)는 study 문서만 변경 → `extensions`/`src`/`packages` byte-동일(메인 세션 검증).
- 비평 정정 반영: **qqbot 골격 함수 미경유**(grep 무결과 확증) / telegram 버튼 checksum은 **opaque value 한정** / whatsapp degrade는 **이중 조건 게이트** / audio-decode는 Baileys용 **간접 peer** / Signal auto 모드 native 50ms grace / Discord deps에 discord.js 부재(voice·types만) — 전부 코드 교차 확증.
