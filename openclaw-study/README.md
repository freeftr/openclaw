# OpenClaw 스터디 — 인덱스 & 로드맵

> OpenClaw(`openclaw/openclaw`) 아키텍처를 서브시스템 단위로 파는 개인 학습 시리즈.
> 방법론: ultracode 멀티에이전트(영역 병렬 매핑 → **정확성 비평**(표본 spot-check) → 종합) + 코드 링크는 `@0fc5a57a` SHA 핀.

## 전체 지도 — 어디까지 왔나

```
                         ┌─────────────────────────────────────────┐
   [운영 축]             │              게이트웨이 ✅                │        [클라이언트]
   CLI·온보딩·config     │   (WS 프로토콜·인가·라우팅·broadcast)      │        앱/UI ⬜
   doctor·state ⬜       └───────▲──────────────▲───────────────────┘
                                 │              │
                     채널+플러그인 ✅      에이전트 런타임 ✅ ── 심화 ✅
                     (시스템+구현 2편)          │
                                        ★ agent-core 루프 ✅ ★
                                                │
                              메모리 ✅ ── 컨텍스트 엔진·압축 (부분)
```

## 완료한 문서

| # | 문서 | 범위 | 성격 |
|---|---|---|---|
| 1 | [openclaw-architecture.md](./openclaw-architecture.md) | 전체 구조 개관 | 학습 |
| 2 | [gateway-study.md](./gateway-study.md) | 게이트웨이 — 목표→구조→왜 WS·인가·라우팅 | 학습 서사 |
| 3 | [openclaw-gateway.md](./openclaw-gateway.md) | 게이트웨이 코드 레퍼런스 (14섹션) | 레퍼런스 |
| 4 | [gateway-qa.md](./gateway-qa.md) | 게이트웨이 Q&A (method 문자열·저장 위치 등) | FAQ |
| 5 | [agent-runtime-study.md](./agent-runtime-study.md) | 에이전트 런타임 — 2중 루프·하니스·툴·모델·세션 | 학습 |
| 6 | [agent-deep-analysis.md](./agent-deep-analysis.md) | agent 심화 — 동시성 lane·steering·승인·cooldown·transport·CLI백엔드·서브에이전트 | **분석**(설계 평가) |
| 7 | [channels-plugins-study.md](./channels-plugins-study.md) | 채널+플러그인 시스템 — 런타임·in/outbound·SDK계약·로더·경계 | 학습+분석 |
| 8 | [channel-plugins-deep.md](./channel-plugins-deep.md) | 채널 구현 심층 — TG/Discord/WA/Slack·Signal 내부, 스펙트럼 19종, doctor·미디어 | 분석 |
| 9 | [agent-core-study.md](./agent-core-study.md) | 내장 에이전트 루프 심장부 — runLoop 이중 while·prompt() API·세션 JSONL DAG·시스템 프롬프트·툴 생애·이벤트 투영·compaction 계약 | 학습+분석 |
| 10 | [main-agent-study.md](./main-agent-study.md) | main 에이전트의 하루 — 부트스트랩 8종·SOUL/IDENTITY·BOOT.md·auto-reply·heartbeat·메모리 유지보수·메인 세션 수명 | 학습+분석 |
| 11 | [nodes-study.md](./nodes-study.md) | 노드 시스템 — node-host·2겹 페어링·invoke 카탈로그·기기 능력·exec 포워딩·게이트웨이측·네이티브 앱 | 학습+분석 |

다이어그램: [diagrams/](./diagrams) — 01 아키텍처 · 02 에이전트 루프 · 03 플러그인 생명주기 · 04 게이트웨이 RPC · 05 end-to-end · 06 agent run 2중 루프 · 07 노드 설계⇄정설 교차검증

## 남은 주제 (추천 순)

| 순위 | 주제 | 범위 | 왜 |
|---|---|---|---|
| 1 | **운영 축: CLI·온보딩·config·doctor·state** | `src/cli`·`commands`·`wizard`·`config`·`state` | "깔면 어떻게 뜨고 설정되나". doctor 마이그레이션(canonical config + `doctor --fix`)이 이 레포의 가장 독특한 설계 철학 — 루트 AGENTS.md 절반의 실체 |
| 2 | **세션·컨텍스트 엔진 심화** | `src/context-engine`·`sessions`·compaction·transcript DAG | 메모리 스터디의 옆 조각. "compaction=DAG 마커" 등 비직관 다수 (agent-core 편과 일부 겹침 — 그 결과 보고 범위 조정) |
| 4 | **보안 축** | `secrets`·`security`·exec approvals·sandbox·credentials | 위협 모델이 명시적인 코드(승인 2-RPC·5분 TTL handoff·CWE-841 차단). 보안 관점 훈련 |
| 5 | **cron·tasks·flows·heartbeat** | `src/cron`·`tasks`·`flows` | "에이전트가 스스로 깨어나는" 축. lane 스케줄링(6번 문서)과 연결 |
| 6 | **모델 카탈로그·프로바이더 런타임** | `model-catalog`·`provider-runtime`·auth-profiles | 수십 provider의 단일 카탈로그 정규화. cooldown/OAuth(6번 문서)의 상류 |
| 7 | **미디어·음성 스택** | `talk`·`tts`·`realtime-transcription`·`*-generation`·`media-understanding` | 멀티모달 파이프라인. Discord realtime voice 같은 독립 서브시스템 |
| 8 | **skills·MCP** | `src/skills`·`mcp` | 에이전트 능력 확장의 또 다른 축(플러그인과 다른 결) |
| 9 | **앱/UI** | `apps/{macos,ios,android}`·`ui/` | Swift/Kotlin 클라이언트가 WS 프로토콜을 소비하는 "반대편" |
| 10 | **빌드·모노레포** | pnpm workspace·bundled 번들링·dist·scripts | bundled vs external 플러그인이 빌드에서 갈리는 지점(7번 문서의 하류) |

## 방법론 메모

1. **영역 분해** — 주제를 5~7개 독립 영역으로 쪼개 Explore 에이전트 병렬 매핑 (구조화 스키마: keyFiles·mechanisms·designTakes·gotchas)
2. **정확성 비평** — 별도 에이전트가 file:line 표본 검증, 틀린 주장·누락·모순 지적 (매 회차 실질 정정이 나옴: qqbot 골격 미경유, HARNESS_PLUGIN_IDS 부재 등)
3. **종합** — 학습 서사(무엇·왜·어떻게) + 설계 평가(강점·트레이드오프·거친 부분)로 작성
4. **핀 검증** — `git diff 0fc5a57a..HEAD -- src packages extensions`가 비어 있는지 확인 후 SHA 핀 링크 사용
