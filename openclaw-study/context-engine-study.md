# OpenClaw 컨텍스트/세션 엔진 심화 — compaction은 삭제가 아니라 DAG 재배선이다

> 이 문서는 `agent-core-study.md`의 위층이다. runLoop 이중 while, PendingMessageQueue, `prompt()` 표면, compaction 상수의 단순 나열, "message_end=커밋" 같은 이미 다룬 사실은 재설명하지 않는다. 여기서 증명할 단 하나의 논지는 이것이다.

**관통 논지 — 트랜스크립트는 append-only DAG이고, compaction은 노드를 지우는 게 아니라 마커 노드를 심어 경로를 우회시킨다.** 저장 계층은 절대 기존 줄을 수정·삭제하지 않는다. "오래된 대화가 사라졌다"는 착시는 전부 *조립(assemble) 시점*의 계산이다. 이 문서의 모든 섹션은 이 명제의 서로 다른 각도의 증명이다.

---

## 0. 재정의 — "히스토리 관리"라는 말이 숨기는 것

대부분의 에이전트 문서는 컨텍스트를 "메시지 배열을 자르고 요약하는 파이프라인"으로 그린다. OpenClaw의 harness(`@openclaw/agent-core`)는 그렇지 않다. 세 가지를 분리해서 봐야 한다.

1. **저장 진실(storage truth)** — 디스크에 있는 것. append-only JSONL 트리. 한 번 쓰면 불변.
2. **활성 경로(active path)** — 트리에서 지금 보이는 잎(leaf)부터 루트까지 `parentId`를 거슬러 올라간 *하나의 선형 경로*.
3. **모델 뷰(model view)** — 그 활성 경로를 `buildSessionContext`가 접어 만든 실제 LLM 메시지 배열.

compaction·branch 요약·fork·되감기는 전부 **(1)을 건드리지 않고** (2)의 모양 또는 (3)의 접기 규칙만 바꾼다. 이 세 층의 분리가 이 엔진의 전부다. 아래는 그 기계장치다.

> **[설계 평가]** 강점: "편집 없는 편집". 라벨 변경조차 파일 수정이 아니라 새 `label` 엔트리 append 후 리플레이로 최종 상태를 계산한다([`storage-base.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/storage-base.ts)`:106-113`). 저장이 불변이라 fork·감사·프롬프트 캐시 결정성이 공짜로 따라온다. 트레이드오프: 무엇 하나 지우지 않으므로 디스크는 단조 증가한다 — 토큰은 줄어도 파일은 절대 안 줄어든다.

---

## 1. 세션 DAG의 실제 구조 — `parentId` 하나 + `leaf` 마커

세션 노드는 정확히 **10종**의 union이다([`types.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/types.ts)`:454-464`): `message` | `thinking_level_change` | `model_change` | `compaction` | `branch_summary` | `custom` | `custom_message` | `label` | `session_info` | `leaf`. 이 중 실제 LLM 대화가 되는 건 `message`뿐이고, 나머지는 상태 마커·우회 마커·표시 메타다.

부모/자식 링크는 단 하나의 필드로 표현된다 — `SessionTreeEntryBase.parentId: string | null`(`null`=루트). 모든 append는 `parentId = 현재 leafId`로 새 노드를 매단다. 그런데도 이게 선형이 아니라 **트리**인 이유는 `leaf` 엔트리 때문이다.

핵심 함수는 이것이다.

```ts
// storage-base.ts:42-44
export function leafIdAfterEntry(entry: SessionTreeEntry): string | null {
  return entry.type === "leaf" ? entry.targetId : entry.id;
}
```

**정정:** `leaf` 노드는 파일에 한 줄로 남지만, 그 자체가 `parentId` 체인의 부모 후보가 되지 않는다. `leafIdAfterEntry`가 `leaf`의 유효 잎을 `targetId`로 되돌리므로, `leaf` 다음 append의 `parentId`는 `leaf.id`가 아니라 `targetId`다. 즉 `moveTo`로 잎을 과거 노드로 되감으면, 이후 append들이 그 과거 노드의 자식으로 붙어 **같은 부모가 둘 이상의 자식을 갖는 분기**가 생긴다 — 이것이 DAG의 발생 지점이다.

활성 경로 재구성은 [`storage-base.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/storage-base.ts)`:131-152` `getPathToRoot`가 담당한다. `leafId`에서 시작해 `byId` 맵으로 노드를 찾고 `parentId`를 따라 `unshift`하며 루트까지 거슬러 올라간다 — O(경로길이). 중간에 dangling `parentId`를 만나면 `invalid_session` throw(`:147`). 저장은 전체 트리를 보관하지만, 컨텍스트 조립은 이 단일 경로 하나만 소비한다.

엔트리 id는 `uuidv7()`의 **앞 8자만** 쓴다([`storage-base.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/storage-base.ts)`:31-38`, 충돌 시 최대 100회 재시도 후 전체 uuidv7 fallback). `uuidv7`은 monotonic이라 같은 ms 호출은 sequence를 증가시켜 시간정렬·유일성을 보장한다([`uuid.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/uuid.ts)`:23-33`). 세션 id는 전체 uuidv7, 엔트리 id는 8자 — 길이·공간이 다르다.

> **[설계 평가]** 강점: `parentId` 하나 + `leaf` 마커라는 최소 원시요소로 선형 로그·트리 분기·되감기·fork를 전부 표현한다. `BaseSessionStorage`가 트리 로직·인덱스(`byId`/`labelsById`)·id 생성을 소유하고, JSONL/InMemory는 `setLeafId`/`appendEntry` 두 추상 메서드만 구현한다. 거친 부분: 8자 id 공간(2^32)의 충돌은 append 시 100회 재시도로 방어하지만, **파일 로드 경로(`parseEntryLine`)는 id 유일성을 재검증하지 않는다.** 손상·수동편집으로 중복 id가 들어오면 `byId` 맵에서 나중 것이 앞 것을 덮는다.

---

## 2. 저장 계층 — JSONL 디스크 vs in-memory, 그리고 "SQLite only"의 사각

공통 계약은 `SessionStorage` 인터페이스([`types.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/types.ts)`:483`)와 추상 `BaseSessionStorage`([`storage-base.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/storage-base.ts)`:54`)다. 베이스가 4개 파생 자료구조(`entries` 삽입순 배열, `byId`, `labelsById`, `leafId`)와 순회·인덱스 로직을 전부 소유하고, 서브클래스가 오버라이드하는 건 **`setLeafId`와 `appendEntry` 딱 두 개**뿐이다(`:158-159`). 즉 *영속 방식*만 다형이고 *트리 의미론*은 공유된다.

- **디스크**: `JsonlSessionStorage`. 한 세션 = 한 `.jsonl` 파일. 첫 줄이 `SessionHeader {type:'session', version:3, id, cwd, parentSession?}`([`jsonl-storage.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/jsonl-storage.ts)`:13-20`), 이후 각 줄이 하나의 엔트리 JSON. 쓰기는 오직 `appendFile` 한 줄 추가(`:223-238`)로만 일어난다. 기존 줄 수정·삭제는 코드에 존재하지 않는다.
- **메모리**: `InMemorySessionStorage`. `appendEntry`가 `recordEntry`만 호출 — 파일 I/O 없음. 주석이 명시하듯 테스트·ephemeral 전용. 같은 베이스를 공유하므로 메모리와 JSONL이 동일 동작을 노출한다.

**읽기는 전량 로드다.** `loadJsonlStorage`([`jsonl-storage.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/jsonl-storage.ts)`:145-171`)가 `readTextFile`로 파일 전체를 통째로 읽고 모든 줄을 파싱해 배열을 만든다. 스트리밍·부분 로드 경로는 없다. `maxLines:1`로 첫 줄만 읽는 최적화는 오직 `list()`의 헤더 스캔에만 있다.

**쓰기 원자성은 약하다.** Node 구현([`nodejs.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/env/nodejs.ts)`:489-498`)은 `fsPromises.appendFile` 단일 호출이 전부 — fsync·락·temp+rename 없음. 손상 방지는 *읽기 시점 검증*(`parseHeaderLine`/`parseEntryLine`)에 의존한다. 부분 기록된 마지막 줄은 파싱 실패로 전체 open을 throw시킬 수 있다(줄 단위 tolerant 재개 없음).

### "SQLite only"와의 관계 — 이것이 이 저장 계층에서 가장 미묘한 지점이다

CLAUDE.md의 "Storage default: SQLite only"는 "OpenClaw-owned runtime state, caches, queues..."를 대상으로 한다. 세션 `.jsonl`은 `@openclaw/agent-core` 소유의 **version 3 헤더를 가진 하네스 트랜스크립트 파일 포맷**이고, 성격상 named product artifact(코딩 에이전트 세션 로그)라 SQLite 대상인 "앱 상태/캐시"와 범주가 다르다.

**정정:** 이 예외를 *명시적 계약으로 선언한 doc/AGENTS.md는 검색 결과 존재하지 않는다.* `docs/`의 jsonl 언급은 전부 trajectory·acp-stream 로그(별개 artifact)이고, `packages/agent-core`에 스코프 AGENTS.md도 없다. 따라서 "named artifact라 허용된다"는 것은 **소스에 박힌 계약이 아니라 합리적 추론이다.** (미확인: 이 예외의 명시적 계약 선언 위치.)

> **[설계 평가]** 강점: 영속 정책과 트리 의미론의 분리가 깔끔해 테스트 더블이 prod와 같은 코드경로를 태운다. 트레이드오프: open이 O(파일 크기)라 긴 세션은 로드 비용이 선형이다. 거친 부분: 내구성이 얕아(fsync·락 없음) 크래시 시 마지막 줄 부분 기록 → 다음 open 전체 실패 위험. 또 `parentId` 무결성이 open 시점 검증에 전적으로 의존해, 파일이 한 번 깨지면 그 브랜치는 통째로 못 읽는다(자기수복 없음).

---

## 3. compaction 재배선 — prepare → compact → append, 그리고 before/after DAG

여기가 논지의 심장이다. Compaction은 트리를 자르지 않는다. **마커 노드 하나를 잎 뒤에 붙일 뿐이고, "가리기"는 저장이 아니라 조립 시점에 일어난다.**

### 3단계

**(1) prepare** — [`compaction.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/compaction/compaction.ts)`:630-706` `prepareCompaction`. 이전 compaction의 `firstKeptEntryId`부터 `boundaryStart`를 잡고, `findCutPoint`([`compaction.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/compaction/compaction.ts)`:385-435`)로 유지/접힘 경계를 정한다. `findCutPoint`는 `endIndex`부터 역방향으로 토큰을 누적해 `keepRecentTokens`에 도달한 지점 이상의 첫 유효 cut을 고른다. 유효 cut은 `user`/`assistant`/`bashExecution`/`branchSummary`/`compactionSummary`/`custom_message` 지점만이고 `toolResult`는 제외 — tool call과 결과가 갈라지지 않게 한다. cut이 turn 중간이면 `isSplitTurn=true`로 표시한다.

**(2) compact** — LLM 요약 생성. 두 번째 이후는 이전 마커의 summary를 `previousSummary`로 이어받아 `UPDATE_SUMMARIZATION_PROMPT`로 "기존 요약 보존 + 신규 반영" 갱신을 한다. `boundaryStart`를 이전 `firstKeptEntryId`로 잡으므로 이미 접힌 구간은 재요약하지 않고, 하나의 최신 마커로 갈아끼운다. split turn이면 history 요약과 turn-prefix 요약을 병렬 생성해 `---Turn Context---`로 이어붙인다.

**(3) append** — [`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:187-205` `appendCompaction`. `type='compaction'`, `parentId=현재 leaf`, `summary`/`firstKeptEntryId`/`tokensBefore`를 담은 **새 노드를 붙이고 기존 노드는 하나도 손대지 않는다.**

### "가리기"의 실제 코드

`firstKeptEntryId`가 우회 스위치다. `buildSessionContext`([`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:68-95`)는 경로에 compaction이 있으면 (a) 합성 요약 메시지를 먼저 push하고, (b) 마커 앞 구간은 `foundFirstKept` 플래그로 `firstKeptEntryId`를 만나기 전까지 전부 스킵, (c) 마커 뒤 신규 노드는 전부 append한다.

```ts
// session.ts:83-92 — "가리기"는 여기, 조립 시점에만 일어난다
let foundFirstKept = false;
for (let i = 0; i < compactionIdx; i++) {
  const entry = pathEntries[i];
  if (entry.id === compaction.firstKeptEntryId) foundFirstKept = true;
  if (foundFirstKept) appendMessage(entry);
}
```

### before / after DAG (마커 재배선의 시각화)

```
BEFORE compaction — 활성 경로가 통째로 재생됨
                                            leaf
  u1 ── a1 ── u2 ── a2 ── u3 ── a3 ── u4 ── a4
  └──────────────── 전부 모델에 보임 ──────────────┘

APPEND (파일에는 노드가 하나도 안 지워짐):
                                            firstKeptEntryId=u3
  u1 ── a1 ── u2 ── a2 ── u3 ── a3 ── u4 ── a4 ── [C]  ← 새 compaction 마커, parentId=a4
  │                        │                        └ leaf 여기로 이동
  └─ 여전히 디스크·byId에 존재, fork/재개에서 접근 가능

AFTER — buildSessionContext가 같은 경로를 접었을 때 모델이 보는 것:
  [요약메시지(C.summary)] ── u3 ── a3 ── u4 ── a4
   └ u1..a2 는 "우회"됨 (firstKeptEntryId 앞이라 스킵) — 삭제 아님
```

즉 compaction은 트리를 자르는 게 아니라, 새 마커 노드를 부모 링크로 매달아 **조립 함수가 `firstKeptEntryId` 앞을 건너뛰도록** 만드는 뷰 변환이다. 히스토리 재작성(rebase) 같은 위험이 구조적으로 없다.

### model 유도까지 같은 원리

**정정:** 조립 시점의 상태 접기는 마커만 보는 게 아니다. `buildSessionContext`는 `model_change` 마커뿐 아니라 **assistant 메시지의 `provider`/`model`에서도** model을 마지막 값으로 갱신한다([`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:38-39`). "model은 `model_change` 마커의 마지막 값"이라는 설명은 불완전하다 — assistant 메시지도 실측 소스다.

> **[설계 평가]** 강점: 저장은 순수 append-only 불변 트리, 가리기는 조립 시점 계산으로 분리 — 원본이 남아 재개·분기·감사에 접근 가능하고, `firstKeptEntryId` 포인터 하나로 결정적으로 재현된다. 거친 부분: 토큰 예산이 마지막 assistant usage 실측 + 그 뒤 문자수/4 추정의 혼합이라 tool-heavy tail이나 이미지(4800 고정 추정)에서 벌어질 수 있다. cut point 규칙이 `findValidCutPoints`/`findCutPoint`/`findTurnStartIndex` 세 함수에 분산돼 경계 불변식을 한눈에 읽기 어렵다.

### 이 섹션의 함정들

- **`firstKeptEntryId` 노드 자체는 "유지" 쪽이다.** 조립은 그 id를 만나는 순간 `foundFirstKept=true`로 켜고 *즉시 append*한다(`:86-90`). 반면 요약 범위(`prepareCompaction`)는 `historyEnd` *미만*이라 경계 노드를 요약에서 제외한다 — 조립과 요약이 대칭적으로 경계 노드를 유지 쪽에 둔다.
- **compaction 마커는 스스로를 요약에서 배제한다.** `getMessageFromEntryForCompaction`이 `type==='compaction'`이면 `undefined`를 반환해([`compaction.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/compaction/compaction.ts)`:112-117`) 이전 마커가 재귀적으로 요약에 중첩되지 않는다.
- **경로에 compaction이 여러 개여도 최신 하나만 유효하다.** 조립 루프가 compaction을 만날 때마다 덮어써(`session.ts:40-42`) 마지막 것만 남긴다. 이전 마커들은 자동 무시되지만 트리엔 남아 저장 증가에 기여한다.
- **연속 compaction 방지 불변식.** 경로 마지막이 이미 compaction이면 `prepareCompaction`은 no-op이고, harness `compact()`는 이를 "Nothing to compact" 에러로 승격한다([`agent-harness.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/agent-harness.ts)`:833-834`). 최소 한 개의 신규 message가 붙기 전엔 마커 위에 마커를 곧바로 쌓지 못한다.

---

## 4. branch-summarization — 같은 트리 위의 또 다른 우회 노드

**정정:** 여기서 "branch"는 서브에이전트 실행이나 도구 호출 하위트리가 *아니다.* 세션 트리(`parentId` DAG)에서의 순수 대화 분기다. 트리거는 `navigateTree()` 하나뿐이다([`agent-harness.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/agent-harness.ts)`:887`).

사용자가 과거 지점으로 되감아 다른 가지로 이동할 때, 버려지는 옛 가지를 LLM으로 압축해 "## Goal / Progress / Next Steps" 요약을 만들고, 이를 **새로 활성화되는 가지의 머리에** `branch_summary` 노드로 삽입한다.

branch의 정의는 코드로 이렇다. `collectEntriesForBranchSummaryFromBranches`([`branch-summarization.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/compaction/branch-summarization.ts)`:94-126`)가 target 경로를 뒤에서부터 훑어 `oldPath`에 있는 첫 id를 최심 공통조상으로 잡고, old 경로에서 그 조상 다음(`index+1`)부터 끝까지를 "버려지는 가지"로 슬라이스한다. 공통조상이 없으면 옛 가지 전체가 대상이다.

**반직관 (정정):** 요약 노드는 "요약 대상 가지(옛 가지)"가 아니라 **"새로 이동한 가지"의 잎을 부모로 삼아** 붙는다. `moveTo`([`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:263-285`)가 먼저 `setLeafId`로 활성 가지를 target으로 바꾼 뒤, `parentId = 그 새 잎`인 `branch_summary`를 append한다. `fromId`만 옛 위치를 기록한다. 즉 "떠난 가지의 요약"이 "도착한 가지의 머리"에 놓인다 — 위치가 직관과 반대다.

```
       common ancestor
  root ── u1 ── a1 ── u2 ── a2 ── a3   ← 옛 가지(버려짐), 요약 대상
                 │
                 └─ [navigateTree(u2)]
                    u2 ── [BS]          ← branch_summary, parentId=u2(새 잎)
                          └ 옛 가지 요약이 여기 매달림, fromId=a3
```

컨텍스트 재조립 시 `buildSessionContext`가 `branch_summary`를 `createBranchSummaryMessage`로 되살려 모델 컨텍스트에 넣는다([`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:61-64`). 즉 **compaction 마커와 branch_summary 마커는 같은 append-only 트리 위의 서로 다른 종류의 우회 노드로 공존한다** — 이것이 논지의 재확인이다.

> **[설계 평가]** 강점: 트리를 파괴적으로 재배선하지 않고 요약 노드를 새 가지 위에 얹어, 옛 가지 엔트리는 그대로 남아 되돌아가기가 가능하면서 활성 컨텍스트엔 응축본만 실린다. DAG 불변식(각 노드 단일 부모)을 유지하는 깔끔한 방식. compaction과 `estimateTokens`·시스템 프롬프트를 공유하되 본문(`BRANCH_SUMMARY_PROMPT`)은 분리. 거친 부분: 예산 임박 시 "이미 압축된 노드는 예산*0.9면 한 번 더 포함" 규칙이 매직넘버로 하드코딩. `navigateTree`의 5단계 오케스트레이션(newLeafId 계산·훅 병합·요약 details 조립)이 한 메서드에 몰려 있다.

이 섹션의 함정:
- `getMessageFromEntry`가 `toolResult`/`leaf`/`label`/`session_info` 역할을 `undefined`로 버려서([`branch-summarization.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/compaction/branch-summarization.ts)`:127-164`), 가지 요약 프롬프트엔 assistant 메시지 안의 `toolCall` 인자만 들어가고 도구 *결과*는 거의 빠진다.
- 요약 생성은 `options.summarize` + hook 오버라이드에 이중으로 걸린 opt-in이라, 산출이 없으면 그냥 leaf 이동으로 degrade된다.
- **미확인:** `src/agents/sessions/compaction/branch-summarization.ts`라는 동명 sibling 파일이 실재하나, `packages/agent-core` 쪽과의 관계·중복 여부는 미확인.

---

## 5. 컨텍스트 조립 — DAG → messages, 그리고 "context-engine"이라는 껍데기

여기서 두 개의 서로 다른 "컨텍스트 엔진"을 구별해야 한다. 이름이 같아 오해를 부른다.

1. **`src/context-engine/*`** — 컨텍스트 관리를 플러그인화하기 위한 얇은 registry/contract 계층. 기본값 `LegacyContextEngine`은 `ingest`=no-op(`{ingested:false}`), `assemble`=pass-through(`estimatedTokens:0`), `compact`=런타임 위임이다([`legacy.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/context-engine/legacy.ts)`:22-88`). **기본 경로에서 이 계층은 사실상 아무 투영도 하지 않는다.**
2. **`packages/agent-core` + `embedded-agent-runner`** — 진짜 DAG→messages 투영·토큰 계산·트리밍이 여기 있다.

**정정:** "컨텍스트 엔진"이라는 이름과 달리, 기본 설정에서 DAG→messages 투영은 context-engine이 아니라 `buildSessionContext`([`session.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/agent-core/src/harness/session/session.ts)`:28`)와 attempt.ts 파이프라인이 한다. `src/context-engine/*`는 미래의 플러그인 훅 + 레거시 호환 어댑터에 가깝다.

실제 조립 파이프라인은 `sanitizeSessionHistory → validateReplayTurns → filterHeartbeat → limitHistoryTurns → repairToolUseResultPairing`로 정제된 뒤([`attempt.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/embedded-agent-runner/run/attempt.ts)`:2918-3074`), context-engine `assemble`이 *옵셔널로* 얹히고, 마지막에 overflow precheck가 돈다.

**토큰 카운팅은 실제 토크나이저가 아니다.** [`preemptive-compaction.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/embedded-agent-runner/run/preemptive-compaction.ts)`:19-24`가 텍스트=4, tool_result=2, JSON=3 chars/token, 메시지 경계 오버헤드=12, 이미지=2000 토큰 상수로 추정해 예방적 compaction/truncation을 라우팅한다. 즉 조립 시점 overflow 판정은 근본적으로 근사치다.

견고성 계층은 registry가 resolve된 엔진을 두 겹의 Proxy로 감싼다. `wrapContextEngineWithSessionKeyCompat`는 외부 엔진이 레거시 키를 zod로 거부하면 에러 메시지 정규식으로 감지해 재시도하고, `wrapContextEngineWithRuntimeQuarantine`는 non-default 엔진이 던지면 프로세스 단위로 quarantine하고 legacy로 폴백한다([`registry.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/context-engine/registry.ts)`:293-343, 767-983`).

> **[설계 평가]** 강점: DAG 접기 로직이 단일 순수 함수로 응집돼 append-only 불변성과 잘 맞고, 서드파티 엔진 실패를 quarantine→legacy로 흡수하는 방어가 견고하다. 거친 부분: 기본 경로에서 context-engine은 껍데기라 실제 무게중심은 attempt.ts(5000+ LOC)와 agent-core에 있다. 토큰 카운팅이 chars/token 상수 휴리스틱이라 provider 실측과 어긋나면 precheck가 통과시켜도 provider가 거부하거나 불필요한 예방 compaction이 돈다. 레거시 키 호환을 위한 에러 문자열 정규식 매칭(`LEGACY_UNKNOWN_FIELD_PATTERNS`)은 깨지기 쉽고 표면적이 크다.

함정:
- **기본 엔진의 `assemble`은 `estimatedTokens:0`을 돌려준다.** "엔진이 토큰을 센다"고 오해하기 쉽지만, 실제 판정은 별도의 preemptive-compaction 휴리스틱이 한다.
- **quarantine 폴백에서 `compact`와 `prepareSubagentSpawn`만 폴백하지 않고 재던진다**(`registry.ts:838-840`). 커스텀 엔진 실패 시 compaction 경로만 하드 실패한다.
- **미확인:** 이미지(attachment)가 조립 순서 어디서 messages에 합류하는지, DAG 엔트리로 저장되는지 vs 현재 프롬프트에만 붙는지는 라인 단위로 확정하지 못했다. precheck는 image당 2000토큰 상수로만 계상.

---

## 6. 세션 정체성·계보 — 재개는 연장, fork는 새 계보

식별자는 세 층위로 분리된다([`packages/acp-core/src/types.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/acp-core/src/types.ts)`:22-96`):

1. **`sessionKey`** — 클라이언트에 노출되는 안정 키이자 저장 store의 키.
2. **`sessionId`** — 트랜스크립트 파일 1개당 하나인 물리 식별자. fork 때마다 `randomUUID`로 새로 발급.
3. **`identity`(`acpxSessionId`/`agentSessionId`)** — 백엔드가 나중에 부여하는 resume용 stable id.

**resume vs fork의 근본 차이는 저장소 레벨에서 갈린다.** resume은 identity의 stable id로 *같은 파일*을 이어가는 연장이다. fork(branch/restore)는 `forkCompactionCheckpointTranscriptAsync`([`session-compaction-checkpoints.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/session-compaction-checkpoints.ts)`:333-393`)가 소스 트랜스크립트를 `sourceLeafId`까지 잘라 *새 파일 + 새 sessionId*로 복제하고, 헤더 `parentSession`에 원본 경로를 남긴다. **원본은 불변** — 여기서도 논지가 반복된다.

branch와 restore는 같은 fork 프리미티브를 재사용하되 저장 키만 다르다([`sessions.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/gateway/server-methods/sessions.ts)`:1663-1884`):
- **branch** = `buildDashboardSessionKey`로 *새 세션 키*를 만들어 `store[nextKey]`에 기록 → 원본과 격리된 별도 세션.
- **restore** = 활성 run을 먼저 인터럽트한 뒤 원래 `store[canonicalKey]`를 덮어써 되감고, `preserveCompactionCheckpoints:true`로 체크포인트를 유지.

compaction의 물리 재배선은 이 계층에서 한 번 더 나온다. `buildSuccessorEntries`([`compaction-successor-transcript.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/src/agents/embedded-agent-runner/compaction-successor-transcript.ts)`:111-202`)는 `truncateAfterCompaction` 설정 시 요약된 message 엔트리를 제거하고, **남은 엔트리의 `parentId`가 제거 대상이면 살아있는 조상까지 거슬러 올라가 재부모화(reparent)한다.** 이것은 harness의 "지우지 않는" append-only 접기와 대비되는, *물리 파일을 잘라내는* 별도 계층이다 — 새 sessionId로 rotate되므로 원본 파일 자체는 여전히 보존된다.

> **[설계 평가]** 강점: 식별자 3층위 분리로 fork가 sessionId를 바꿔도 클라이언트 키는 유지되고, "resolved id는 pending 관찰로 downgrade 금지" 불변식([`session-identity.ts`](https://github.com/openclaw/openclaw/blob/0fc5a57a34409782c8e0c9260cedbf788ed382d8/packages/acp-core/src/runtime/session-identity.ts)`:161-179`)이 provisional id 유출을 구조적으로 막는다. branch/restore가 한 fork 프리미티브를 재사용하되 키만 다르게 둔 것도 깔끔하다. 거친 부분: successor의 in-place reparent + thinking 서명 제거 로직이 한 함수에 몰려 있어 `parentId` 체인 무결성 위반 시 디버깅이 어렵다. lineage-meta가 `parentSessionKey`/`spawnedBy` 두 세대 필드를 계속 병존시켜야 한다(정규화 계층 흡수).

함정·정정:
- **interaction-mode는 컨텍스트 조립·compaction에 전혀 개입하지 않는다.** grep 0 hits — 유일한 소비처는 delivery 억제와 A2A skip뿐인 순수 "전달 경로 게이트"다.
- **비대칭:** branch는 활성 run을 인터럽트하지 않지만 restore는 반드시 먼저 인터럽트한다(같은 canonical 키를 덮어쓰므로). restore만 체크포인트를 보존하고 branch는 비운다 — 되감긴 세션은 또 되감을 수 있어야 하지만 갈라낸 branch는 원본 체크포인트를 물려받지 않는 의도적 설계.
- **미확인:** `runId`가 저장소(트랜스크립트/lineage)에 영속되는지 여부, `ledgerSessionId`가 트랜스크립트 `sessionId`와 어떻게 대응/분기되는지.

---

## 7. 종합 — 하나의 원시요소가 다섯 가지 편집을 표현한다

이 엔진의 지적 무게중심은 딱 하나다. **`parentId` + `leaf` 마커라는 최소 원시요소 위에서, 모든 "히스토리 편집"이 파괴적 mutation이 아니라 append + 조립 규칙 변경으로 환원된다.**

- 라벨 변경 = 새 `label` 노드 append + 리플레이.
- 되감기 = `leaf` 이동.
- compaction = `compaction` 마커 append + `firstKeptEntryId` 앞 스킵.
- 가지 요약 = `branch_summary` 마커를 새 가지 머리에 append.
- fork = 경로를 새 파일로 복제(원본 불변).

다섯 개 모두 **저장을 절대 건드리지 않는다.** 이것이 프롬프트 캐시 결정성(옛 transcript bytes 보존), fork·감사, 재개를 전부 공짜로 만든다.

**강점 요약:** 불변 트리 + 조립 시점 접기의 분리; 영속 정책과 트리 의미론의 다형 분리(2개 추상 메서드); 서드파티 엔진 quarantine 폴백.

**거친 부분 요약:** (1) 저장이 단조 증가 — 토큰은 줄어도 파일은 안 준다. (2) open이 O(파일)이고 내구성이 얕다(fsync·락 없음). (3) 토큰 회계가 chars/token 휴리스틱이라 overflow 판정이 근사치. (4) 경계·cut point 로직이 여러 함수에 분산돼 불변식을 한눈에 못 읽는다. (5) 8자 id 유일성이 append 시점에만 방어되고 로드 시엔 재검증 안 됨.

---

## 부록. 검증 메모

핀 SHA `0fc5a57a` 기준, 실제 파일을 열어 확인한 사실과 미확인 항목:

**직접 확인:**
- 세션 노드 union 정확히 10종 — `types.ts:454-464`.
- append-only: `appendCompaction`/`appendEntry`/`setLeafId` 모두 `appendFile` 한 줄 추가, 기존 노드 무수정 — `session.ts:187-205`, `jsonl-storage.ts:223-238`.
- 가리기는 조립 시점(`firstKeptEntryId` 스위치) — `session.ts:83-92`.
- `leafIdAfterEntry`가 leaf면 targetId 반환 → leaf가 parentId 부모 후보 아님 — `storage-base.ts:42-44`.
- **정정 1 확인:** model이 `model_change`뿐 아니라 assistant 메시지에서도 유도됨 — `session.ts:38-39`.
- version!==3 즉시 throw, 하위호환 리더 없음 — `jsonl-storage.ts:60-62`.
- `LegacyContextEngine` assemble pass-through / `estimatedTokens:0` — `legacy.ts:51-54`.
- commonAncestor 역순 스캔 + slice(ancestor+1), `getMessageFromEntry`가 toolResult/leaf/label 버림 — `branch-summarization.ts:100-164`.

**미확인(정직 표기):**
- 세션 JSONL이 "SQLite only" 예외라는 *명시적 doc/AGENTS.md 계약* — 검색 결과 없음. "named artifact" 논리는 추론.
- `src/agents/sessions/compaction/branch-summarization.ts` 동명 sibling과 agent-core 버전의 관계.
- 이미지 엔트리의 DAG 저장 여부·조립 순서 정확 위치.
- `runId`의 계보 영속 여부, `ledgerSessionId`의 정확한 의미.
- 8자 id 파일 로드 시 중복 검증 부재의 실제 발생 조합.

관련 핵심 파일(repo-root 상대): `packages/agent-core/src/harness/session/session.ts`, `.../storage-base.ts`, `.../jsonl-storage.ts`, `.../uuid.ts`, `packages/agent-core/src/harness/compaction/compaction.ts`, `.../branch-summarization.ts`, `packages/agent-core/src/harness/types.ts`, `src/context-engine/legacy.ts`, `src/context-engine/registry.ts`, `src/agents/embedded-agent-runner/run/attempt.ts`, `.../preemptive-compaction.ts`, `.../compaction-successor-transcript.ts`, `src/gateway/server-methods/sessions.ts`, `src/gateway/session-compaction-checkpoints.ts`, `packages/acp-core/src/types.ts`, `packages/acp-core/src/runtime/session-identity.ts`.