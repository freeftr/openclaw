# OpenClaw 채널 + 플러그인 시스템 — 학습/분석

> ultracode 멀티에이전트로 채널·플러그인 7영역을 병렬 매핑 → 정확성 비평 → 종합. 설명 + 각 절 설계 평가.
> 코드 `file:line`은 **`openclaw/openclaw@0fc5a57a`** 기준 (study 커밋들은 `src`/`packages`/`extensions`를 변경하지 않아 working tree == 이 SHA로 검증됨). 핵심 파일만 링크, 나머지는 코드 스팬.
> 독자 전제: 게이트웨이·에이전트는 안다([gateway-study.md](./gateway-study.md)·[agent-runtime-study.md](./agent-runtime-study.md)). 이 문서는 **"채널 = 게이트웨이 안 in-process 플러그인"이 *어떻게* 코드로 실현되는가**에 집중한다.

---

## 0. 큰 그림 — 채널과 플러그인 시스템의 관계

### 두 평면, 하나의 진입 객체
- **플러그인 시스템(control + runtime plane)**: 디스커버리 → 매니페스트 → 로더 → 활성 레지스트리. 채널은 이 시스템이 로드/활성화하는 **여러 플러그인 종류 중 하나**.
- **채널 = 그 시스템의 한 소비자**: 코어는 채널 id/기본값을 하드코딩하지 않고, 플러그인이 `registerChannel`로 등록한 `ChannelPlugin` 객체의 슬롯(`gateway.startAccount`·`outbound`·`config` 등)만 호출.
- **유일 진입 객체 `OpenClawPluginApi`**: host가 record별로 조립해 주입(`src/plugins/registry.ts:2764-2839`). 플러그인은 `src/**`를 직접 import하지 않고 이 객체 + `openclaw/plugin-sdk/*` 배럴 + 주입된 `PluginRuntimeChannel`로만 코어에 닿는다.

### "in-process plugin"의 코드적 의미
별도 프로세스가 아니라, 게이트웨이 프로세스 안에서 `plugin.gateway.startAccount(ctx)` 호출로 transport가 산다([`src/gateway/server-channels.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-channels.ts)`:428-438`). 게이트웨이가 활성 레지스트리의 채널 surface를 **pin**해 살아있는 런타임 핸들로 유지(`src/plugins/runtime.ts:25-53,283-345`).

> **[설계 평가]** "플러그인 시스템 → 채널"의 단방향 의존이라 채널 종류가 늘어도 코어 코드가 안 바뀐다(강점). 다만 활성 레지스트리가 `globalThis` 가변 싱글톤(`runtime.ts:25-53`)이라, `src/plugins/CLAUDE.md`가 스스로 "호환 scaffolding으로 취급, request-scoped 핸들 선호"라 경고하는 약한 고리.

---

## 1. 한 메시지의 채널 왕복 (end-to-end)

이후 절의 지도. 각 단계 = 해당 절 번호.

```
1. 플랫폼 이벤트 수신 (플러그인 transport)                              → §2/§8
2. inbound 진입: buildChannelInboundEventContext                       → §3
   (plugin-sdk/channel-inbound.ts:81-205, 플러그인은 이 배럴로만 진입)
3. turn kernel: ingest→classify→preflight→resolveTurn→dispatch         → §3/§5
   (src/channels/turn/kernel.ts:619-783)
4. 라우팅·세션키: route 결정은 resolve-route.ts:71-118, inbound는 운반만 → §3
5. durable ingress 큐 중복제거 (message/ingress-queue.ts:319-389)       → §3
6. 에이전트 dispatch                                                    → (기존 문서)
7. outbound durable delivery: write-ahead 큐 (infra/outbound/deliver.ts) → §4
8. presentation 한계 적응 (presentation-limits.ts:469-567)              → §4
9. native envelope 매핑 (예: Telegram tgcmd:/tgcb1:)                    → §4/§8
```

> **[설계 평가]** "플러그인=transport+facts, 코어=정책+정규화"가 왕복 전 구간에서 일관 — 신뢰·정책 판단(분류/세션키/명령권한/presentation 적응)이 코어 한곳에 모여 채널 간 동작 일관성이 구조적으로 보장된다. 거친 부분: 양 끝(inbound `context.ts`, outbound `payload.ts`)에 `@deprecated` 호환 표면이 남아 canonical 경로 식별이 어렵다(마이그레이션 중 이중 표면).

---

## 2. 채널 런타임 · 계정(account) 관리

**무엇/왜:** 게이트웨이 채널 매니저가 in-process 채널의 계정 단위 라이프사이클(start/stop/restart)을 격리 스토어로 구동. 멀티 계정 해석은 **플러그인 config가 소유**.

**어떻게** ([`server-channels.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-channels.ts)):
- **도킹 = gateway 훅만 호출**: `createChannelManager`는 채널을 모르고 `plugin.gateway.startAccount/stopAccount`만 호출, 없으면 즉시 return(`:433-435`). "채널=in-process 플러그인" 전제의 코어측 구현.
- **계정별 격리 스토어**: `ChannelRuntimeStore`(aborts/starting/tasks/runtimes 4 Map, accountId 키, `:48-53,343-351`). **AbortController를 첫 await 전에 예약**(`:483-486`)해 중복 부팅 레이스 차단. 계정 병렬 기동 `CHANNEL_STARTUP_CONCURRENCY=4`.
- **멀티 계정은 플러그인 소유**: 목록 `plugin.config.listAccountIds`(`:444-446`), 각 계정 `resolveAccount`(`:513`). id 후보 `account-helpers.ts:120-146`, 정규화 `routing/account-id.ts:38-50`, 조회 `account-lookup.ts:6-39`.
- **게이팅 + 승인 부트스트랩**: isEnabled→isConfigured 미충족 시 사유 박고 종료(`:514-545`), 통과 시 task-scoped 런타임 + 네이티브 승인 핸들러(`:559-580`).
- **자동 재시작 백오프 상태기계**: `:639-732`(initial 5s/max 5m/factor 2/jitter 0.1, MAX_RESTART_ATTEMPTS=10), `recoveryStopTimedOut`/`recoveryStartRequested` Set으로 "stop 타임아웃 후 늦게 끝남" 분기(`:656-693`).
- **stop = abort 후 bounded graceful 대기**(`CHANNEL_STOP_ABORT_TIMEOUT_MS=5s`, `:774-834`). manual 타임아웃 시 `running:true` 유지(아직 안 멈춤 표시, `:843-848`).
- **헬스모니터 폴백**: 계정>채널>플러그인, resolver는 프로브용·fail-closed(`:284-341`).

> **[설계 평가]** 모든 라이프사이클 상태가 `channel:account` 키로 격리돼 멀티 계정 무간섭 + AbortController 사전 예약으로 레이스 구조 차단(강점). 매니저가 채널 식별자/기본값을 전혀 하드코딩 않고 `plugin.config`에 위임해 plugin-agnostic 일관(강점). 거친 부분: stop 타임아웃 복구가 두 Set + 여러 then 분기로 분산돼 "closed 모드/결과 셰이프" 선호보다 nullable 플래그 조합에 가까움.

---

## 3. inbound — 외부 메시지가 내부 표현이 되는 길

**무엇/왜:** 플러그인은 facts만, 코어가 분류·정규화·세션키·중복제거·신뢰경계를 소유.

**어떻게:**
- **SDK 배럴 진입**: 플러그인은 `src/channels` 직접 import 안 하고 [`src/plugin-sdk/channel-inbound.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/plugin-sdk/channel-inbound.ts)`:81-205`로만 진입(telegram/discord/slack/signal/whatsapp/matrix/qqbot/imessage 등).
- **facts → FinalizedMsgContext**: `inbound-event/context.ts:457-534`. SessionKey는 `route.dispatchSessionKey ?? route.routeSessionKey`(`:485`) — inbound는 운반, 생성은 routing 책임.
- **room_event 분류(에이전트 깨움 게이트)**: `classification.ts:26-62`. 멘션/native command/control/abort는 항상 user_request.
- **미디어 정규화**: `media.ts:55-114`, 경로 화이트리스트 `packages/media-core/src/inbound-path-policy.ts:85-110`.
- **durable ingress 큐**: `message/ingress-queue.ts:319-389`, `(queue_name, event_id)` onConflict로 멱등(queue_name=`JSON.stringify([channelId,accountId])`).
- **세션키 생성**: `session-key.ts:212-347`(dmScope 분기, identityLinks 병합, `:thread:` 접미사), route 결정 `resolve-route.ts:71-118`.
- **명령/신뢰 경계**: native/text-slash/normal 닫힌 유니온 `command-turn-context.ts:96-227`(normal은 `authorized=false` 강제), 슬래시 파싱 `commands-slash-parse.ts:16-62`.

> **[설계 평가]** inbound 경계가 "플러그인=transport+facts / 코어=정책+정규화"로 깔끔히 갈리고 신뢰 판단이 전부 코어(강점). spoofed system marker를 `GroupSystemPrompt`가 아닌 `UntrustedStructuredContext`로 격리(`context.ts:417-428`)하는 등 신뢰경계가 명시적. 거친 부분: `context.ts`에 `@deprecated` 호환 타입/오버로드 다수 — SDK 표면을 넓힘.

---

## 4. outbound · presentation — durable delivery / 한계 적응 / typed action

**무엇/왜:** write-ahead 큐로 durable 전송, portable presentation을 채널 limit에 맞춰 코어가 적응, 채널은 native envelope 매핑만. 토큰-델타 스트리밍 없음.

**어떻게:**
- **write-ahead 큐**: enqueue→send→ack/fail, "persist before sending, remove after success"([`infra/outbound/deliver.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/infra/outbound/deliver.ts)`:1257-1399`). 단일 소유 클레임 `withActiveDeliveryClaim`(`:1300-1308`), unknown-send 화해 `markQueuedPlatformOutcomeUnknown`(`:1365-1370`).
- **durability 결정**: `reconcileUnknownSend===true ? "required" : "best_effort"` 단일조건(`durable-delivery.ts:161-162`). capability 게이트 `capabilities.ts:34-62` → `deliver.ts:305-335`.
- **presentation 채널 limit 적응**: `presentation-limits.ts:469-567`. 버튼 capacity는 셀렉트 슬롯 선예약 후 계산(`:398-448`), byte 초과 시 버튼 아닌 그 필드만 제거(`:205-235`). truncate는 code-point/utf8/utf16 단위(`:38-74`).
- **typed action → native envelope**: command|callback 유니온 `src/interactive/payload.ts:16-99`. Telegram: url→url버튼, command→`tgcmd:`, callback→`tgcb1:`+checksum, web_app→web_app(`extensions/telegram/src/button-types.ts:38-83`). prefix로 transport-private 불변식 보장.
- **토큰-델타 금지**: `docs/concepts/streaming.md:12-33`. preview는 send+edit, progress draft 라인 merge(`streaming.ts:1015-1108`).
- **partial-send 중복 억제**: handled_visible/handled_no_send/failed 닫힌 union(`durable-delivery.ts:122-132,216-222`).

> **[설계 평가]** 닫힌 result union으로 호출자가 모든 상태 처리를 강제받고, partial-send visible 마킹으로 중복 응답을 구조 차단(강점). presentation 적응이 코어 1곳 집중 → "채널=transport-only" 경계 준수. 트레이드오프: durability를 `reconcileUnknownSend` 구현 여부 단일조건으로만 가르므로 채널이 그 함수 미구현 시 조용히 best_effort 강등 → unknown-send 중복 위험이 채널 성실도에 의존.

---

## 5. 채널 플러그인 계약 (src/channels/plugins/*)

**무엇/왜:** ~40개 optional 어댑터를 합성한 단일 `ChannelPlugin` 객체. 코어는 loaded>bundled로 해석, 슬롯 유무로 분기.

**어떻게:**
- **단일 객체 + optional 어댑터**: `types.plugin.ts:66-111`(필수 id/meta/capabilities/config), 어댑터 계약 `types.adapters.ts:121-160,342-360`.
- **loaded>bundled 해석**: `getChannelPlugin = getLoadedChannelPlugin(...) ?? getBundledChannelPlugin(...)`([`channels/plugins/registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/channels/plugins/registry.ts)`:49-57`) — 설치 플러그인이 번들 override/pin.
- **수신 = turn kernel**(`turn/kernel.ts:619-783`), **송신 = outbound adapter + presentation 한계 선언**(`outbound.types.ts:49-235`).
- **message-action discovery**: `actions.describeMessageTool`(`types.core.ts:725-770`), cross-channel 스코프 숨김(`message-action-discovery.ts:271-313`), 경량 아티팩트 우선.
- **transport-only 명령 경계**: typed presentation action(`interactive/payload.ts:17-89`) — 채널이 value의 `/` 시작 여부 추론 불필요.
- **바인딩 = 코어 소유, 플러그인 관찰**: `binding-routing.ts:83-127`.

> **[설계 평가]** 거대한 optional-어댑터 합성이라 채널은 지원 능력만 채우면 되고, transport-only가 typed action으로 코드 레벨 강제(강점). 거친 부분: ~40 어댑터 + `types.core.ts` 800줄 + deprecated 별칭 누적으로 신규 채널 작성자가 canonical 경로 식별이 어려움(AGENTS도 "SDK surface too large" 명시). `ChannelPlugin<ResolvedAccount=any>`의 `any`는 contravariance 회피용 의도된 타협이나 타입 안전성 구멍.

---

## 6. 플러그인 로딩 메커니즘 (src/plugins/*)

**무엇/왜:** 매니페스트-우선 control-plane(디스커버리→매니페스트 레지스트리→installed-index)이 후보를 모으고, cacheKey 캐시로 레지스트리를 조립, capability/facade는 활성 우선 + lazy fallback.

**어떻게:**
- **control plane**: 디스커버리 `discovery.ts:1429-1628`, 정규화·중복해소 [`manifest-registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/plugins/manifest-registry.ts)`:953-1208`, 인덱스 `installed-plugin-index.ts:49-153`.
- **bundled vs external + 두 rank 함수**: origin은 bundled/global/workspace/config 4종(`plugin-origin.types.ts:2`, **`external` 값 없음** — 설치형=global, `plugins.load.paths`=config).
  - `PLUGIN_ORIGIN_RANK`(`manifest-registry.ts:192`, **4단계**): **동일 on-disk root 정규화용**.
  - `resolveDuplicatePrecedenceRank`(`:864-902`, **6단계** config<dev-bundled<installed-global<bundled<workspace<global): **중복 plugin id 해소용**(설치형 global이 bundled를 이김).
- **cacheKey 캐시**: `loader.ts:923-1001`(buildCacheKey), 호환 재사용 `getCompatibleActivePluginRegistry:1403-1524`, LRU+재진입 `loader-cache-state.ts:5-74`.
- **활성 레지스트리 싱글톤**: `runtime.ts:25-206,359-408`(active/channel/http-route surface + pin/release + retire/cleanup).
- **capability 3단 해소**: 활성→매니페스트 cold load→bundled compat(`capability-provider-runtime.ts:473-561,587-680`).
- **facade 런타임**: `loadBundledPluginPublicSurfaceModuleSync`는 **`src/plugin-sdk/facade-runtime.ts:74-101,194-256`**에 있음(`src/plugins` 아님). bundled 우선, 실패 시 registry 폴백.

> **[설계 평가]** control-plane / runtime-plane 분리가 중심이고 capability 3단 fallback이 laziness 정책의 직접 구현(트레이드오프 일관). bundled/external이 동일 `api.js`/`runtime-api.js` facade를 공유해 backdoor 없음(강점). 거친 부분: `getCompatibleActivePluginRegistry`가 옵션 조합마다 cacheKey 일치를 반복 시도하는 분기 피라미드라 조용한 캐시 미스/오재사용 위험. 약한 고리: 활성 레지스트리 `globalThis` 가변 싱글톤.

---

## 7. plugin-agnostic 경계 강제

**무엇/왜:** 플러그인은 host 조립 `OpenClawPluginApi`로만 진입, import 방향은 스캐너+package-local tsc+canary로, 게이트웨이 메서드 스코프는 정규화 헬퍼로 강제.

**어떻게:**
- **OpenClawPluginApi 조립**: `registry.ts:2745-2839`(buildPluginApi, runtime 헬퍼 record 바인딩). `registrationMode` 게이팅(full/discovery/tool-discovery/setup-only/…)으로 광범위 쓰기 제한(`:382-391,2778-2825`).
- **채널 = in-process(registerChannel)**: `registry.ts:931-975`(runtimeChannel off면 갱신 스킵).
- **import 방향 강제(다층)**: extension→코어 스캐너 `scripts/check-extension-plugin-sdk-boundary.mjs:47-54,151-227`(3모드 baseline diff), 코어→extension `check-src-extension-import-boundary.mjs:7-27`(예외 `api.js`/`runtime-api.js`). 타입 레벨 `extensions/tsconfig.package-boundary.paths.json`(deep 코어 resolve 불가) + **canary는 컴파일 *실패* 기대**(`check-extension-package-tsc-boundary.mjs`).
- **게이트웨이 메서드 스코프**: `gateway-method-policy.ts:2-45`(`config.`/`exec.approvals.`/`wizard.`/`update.` → `operator.admin` coerce) — [§gateway-study](./gateway-study.md) 의 그 정책.
- **SDK 진입 단일 출처 + 패키지 재노출**: `entrypoints.ts:1-67`, `packages/plugin-sdk/package.json`이 코어 `src/plugin-sdk/*.ts`를 재노출하는 얇은 facade.

> **[설계 평가]** 다층 방어(런타임 게이팅 + 양방향 소스 스캐너 + 타입 레벨 + canary 역검증). **canary가 "경계 작동"을 적극 증명**하는 드문 패턴(강점). 트레이드오프: 스캐너가 baseline diff라 기존 위반을 fixture로 동결 → baseline 자체가 리뷰 부채. 약점: 경계 단일 출처가 4곳(entrypoints + tsconfig paths + package.json exports + 스캐너)에 분산돼 동기화가 사람/체크 의존이고, 실제로 `@openclaw/plugin-sdk` exports의 일부 default 경로(`./account-id`→`./src/account-id.ts`)가 **부재 파일을 가리키는 정합 빈틈**이 존재(검증됨).

---

## 8. 실제 채널 비교 — telegram / discord / whatsapp / slack

**무엇/왜:** 네 채널 모두 `createChatChannelPlugin` 단일 골격 + `base.gateway.startAccount` 단일 격리점. 플랫폼 차이만 슬롯 콜백으로 흡수.

**어떻게:**
- **공통 골격**: `createChatChannelPlugin`(`src/plugin-sdk/core.ts:804-828`). 번들 진입 `defineBundledChannelEntry`. Discord/Slack만 `registerFull`(discord subagent hooks+transcript provider, slack http routes), telegram/whatsapp는 없음.
- **transport 4종(전부 outbound)**:
  - **Telegram** = grammY 롱폴(`extensions/telegram/.../monitor.ts:111-317`), webhookUrl 있으면 webhook 전환.
  - **Discord** = **자체 ws 게이트웨이**(discord.js 미사용, `internal/`+discord-api-types). 단 deps에 `@discordjs/voice` 존재(voice transcript용).
  - **WhatsApp** = Baileys `makeWASocket`(QR 페어링).
  - **Slack** = Bolt 동적 import(Socket Mode/HTTP).
- **인증 모델**: Telegram/Discord/Slack=봇 토큰(`channelEnvVars` 선언), WhatsApp=QR(매니페스트엔 `messageReceived` 훅 옵트인만, `channelEnvVars` 없음).
- **네이티브 명령**: Telegram setMyCommands, Discord 앱 슬래시 배포, Slack Bolt 미들웨어, WhatsApp 텍스트 command-policy(슬래시 없음).

> **[설계 평가]** 극단적으로 다른 4개 transport를 단일 골격 + 단일 격리점으로 통일하고 차이를 슬롯 콜백에만 가둠(강점). 트레이드오프: Discord는 discord.js 대신 게이트웨이/REST/커맨드 배포를 자체 구현 → 정밀 제어 대가로 유지보수 표면 최대. 거친 부분: 플랫폼 비대칭이 매니페스트·슬롯에 누수 → "동형"은 형태일 뿐 동작 동등성은 아님. `getOptionalRuntime()?.channel?.x?.messageActions ?? impl` override 패턴이 3채널에 동형 복제 → SDK 헬퍼 추출 여지.

---

## 9. 종합 평가 — plugin-agnostic 경계는 실제로 얼마나 지켜지나

### 잘 지켜지는 곳 (강점)
- **채널 식별/정책의 코어 하드코딩 부재** — 매니저(§2)·registry(§5)·로더(§6) 전부 `plugin.config`/매니페스트 위임.
- **transport-only 경계의 코드 레벨 강제** — typed presentation action(§4/§5)으로 채널이 제품 명령 문자열을 추론하지 않음.
- **다층 경계 방어 + canary 역검증**(§7) — "경계 작동"을 적극 증명.
- **단일 골격 4채널**(§8) — 새 채널 추가 시 채울 구멍이 명확.

### 위험 · 거친 부분
- **globalThis 가변 싱글톤 활성 레지스트리**(§0/§6) — CLAUDE.md가 스스로 scaffolding 경고.
- **넓은 SDK 표면 + deprecated 이중 표면**(§3/§4/§5) — canonical 경로 식별 비용, 마이그레이션 미완.
- **단일조건 durability 강등**(§4) — 채널 성실도 의존.
- **경계 단일 출처 4곳 분산 + exports↔실파일 어긋남**(§7) — `./account-id` default가 부재 파일을 가리키는 정합 빈틈.
- **상태기계 분산**(§2) — stop 복구가 Set+then 분기.

### 한 줄 결론
> "채널 = in-process 플러그인"은 **(a) 게이트웨이가 `plugin.gateway.startAccount`만 호출하고 (b) 플러그인이 host 조립 `OpenClawPluginApi`로만 진입하며 (c) 경계가 4층(런타임 게이팅·양방향 스캐너·타입·canary)으로 강제됨**으로 실현된다. 핵심 invariant는 견고하나, 비용은 넓은 SDK 표면·전환기 deprecated 표면·globalThis 싱글톤·단일출처 분산에 누적돼 있다.

---

### 부록. 검증 메모
- **핀 유효성**: `0fc5a57a..HEAD`는 `STUDY.md`+`openclaw-study/`만 변경 → `src`/`packages`/`extensions` byte-동일, 라인참조 유효(검증).
- **정정 반영**: `PLUGIN_ORIGIN_RANK`(4단계)와 `resolveDuplicatePrecedenceRank`(6단계)는 별개 함수(§6) / facade-runtime은 `src/plugin-sdk`에 위치(§6) / origin에 "external" 없음(설치형=global) / whatsapp `persistedAuthState`는 코드 측 개념(매니페스트 아님, §8) / discord `@discordjs/voice` deps 존재(§8) / `@openclaw/plugin-sdk` exports `./account-id`가 부재 파일 지목(§7).
