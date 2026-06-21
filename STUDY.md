# OpenClaw 구조 공부 가이드

> 개인 학습용 메모. OpenClaw `2026.6.2` 기준.
> Fork: https://github.com/freeftr/openclaw · Upstream: https://github.com/openclaw/openclaw

## 0. 이 레포가 뭔지 (TL;DR)

- **개인 AI 어시스턴트** — 내 기기에서 돌리고, 내가 이미 쓰는 메신저(Telegram/WhatsApp/Discord/Slack…)로 응답.
- **pnpm 워크스페이스 모노레포** (TypeScript, MIT).
- 4개 영역으로 나뉨:

| 영역 | 경로 | 역할 |
|------|------|------|
| 코어 런타임 | `src/` | 에이전트 루프 · 채널 · LLM · 게이트웨이 본체 (76개 하위 디렉터리) |
| 공유 패키지 | `packages/*` | 코어/플러그인 공용 라이브러리 (21개) |
| 플러그인 | `extensions/*` | 채널·기능 통합 (143개). 내부명 "extensions", 제품명 "plugins" |
| 네이티브 앱 | `apps/*` | android · ios · macos 클라이언트 |
| UI | `ui/` | 웹 인터페이스 / Canvas |

## 1. 두 가지 설계 원칙 (먼저 머리에 박을 것)

이 두 개만 알면 코드 읽기 난이도가 확 떨어진다.

1. **코어는 플러그인을 모른다 (plugin-agnostic)**
   - 플러그인 ID/기본값이 `src/`에 하드코딩되지 않음.
   - 플러그인은 오직 `src/plugin-sdk/*` 계약 + manifest 메타데이터로만 코어에 진입.
2. **상태는 전부 SQLite**
   - 전역 상태: `state/openclaw.sqlite`
   - 에이전트별 상태: `agents/<agentId>/agent/openclaw-agent.sqlite`
   - JSON/JSONL 사이드카 파일 안 씀. 접근은 Kysely 헬퍼 사용 (raw SQL 아님).

## 2. 데이터 흐름 (큰 그림)

```
네이티브 앱 / 채널(extensions)
   → 게이트웨이 (src/gateway, packages/gateway-protocol)
   → 에이전트 런타임 (src/agents)
   → LLM (src/llm) + 툴 (src/tools) + 메모리 (src/memory, SQLite)
   → 응답이 같은 채널로 역방향
```

## 3. 공부 순서 (추천 루트)

### Step 1 — 전체 지도 잡기
- [ ] `AGENTS.md` 통독 ← **가장 먼저**. 아키텍처 철학·경계 규칙이 전부 여기 있음.
- [ ] `README.md` 훑기 (기능·설치 개요)
- [ ] `VISION.md` (프로젝트가 지향하는 방향)
- [ ] `pnpm-workspace.yaml` + 루트 `package.json` (워크스페이스 패키지 경계)

### Step 2 — 진입점 & 부트스트랩
- [ ] `openclaw.mjs` (CLI bin 진입점)
- [ ] `src/entry.ts` → `src/bootstrap/`
- [ ] `src/cli/`, `src/commands/` (명령어 구조)
- [ ] `src/daemon/` (Gateway 데몬 동작)

### Step 3 — 에이전트 코어 (심장부)
- [ ] `src/agents/` (에이전트 메시지 루프)
- [ ] `packages/agent-core/` (공유 에이전트 로직)
- [ ] `src/context-engine/` (컨텍스트 관리)
- [ ] `src/sessions/`, `src/state/` (세션/상태)

### Step 4 — LLM 레이어
- [ ] `src/llm/` (LLM 호출 흐름)
- [ ] `packages/llm-core/`, `packages/llm-runtime/`
- [ ] `src/model-catalog/` (모델 카탈로그/라우팅)
- [ ] `src/tools/` (툴 호출)

### Step 5 — 채널 / 플러그인 (확장 메커니즘)
- [ ] `src/channels/` (채널 공통 인터페이스)
- [ ] `src/plugin-sdk/` (플러그인 ↔ 코어 계약)
- [ ] `src/plugins/` (플러그인 로더)
- [ ] `extensions/telegram/src/index.ts` ← 실제 어댑터 1개를 끝까지 따라가 보기

### Step 6 — 게이트웨이 / 프로토콜
- [ ] `src/gateway/`
- [ ] `packages/gateway-protocol/` (클라이언트↔게이트웨이 통신)
- [ ] `packages/gateway-client/`

### Step 7 — 메모리 / 저장소
- [ ] `src/memory/`, `src/memory-host-sdk/`
- [ ] SQLite 스키마·마이그레이션 (`openclaw doctor --fix` 흐름)

### Step 8 (선택) — 주변부
- [ ] `apps/` (네이티브 클라이언트, 관심 있으면)
- [ ] `ui/` (웹/Canvas)
- [ ] `src/mcp/` (MCP 연동)
- [ ] `docs/` (`pnpm docs:list` 로 목록 확인)

## 4. 읽을 때 팁

- 각 서브트리에 **스코프별 `AGENTS.md`** 가 또 있다. 해당 디렉터리 작업 전 그것부터 읽으면 맥락이 잡힌다.
  (예: `extensions/`, `src/{plugin-sdk,channels,plugins,gateway,agents}/`, `packages/`, `docs/`)
- 파일 참조는 repo-root 기준으로: `extensions/telegram/src/index.ts:80`
- 테스트(`*.test.ts`)가 동작 명세 역할을 한다. 헷갈리면 옆 테스트를 읽으면 의도가 보임.
- 의존성 동작이 궁금하면 추측 말고 OSS 소스를 직접 확인 (대부분 OSS).

## 5. 로컬 셋업 (코드만 읽을 거면 생략 가능)

```bash
# 의존성 설치 (모노레포라 수백 MB~1GB+ 추가됨)
pnpm install

# 문서 목록
pnpm docs:list
```

## 6. Git remote 메모

```
origin    → git@github.com:freeftr/openclaw.git   (내 포크)
upstream  → git@github.com:openclaw/openclaw.git   (원본)

# 원본 최신 따라잡기
git fetch upstream && git merge upstream/main
```

> 현재 `--depth 1` shallow clone 상태라 히스토리가 없다.
> 전체 히스토리가 필요하면: `git fetch --unshallow upstream`
