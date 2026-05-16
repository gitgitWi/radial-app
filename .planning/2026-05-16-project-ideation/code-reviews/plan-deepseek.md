---
title: "Plan review — agentflow-project-ideation — deepseek-v4-pro"
type: plan-review
task: agentflow-project-ideation
task_date: 2026-05-16
created: 2026-05-17
last_updated: 2026-05-17
status: active
size: L
parent: ../plan.md
related:
  - ./plan-summary.md
reviewer: opencode-go/deepseek-v4-pro
cli: opencode
verdict: ship-after-minor-edits
prompted_against:
  - /Users/jh/Codes/radial-app/.planning/2026-05-16-project-ideation/plan.md
---

## 강점 (Strengths)

- **데이터 모델과 UI 모델의 분리** (Section 3.2). `Ticket.status`와 `AgentSession.state`를 별도로 관리하고, UI 컬럼은 데이터 상태를 그대로 노출하지 않아도 된다고 명시한 점은 탁월한 설계 결정이다. 특히 `assignee_type`을 `human`/`agent`/`none`으로 분리하여 Kanban의 "In Progress" 컬럼에서 발생하는 type/state 혼재 문제를 근본적으로 해결했다.

- **Raw log always wins 원칙** (Section 3.3, 3.5). hook 파싱 실패나 상태 추론 오류가 발생해도 원본 stdout/stderr가 유실되지 않도록 설계한 점은 복구 가능성의 핵심 보험이다. CLI Bridge나 hook 신뢰성과 무관하게, raw log가 최종 진실의 원천(source of truth)이라는 방침이 명확하다.

- **Phase 게이트 구조** (Section 5). "통과 기준을 충족하지 못하면 다음 phase로 넘어가지 않는다"는 엄격한 gating과, 실패 시 복귀 경로(`wterm 실패 → xterm.js fallback`, `복구 불안정 → tmux lifecycle 보강`)를 Mermaid diagram으로 시각화했다. 대부분의 프로젝트 plan이 타임라인만 나열하는 것과 대비된다.

- **CLI-specific abstraction over protocol lock-in** (Section 3.5). MCP 서버를 조기 도입하지 않고, `agentflow` CLI + Skill/Instruction 패키지로 시작하는 결정은 현실적이다. 지원 대상 CLI 4종이 동일한 protocol/tooling 모델을 공유하지 않는다는 전제가 정확하다.

- **Mobile 기술 선택 이연** (Section 5, Phase 3). Tauri v2 Mobile vs React Native를 Phase 3 진입 직전 spike로 판단하도록 설계한 점은 중복 구현 위험 회피에 효과적이다. "지금 결정할 수 없는 것은 지금 결정하지 않는다"는 원칙이 잘 적용되었다.

- **앱 크래시 복구 경로의 명시적 설계** (Section 3.3, Mermaid diagram). `tmux ls`와 SQLite metadata를 교차 검증하여 `orphaned`/`recovered` 상태로 전환하는 흐름이 구체적으로 기술되어 있다.

---

## 위험 및 누락 (Gaps and risks)

### 1. AgentSession 상태 전이의 race condition (심각)

Plan은 AgentSession 상태 목록(`created`, `worktree_preparing`, `queued`, `starting`, `running`, `waiting_input`, `exited`, `failed`, `orphaned`, `recovered`)을 나열했지만, **상태 전이 간 경합 조건(race condition)**을 다루지 않았다.

구체적 문제점:

- **`worktree_preparing` ↔ `queued` 동시성**. Plan은 다음 순서를 가정한다: QueueItem 생성 → worktree 생성 → 실행. 그러나 QueueItem이 생성되어 `queued` 상태에서 대기 중일 때, 외부 watchdog이 "실행 슬롯이 있다"고 판단해 worktree 생성을 trigger하고, 동시에 사용자가 UI에서 Ticket을 다른 상태로 drag/drop하는 경우 — 어떤 상태 변경이 우선하는가? `queued`와 `worktree_preparing` 사이에 **compare-and-swap(CAS) 시맨틱이 누락**되어 있다. 예: "Tick 상태가 `queued`이고 QueueItem.state가 `pending`일 때만 worktree_preparing으로 전환 가능" 같은 원자적 조건이 필요하다.

- **`starting`의 정의가 모호하다**. Plan은 "프로세스 시작 시 `running`"이라고 명시하지만, `starting` → `running` 전환의 trigger가 정확히 무엇인지 불분명하다. CLI 프로세스가 spawn 되어 PID가 할당된 시점인가, 아니면 첫 stdout 라인이 관측된 시점인가? tmux session은 생성되었는데 CLI spawn이 config parse 오류로 즉시 exit되는 상황에서, AgentSession은 `starting`에 머무르는가, `failed`로 가는가? 이 모호성은 orphaned 판정 로직에도 영향을 준다.

- **`waiting_input` 검출**. 에이전트가 터미널에서 stdin 입력을 block하고 있을 때, 이를 "running"과 구별하는 메커니즘이 없다. `tmux capture-pane`의 마지막 N줄을 정규식으로 검사할 것인가, 아니면 CLI별 hook을 통할 것인가? 전자는 CLI 출력 포맷에 의존하고, 후자는 4개 CLI마다 별도 구현이 필요하다. 두 방식 모두 기술되어 있지 않다.

- **Orphaned 판정 알고리즘 부재**. Plan은 "앱 재시작 후 tmux 세션은 있는데 DB state가 불명확하면 `orphaned`로 표시"한다고만 명시했다. 구체적 판정 알고리즘이 없다:
  - `tmux ls`에는 있으나 AgentSession.state가 `running`/`starting`이 아닌 경우 → orphaned?
  - AgentSession.state가 `running`이지만 `tmux ls`에 없는 경우 → `failed`? `orphaned`?
  - tmux 세션 안에 CLI 프로세스가 zombie 상태인 경우는 어떻게 구분하는가?
  - PID가 재할당(reuse) 된 극단 케이스에서 `AgentSession.pid`가 우연히 다른 프로세스를 가리키는 오탐(false positive)을 어떻게 방지하는가?
  - 복구 시점에 `worktree`가 dirty인지 판정하기 위해 `git status --porcelain`을 실행할 텐데, 이 자체가 수 초 지연을 유발할 수 있다. 복구 UI에서 spinner를 보여주는 전략이 언급되지 않았다.

### 2. SQLite 단일 DB에 SessionEvent raw log append 시 WAL/락 충돌 (심각)

Plan은 `SessionEvent` 테이블을 사용해 raw log를 append한다고 명시했으나, 다음 충돌 시나리오를 고려하지 않았다:

- **N개 Agent가 동시에 동일 SQLite 파일에 write한다는 전제**. Phase 2에서 N=3 Agent가 병렬 실행될 때, 각 tmux session의 `pipe-pane`이 Tauri Rust backend를 통해 동일 `.db` 파일에 `SessionEvent` row를 insert한다. Rust 측에서 단일 connection pool을 사용한다면:
  - SQLite는 writer-locking-only 모델이다. WAL 모드라고 해도 **동시 writer는 직렬화**된다.
  - 3개 Agent가 초당 수백 라인의 stdout을 생성한다면, insert contention이 발생할 수 있다.
  - 더 심각한 문제: **Tauri의 Rust command handler는 비동기로 실행**된다. 여러 tokio task가 `spawn_blocking`이나 `rusqlite::Connection`을 공유할 때, `SQLITE_BUSY`가 발생할 가능성이 있으나 plan에 retry 전략이나 write-ahead batching 전략이 없다.

- **Raw log를 SessionEvent 테이블에 저장하는 것의 DB 크기 문제**. Phase 4까지 raw log를 sync에서 제외한다고 해도, 로컬 SQLite DB는 수일 dogfood로 수백 MB에 도달할 수 있다. Vacuum 정책, row 단위가 아닌 파일 기반 raw log 저장 대안, `ATTACH DATABASE`로 log DB를 분리하는 전략 중 어느 것도 언급되지 않았다.

- **`tmux pipe-pane`과 `portable-pty` capture의 중복** (Section 3.7에서 언급만 하고 해결책 없음). Plan은 "중복 문제"를 인식했지만, 중복 시 어느 쪽을 discard할지, 중복 감지(hash 또는 sequence number)는 어디서 할지 결정되지 않았다. 이 결정이 Phase 0에 포함되어야 한다.

### 3. Turso/libSQL Embedded Replica의 last-write-wins + conflict notification 실제 구현 가능성 (중간)

Plan은 Turso sync 방식을 "last-write-wins + conflict notification"으로 규정했으나:

- **libSQL Embedded Replica는 현재(2026년 5월 기준) last-write-wins를 기본 모델로 제공하지 않는다**. Embedded Replica는 read replica로서 write는 primary로 proxy된다. 클라이언트가 오프라인에서 write한 후 sync할 때 충돌 resolution은 application layer에서 구현해야 한다.
- Mac ↔ Web/Mobile 간 동시 write 충돌을 "last-write-wins"로 해결한다면, `sync_version` 컬럼이 낙관적 락(optimistic locking) 역할을 하게 된다. 그러나 plan은 충돌이 "발생했을 때" 어떻게 감지하는지 명시하지 않았다. `last_write_timestamp` 비교인가, vector clock + server-assigned stamp인가, 아니면 CRDT merge인가?
- "conflict notification"이 사용자에게 표시된다고 해서, 실제로 overwrite된 데이터를 복구할 수 있는가? 이전 버전을 별도 테이블에 보존하지 않으면 영구 유실이다. 개별 티켓에 대해 conflict가 발생해도 raw log 유실만 없다면 문제가 없을 수 있지만, `Ticket.title`, `Ticket.description`, `Comment.body` 등 핵심 텍스트 데이터의 conflict resolution에는 **3-way merge나 manual resolution UI** 가 필요할 가능성이 높다. Plan은 이 부분을 Phase 3 세부 설계로 이연할 뿐이다.

### 4. Worktree 격리에서 secrets/env 정책의 디폴트가 실제 dogfood에서 깨지는 시나리오 (중간)

Plan은 "기본은 자동 복사하지 않는다. workspace 설정에서 명시적으로 허용한 경우에만 복사한다"고 규정했다 (Section 3.6). 이 원칙은 보안 관점에서 올바르지만, **dogfood 초기부터 깨질 가능성이 높은 시나리오**가 누락되었다:

- **대부분의 실제 레포는 `.env` 파일을 필요로 한다**. Agent가 `npm run dev`나 `cargo build`를 실행하기 위해 최소한의 환경변수가 필요하다. 사용자가 "`.env` 파일을 복사하지 않음" 상태에서 첫 Agent 티켓을 만들면, Agent는 dependency install까진 성공하지만 runtime config 오류로 실패한다. 사용자 경험: "AgentFlow가 작동하지 않는다" → workspace 설정을 들여다보지 않고 포기.
- **복사 정책의 세분성 부재**. "명시적으로 허용한 파일만" 복사한다고 했지만, 어떤 파일을 허용할지 결정할 UI나 가이드가 Phase 1에 포함되어 있지 않다. workspace 최초 생성 시 "이 workspace는 어떤 env 파일이 필요합니까?" 라는 guided setup이 없으면, 사용자는 실패를 경험한 후에야 설정을 찾게 된다.
- **Worktree는 원본 repo의 `.env`를 상속하지 않으므로**, `AGENTFLOW_*` 환경변수 외의 모든 환경변수는 빈 값으로 CLI가 시작된다. Claude Code의 경우 `ANTHROPIC_API_KEY`가 없으면 작동하지 않는다. 이 키는 worktree마다 복사되어야 하는데, plan은 `.env` 파일만 언급하고 shell 환경변수/secret manager (예: macOS Keychain, 1Password CLI)에서의 주입 경로를 언급하지 않았다.
- **요약**: secrets 정책은 보안적으로 정답이지만, **"작동하지 않는 기본값"을 제공하면 dogfood가 시작되기 전에 사용자 이탈이 발생**한다. v1에서 최소한의 smart default 또는 workspace 생성 시 명시적 env 정책 선택 UI가 필요하다.

### 5. Yolo mode + worktree 격리만으로 충분한 안전 모델인지 (중간)

Plan은 "yolo mode"를 기본값으로 두고, worktree 격리로 변경사항을 분리한다고 설계했다 (Section 1.1, 3.6). 이 모델의 누락된 지점:

- **Worktree 격리는 파일시스템 수준에서만 안전하다**. Agent가 `rm -rf ~/important-data` 나 `git push --force origin main` (worktree 내에서 원격 브랜치 조작)을 실행하는 것을 막지 못한다. worktree는 원본 repo의 git 객체와 ref를 공유하므로, `git push --delete origin main`은 격리되지 않는다.
- **네트워크 side effect**. Agent가 `curl -X POST`로 production API를 호출하거나, `aws s3 rm`을 실행하는 것을 worktree 격리는 막지 못한다.
- **Plan은 workspace 단위 권한 정책을 Phase 4까지 미룬다** (Section 5, Phase 2 제외 범위: "정교한 권한 정책"). 즉, Phase 1~3 기간(dogfood ~ sync) 동안 "사용자가 agent 실행 결과를 diff로 검토하는 것"이 유일한 안전장치다. 그러나 Phase 1부터 "yolo by default"로 설정되어 있다면, **검토하기 전에 side effect가 이미 발생한 후**다.
- **권고사항**: v1에서 최소한 "workspace 단위 permission dry-run mode"를 도입할 필요가 있다. 예: `network: allow | deny | prompt`, `git.push: allow | deny`, `file.delete_outside_worktree: allow | deny` 같은 최소한의 denylist. 완전한 permission model은 Phase 4까지 이연하더라도, **기본 deny 몇 개는 yolo mode와 함께 v1에 포함**되어야 dogfood 자체가 위험하지 않다.

### 6. 기타 누락

- **tmux 설치 실패 시 UX** (Section 6). "dependency check, 설치 가이드, fallback 안내"라고만 언급되었다. macOS에서 tmux는 기본 설치되어 있지 않으며, brew를 통한 설치가 필요하다. homebrew 자체가 설치되지 않은 환경, 또는 brew install이 network timeout으로 실패하는 경우의 UX가 없다. 더 중요한 것은: **tmux 없이 Agent를 실행할 수 있는 fallback이 존재하는가?** `portable-pty`만으로 detached session을 구현할 수 있다면, tmux는 optional dependency가 될 수 있다. Plan은 tmux를 session manager로 고정했지만, 기본 설치 실패 시의 graceful degradation 전략이 없다.

- **Watchdog 디자인 상세 부재** (Section 3.3, Phase 2). "메모리 watchdog", "watchdog limit"이 언급되었지만, 어떤 프로세스 메트릭을 모니터링하는지(`getrusage`? `sysctl`? `ps`?), 어떤 threshold로 동작하는지, watchdog 자체가 crash 났을 때 재시작되는지 명시되지 않았다. 특히 macOS에서 프로세스 트리 모니터링(tmux → shell → CLI process → 하위 process들)의 총 RSS를 추적하는 방식이 필요하다.

- **`agentflow` CLI의 Unix domain socket 보안** (Section 3.5, 4.1). `AGENTFLOW_SOCKET` 환경변수를 통해 CLI가 소켓 접근 권한을 얻는다고 추정되나, socket path, permission model, 인증 방식이 명시되지 않았다. Worktree 내 CLI 프로세스가 Tauri 백엔드의 Unix domain socket에 접근할 때, **다른 worktree의 socket에 접근하지 못하도록 격리**하는 메커니즘이 필요하다.

- **Session replay의 데이터 출처** (Section 5, Phase 4). `SessionEvent`와 raw log를 replay한다고 했지만, replay 가능한 형식으로 stdout/stderr를 저장하려면 ANSI escape sequence + 터미널 크기(columns/rows) + 타이밍 정보가 필요하다. 단순 raw log append로는 replay가 불가능하다. `script` 명령어 수준의 terminal recording이 필요한데, plan은 이 차이를 인식하지 못했다.

---

## 구체적 제안 (Concrete suggestions)

### 제안 1: AgentSession 상태 전이에 CAS 프리컨디션 명시

**무엇을**: Section 3.3의 Agent 실행 파이프라인에 각 상태 전이의 precondition을 추가한다.

**예시**:
- `queued → worktree_preparing`: "Ticket.status == `queued` AND QueueItem.state == `pending`일 때만 전환. CAS 실패 시 QueueItem은 `skipped`로 표시하고 Ticket은 `queued` 유지."
- `starting → running`: "CLI 프로세스의 PID가 할당되고 stdout 첫 라인이 관측된 시점. spawn 실패 시 `failed`로 전환."
- `any → orphaned`: "앱 재시작 시 (a) AgentSession.state IN (`starting`, `running`, `waiting_input`) AND tmux ls에 없음 → `failed` (추정 사망), (b) tmux ls에 있고 AgentSession.state가 위 셋이 아닌 경우 → `orphaned`, (c) tmux ls에 있고 state가 맞으면 → `recovered`"

**왜**: 상태 전이 경합은 unreproducible bug의 주요 원인이다. CAS precondition이 없으면 multi-agent queue, 사용자 drag/drop, crash recovery가 동시 발생할 때 DB state가 모순될 수 있다.

**어느 섹션에**: Section 3.3 "Agent 실행 파이프라인" 하위.

---

### 제안 2: Raw log를 별도 SQLite 파일 또는 append-only 파일로 분리

**무엇을**: `SessionEvent` 테이블을 메인 DB에서 분리하여 `.agentflow/logs/<session-id>.log` 와 같은 append-only 파일로 저장하거나, `ATTACH DATABASE`로 별도 `.log.db` 에 저장한다.

**왜**: 메인 DB와 raw log의 write 특성이 다르다. 메인 DB는 소량의 구조화된 write가 대부분이지만, raw log는 초당 수백 줄의 append다. 동일 파일에서 두 패턴이 충돌하면 write contention과 WAL checkpoint latency가 모두 증가한다. 또한 raw log 전용 파일은 rotation, compression, retention 정책을 독립적으로 적용할 수 있다.

**어느 섹션에**: Section 4.1 "Local DB" 하위, 또는 Section 3.7 "터미널/PTY 전략"에 log storage 전략으로 추가.

---

### 제안 3: Phase 0에 log capture strategy 결정 포함

**무엇을**: Phase 0 통과 기준에 다음 항목을 추가한다:
- "`tmux pipe-pane`과 `portable-pty` capture 간 중복 제거 전략이 확정된다."
- "raw log storage format (DB row vs file vs hybrid)이 결정된다."

**왜**: Plan이 "중복 문제"를 인식만 하고(3.7), 해결책을 Phase 0 검증 항목에 포함하지 않았다. 이 결정 없이 Phase 1 구현에 진입하면 log storage 아키텍처를 나중에 바꾸는 비용이 크다.

**어느 섹션에**: Section 5, Phase 0 통과 기준.

---

### 제안 4: Workspace 생성 시 env 정책 guided setup UI 언급 추가

**무엇을**: Phase 1의 Workspace CRUD에 "env 파일 자동 감지 및 복사 정책 설정 wizard"를 포함한다.

**구체적**:
- Workspace 최초 생성 시, workspace root_path에서 `.env`, `.env.local`, `.envrc` 등을 자동 감지한다.
- 각 파일에 대해 "worktree에 복사하시겠습니까? [예/아니오]" 선택지를 UI로 제시한다.
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` 등 환경변수 주입을 macOS Keychain 또는 workspace `.env.agentflow`에서 로드하는 경로를 제공한다.

**왜**: "기본 복사 안 함" 원칙은 올바르지만, 이를 적용하는 UX가 없으면 Agent가 아무것도 실행하지 못하는 broken state에서 dogfood가 시작된다. 사용자가 포기하기 전에 guided setup이 필요하다.

**어느 섹션에**: Section 3.6 "Worktree와 workspace 준비", Phase 1 포함 범위.

---

### 제안 5: Phase 1에 최소한의 denylist 권한 정책 포함

**무엇을**: Phase 1 포함 범위에 "workspace별 최소 denylist 설정"을 추가한다.

**최소 스펙**:
- workspace 설정: `network: allow | deny`, `git.push: allow | deny`, `file.rm_outside_worktree: allow | deny`
- Agent CLI spawn 전에 denylist를 환경변수 또는 instruction으로 주입한다.
- denylist 위반 감지 시 티켓 상태를 `waiting`으로 변경하고 OS 알림을 발송한다.

**왜**: "Yolo by default" + "worktree 격리"만으로는 git push --force, 외부 API 호출, 파일시스템 탈출을 막을 수 없다. Phase 4까지 완전한 권한 모델을 기다리면, Phase 1~3 dogfood 자체가 리스크를 동반한다. Phase 1에 3개 denylist만 있어도 yolo mode의 위험을 크게 줄일 수 있다.

**어느 섹션에**: Section 5, Phase 1 포함 범위; Section 3.6 하위.

---

### 제안 6: Worktree에서 원격 ref 조작 방지 전략 추가

**무엇을**: Section 3.6에 worktree 생성 시 `git config --local receive.denyCurrentBranch updateInstead` 또는 push URL을 빈 값으로 설정하는 전략을 명시한다.

**왜**: Worktree는 `.git` 을 원본 repo와 공유하므로, Agent가 worktree 안에서 `git push` 하면 원본 repo의 remote로 push 된다. worktree 격리가 파일시스템 읽기/쓰기만 다룰 뿐 git remote 접근은 격리하지 않는다. Agent가 `rm -rf ~/`를 못 하더라도 `git push --force origin main`을 할 수 있으면 격리가 무의미하다.

**어느 섹션에**: Section 3.6 "Worktree와 workspace 준비".

---

### 제안 7: Session replay를 위해 terminal recording format 검토 포함

**무엇을**: Phase 4의 "session replay"를 `SessionEvent` raw log로 구현할 수 없다는 점을 인정하고, data capture strategy를 Phase 0에서 검토할 항목으로 추가한다.

**구체적**:
- `script` / `asciicast v2` / `ttyrec` 등 터미널 녹화 포맷을 검토한다.
- 필요하면 SessionEvent 외에 `tmux pipe-pane -t <session>:<window>.<pane> 'cat > .agentflow/logs/<session-id>.cast'` 방식으로 별도 recording stream을 유지한다.

**왜**: ANSI escape sequence + timing 정보 없이 raw stdout만으로는 터미널 replay가 불가능하다. Phase 4에서 "안 된다"를 발견하면 재설계 비용이 너무 크다.

**어느 섹션에**: Section 3.7 "터미널/PTY 전략", 또는 Section 5 Phase 0 통과 기준.

---

### 제안 8: Tmux 없이 Agent 실행할 수 있는 fallback 경로 고려

**무엇을**: `portable-pty`만으로 detached session을 생성하는 alternative path를 Phase 1 scope에 포함한다.

**왜**: macOS에서 tmux는 기본 설치되어 있지 않다. homebrew조차 없는 환경, 기업용 macOS에서 brew 금지된 환경을 고려하면, tmux를 hard dependency로 두는 것은 진입 장벽이다. `portable-pty`로 최소한의 session persistence만 구현해도 Phase 1 MVP의 핵심 목표인 "앱 종료 후에도 작업이 살아 있고, 재접속 가능"을 달성할 수 있다.

**어느 섹션에**: Section 3.7, Section 6의 "tmux 외부 의존성" 리스크 대응.

---

## Verdict

**ship after minor edits** — 핵심 아키텍처 결정(상태 모델 분리, raw log 중심 설계, phase gating)은 견고하나, AgentSession 상태 전이의 race condition, SQLite raw log write contention, yolo mode 기본값의 안전성, secrets 정책의 broken-by-default dogfood 시나리오에 대한 구체적 대응이 plan에 반영되어야 Phase 1으로 안전하게 진입할 수 있다. 위 8개 제안 중 제안 1, 2, 5는 Phase 1 진입 전 반영이 권장되고, 나머지는 Phase 0~1 세부 설계에서 다룰 수 있다.

<!-- council-flow:review-complete -->
