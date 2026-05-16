---
title: "Plan review summary — agentflow-project-ideation"
type: plan-summary
task: agentflow-project-ideation
task_date: 2026-05-16
created: 2026-05-17
last_updated: 2026-05-17
status: active
size: L
parent: ../plan.md
related:
  - ./plan-claude.md
  - ./plan-gemini.md
  - ./plan-kimi.md
  - ./plan-deepseek.md
reviewers:
  - claude-opus-4-7
  - gemini-3.1-pro-preview
  - opencode-go/kimi-k2.6
  - opencode-go/deepseek-v4-pro
missing_reviewers: []
---

# 플랜 리뷰 요약 — agentflow-project-ideation (plan.v3)

## 한 줄 평가

4명 리뷰어 모두 **"ship after minor edits"** — v3는 v2 대비 데이터 모델 분리, phase gate, orphaned/recovered 상태, raw-log 보존 원칙 등 핵심 결정 지점을 잘 닫았다. 다만 v1 구현 전에 일부 "측정/결정/기본값" 항목을 plan에 명시하지 않으면 Phase 1 dogfood가 의미 있는 검증이 되지 않거나 안전성 게이트가 깨질 수 있다.

## 합의된 강점 (3명 이상이 명시)

- **데이터 모델과 UI 모델의 분리** (Section 3.2). `Ticket.status` / `Ticket.assignee_type` / `AgentSession.state` 3축 구조. v2의 "In Progress(Human/Agent/Waiting/Failed)" 컬럼 혼재 문제를 근본적으로 해결.
- **"Raw log always wins" 원칙의 일관성**. hook/CLI Bridge/추론이 모두 실패해도 원본 stdout/stderr가 사실 계층으로 남도록 설계 — 3.3 파이프라인, 4.3 상태 전이, 6장 hook 리스크 대응까지 일관 적용.
- **Phase 게이트 구조와 실패 시 복귀 경로**. wterm 실패 → xterm.js fallback, 복구 불안정 → tmux lifecycle 보강 같은 분기를 mermaid로 시각화. 통과 기준이 비교적 측정 가능.
- **Skill / Instruction 패키지로 MCP 조기 도입 회피** (3.5). 지원 CLI 4종이 동일 protocol을 공유하지 않는 현실에서 실용적 선택. 개인용 local-first 디버깅성 측면에서도 타당.
- **orphaned / recovered를 AgentSession.state의 1급 상태로 포함** (3.2). local-first + tmux 조합의 가장 흔한 실패 모드(앱 강제 종료)를 정면 대응.
- **Turso sync와 raw log sync 제외 결정의 명시성** (Section 4.2, Phase 3 게이트). 비용/충돌/PII 문제를 한 번에 차단.

## 합의된 위험 / gap (다수 의견)

### A. v1 phase 게이트의 "3초" 정의 모호 (Claude, Gemini, Kimi)
"floating input에서 Agent 티켓을 만들고 3초 내 실행 큐 또는 running 상태로 전환된다"(Section 8 게이트, Section 5 Phase 1)가 측정 가능해 보이지만 실제로는 worktree 생성·instruction 주입·`post_worktree` 스크립트 실행이 포함되는지 명시되지 않음. 6장 리스크에서 "worktree 초기화 지연이 3초 기준 실패 원인"이라 자인한 셈. 큰 repo는 git worktree만으로 3초 초과 가능.

### B. tmux + portable-pty 이중 capture / 단일화 결정 부재 (Gemini, Kimi, DeepSeek)
3.7절은 "`tmux pipe-pane` 또는 PTY capture"이라 모호하게 표현. 두 메커니즘을 동시에 사용하면 race condition · 중복 기록 · 순서 뒤섞임 발생 가능. Phase 0에서 "검증한다"고만 적혀 있고, 실패 시 어느 쪽을 primary로 삼을지 결정 기준이 없음.

### C. Watchdog 구현 상세 부재 (Gemini, Kimi, DeepSeek)
어느 레이어(Rust core / OS 프로세스 / 분리된 데몬), 어떤 메트릭(`getrusage` / `sysctl` / `ps`), per-session vs system-wide 합산, watchdog 자체 crash 시 재시작 정책 모두 미정. Kimi는 추가로 "failed 후 cooldown 없이 즉시 dequeue하면 연쇄 실패"를 지적.

### D. Conflict policy "last-write-wins + conflict notification"의 추상성 (Claude, Gemini, DeepSeek)
어떻게 충돌을 감지하는지(timestamp / vector clock / `sync_version`?), 무엇을 사용자에게 보여주는지(diff / side-by-side / toast?), 덮어쓴 데이터를 복구할 수 있는지(이전 버전 보존?) 모두 미정. DeepSeek는 **libSQL Embedded Replica가 last-write-wins를 기본 제공하지 않으며, write는 primary로 proxy되고 충돌 resolution은 app layer 구현**임을 명시.

### E. AgentSession.state ↔ Ticket.status 매핑과 CAS 시맨틱 부재 (Kimi, DeepSeek)
`worktree_preparing`/`starting` 동안 Ticket.status는 무엇인지, `Ticket.status = todo`는 언제 쓰는지, `archived` 전이 조건은 무엇인지 미정. Kimi는 명시적 매핑 표를 제안. DeepSeek는 `queued → worktree_preparing`, `starting → running`, `any → orphaned` 각각에 CAS 프리컨디션이 빠져 multi-agent 큐 + 사용자 drag/drop + crash recovery 동시 발생 시 DB state가 모순될 위험을 지적.

### F. yolo mode + worktree 격리만의 안전성, Phase 4까지 권한 이연 (Claude, DeepSeek)
worktree 격리는 파일시스템만 다룬다. `git push --force origin main`, `curl -X POST production`, `rm -rf ~/`, `aws s3 rm` 등은 격리되지 않음. Phase 1~3 기간(dogfood ~ sync) 동안 유일한 안전장치가 "사용자의 사후 diff 검토"인데, 이는 side effect가 이미 발생한 후. **v1에 최소 denylist가 필요**하다는 데 두 명 합의 (단, 강도는 다름 — 후술).

### G. Secrets / env 정책 디폴트의 broken-by-default UX (DeepSeek 강조, Claude 간접)
"기본 자동 복사 안 함"은 보안적으로 정답이지만, 대부분의 실제 repo는 `.env`/`ANTHROPIC_API_KEY` 없이 동작 불가. 첫 Agent 티켓이 broken state로 실패하면 사용자는 workspace 설정을 들여다보지 않고 이탈. workspace 생성 시 guided setup 또는 명시적 env 정책 선택 UI가 Phase 1에 필요.

### H. Hook 처리의 edge case 부재 (Kimi 강조, Claude 보조)
부분 성공(hook이 `done` 후 exit code != 0 → UI 깜빡임), 형식 검증(에이전트 문서 작성 중 우연한 명령 매칭), 타임아웃/debounce, 시작·종료 마커가 모두 미정. Claude는 Phase 0에 "Claude Code hook round-trip PoC"를 추가하라 제안.

### I. 로그 회전 / 압축 / 디스크 관리 정책 부재 (Kimi, DeepSeek)
raw log append가 무제한 누적되면 디스크 고갈. 회전·압축·자동 삭제 정책이 plan 어디에도 없음. DeepSeek는 추가로 SQLite SessionEvent 테이블에 raw log를 넣을 때의 WAL/락 contention(`SQLITE_BUSY` retry 전략 없음, DB 크기 수백 MB 도달 가능)을 강하게 지적.

### J. v1 Dogfood 검증 기준의 정량성 부족 (Claude 단독, 매우 강함)
Section 8의 10번 항목 "기존 단일 터미널 워크플로우보다 작업 추적이 개선됐다고 판단된다"는 self-report라 ship/no-ship 판정에 쓸 수 없음. raw log 유실 0건, orphaned 복구 성공률, p50 첫 stdout 출력 시간 같은 정량 지표가 빠짐.

### K. 기타 단일 의견 (참고용)
- **전역 단축키 `Cmd+Shift+Space` 충돌과 모드 토글 `Shift+Tab` ↔ 터미널 오버레이 입력 컨텍스트 충돌** (Claude). 첫 실행 단계에서 막히면 dogfood가 시작되지 않음.
- **MCP 재검토 트리거 명시 부재** (Claude). 어떤 신호가 보이면 Skill 패키지에서 MCP로 다시 검토할지 결정 지점이 닫혀 있지 않음.
- **CLI 4종 동시 지원의 Phase 2 부담** (Claude). OpenCode는 다른 3개 대비 표준화 정도가 낮음. Phase 2 후반 분리 또는 별도 phase로 이동 제안.
- **Session replay의 데이터 출처 (raw log != asciicast)** (DeepSeek). Phase 4의 replay는 ANSI escape + timing 정보가 필요해 단순 raw log append로는 불가능. Phase 0에 capture format 검토 항목 추가 필요.
- **Phase 게이트 회귀 규칙 부재** (Claude). 다음 phase에서 이전 phase 게이트 항목이 회귀하면 어떻게 대응할지 규칙이 없음.

## 모델 간 의견이 갈리는 지점 (가장 중요 — 사용자 결정 필요)

### 1. tmux를 hard dependency로 둘 것인가, optional fallback을 만들 것인가?
- **Kimi**: tmux를 v1 필수 의존성으로 못박고, `tmux pipe-pane`을 primary log capture로 단일화. portable-pty는 Phase 0 PoC용으로만. 향후 Windows 지원 등이 필요할 때만 portable-pty adapter를 추가.
- **DeepSeek**: 반대로 macOS에서 tmux는 기본 미설치, homebrew 없는 환경/기업용 macOS 고려하면 tmux는 진입 장벽. `portable-pty`만으로 detached session 구현하는 fallback path를 Phase 1 scope에 포함.
- **Gemini**: tmux를 lifecycle 관리용 primary로, log capture는 `pipe-pane` → 파일 → Rust core가 tailing으로 일원화. portable-pty와 역할을 명시적으로 분리.
- **Claude**: 단축키/dependency check 책임 소재가 누락됐다는 일반론만 지적, tmux vs portable-pty 우선순위는 직접 다루지 않음.

**결정 포인트**: tmux가 안전·간단·검증된 경로지만 macOS dogfood 첫 분의 1순위 실패 원인이 될 수 있다. 사용자가 직접 brew 설치를 안내받는 것을 v1 UX로 수용할지, 또는 portable-pty fallback path를 v1부터 마련할지.

### 2. 권한 모델을 어느 phase에 어디까지 도입할 것인가?
- **DeepSeek**: v1에 3개 최소 denylist 필수 — `network: allow/deny`, `git.push: allow/deny`, `file.rm_outside_worktree: allow/deny`. 위반 시 ticket을 `waiting`으로 변경 + OS 알림. 완전한 권한 모델은 Phase 4까지 이연하더라도 기본 deny 일부는 v1.
- **Claude**: 더 가벼운 안전장치만 — "worktree 안에서만 yolo 허용, repo root에서 yolo 불가" + "yolo 세션의 git status 변경 파일 수를 ticket 상세에 표시". 권한 정책 자체는 Phase 4 유지.
- **Kimi**: yolo 기본값을 합리적이라 평가, 권한 모델 v1 도입은 제안 안 함.
- **Gemini**: 권한 정책에 대해 직접 언급 안 함.

**결정 포인트**: dogfood 안전성을 어디까지 보수적으로 잡을지. 개인용 + 본인이 사용자라는 점에서 어느 정도까지 책임을 사용자에게 위임할 수 있는지.

### 3. Raw log 저장을 SessionEvent 테이블에 둘 것인가, 별도 append-only 파일/DB로 분리할 것인가?
- **DeepSeek (강함)**: SQLite WAL 모드에서도 동시 writer는 직렬화됨. N=3 Agent가 초당 수백 라인 stdout을 동일 .db에 insert하면 `SQLITE_BUSY` 발생 + retry 전략 없음. `.agentflow/logs/<session-id>.log` append-only 파일 또는 `ATTACH DATABASE`로 `.log.db` 분리.
- **Kimi**: 로그 회전/압축 정책 부재를 지적했지만 저장 위치 자체는 SessionEvent 테이블 유지에 큰 이의 없음.
- **Gemini, Claude**: 직접 언급 안 함.

**결정 포인트**: Phase 0 spike 항목으로 SessionEvent vs 파일 분리를 측정해 결정할지, plan 단계에서 분리를 못박을지.

### 4. CLI 4종 동시 지원의 Phase 분리 방식
- **Claude (강함)**: Phase 1은 Claude Code only, Phase 2 전반은 multi-agent runtime(Claude Code 기반), Phase 2 후반(또는 Phase 2.5 신설)에 Code CLI / Gemini CLI 추가, OpenCode는 capability registry 안정화 후 Phase 4 또는 별도 spike. Phase 2 통과 기준에 "비-Claude CLI 1개 이상이 capability registry로 동작" 추가.
- **Kimi**: capability registry가 Phase 2에 등장하지만 `agentflow` CLI 인터페이스 변경 시 Skill 패키지 호환성 문제를 우려 — 프로토콜 버전 관리(`AGENTFLOW_PROTOCOL_VERSION`) 제안.
- **Gemini, DeepSeek**: 4종 동시 지원의 Phase 부담을 직접 다루지 않음.

**결정 포인트**: Phase 2를 "multi-agent runtime"으로 묶을 것인가, "multi-agent runtime + multi-CLI"로 묶을 것인가. 후자는 Phase 2 자체를 비대화시킬 위험.

## 권장 후속 조치

### v1 진입 전 plan 본문에 반드시 반영 (4명 이상이 강하게 지적한 항목)
- [ ] **3.3 / 8장**: "3초"의 측정 정의 명시 — submit → queued/`worktree_preparing`까지 3초로 한정, worktree 생성·instruction 주입은 별도 substate로 표시. v1 게이트 5번 항목 재작성.
- [ ] **3.7 / Phase 0 게이트**: log capture 단일화 결정을 Phase 0 통과 기준에 포함 — primary는 무엇이고, 중복 방지는 어떻게 하는지, fallback은 무엇인지.
- [ ] **3.3 / 4.3**: AgentSession.state ↔ Ticket.status 매핑 표 + 각 전이의 CAS 프리컨디션 명시. orphaned 판정 알고리즘(`tmux ls` × `AgentSession.state` × `pid` 조합 매트릭스) 구체화.
- [ ] **Phase 2 / 4.1**: watchdog 아키텍처 구체화 — 어느 레이어, 어떤 메트릭, system-wide 합산 체크 포함 여부, failed 후 cooldown 정책.
- [ ] **Phase 3 / Section 4.2**: conflict policy를 추상에서 UI 명세로 — libSQL Embedded Replica의 실제 동작에 맞춰 충돌 감지 메커니즘(`sync_version` 낙관적 락)과 UI 표시 방식(side-by-side conflict toast) 한 줄 명세.
- [ ] **8장**: v1 dogfood 검증 기준에 정량 항목 3개 추가 — raw log 유실 0건, orphaned 복구 성공률, Agent submit → 첫 stdout p50 시간. self-report 항목은 보조 지표로 격하.
- [ ] **Phase 1 / 3.6**: workspace 생성 시 env 정책 guided setup UI를 Phase 1 scope에 포함. broken-by-default 시나리오 방지.
- [ ] **3.3 / 5장**: 로그 회전·압축·자동 삭제 정책 추가 (세션당 크기 한계, 완료 세션 압축 시점, 삭제 시점).
- [ ] **3.5 / 6장**: hook 처리 규칙 — exit code 우선 명시, 시작·종료 마커, debounce 간격, 형식 검증.

### 사용자 결정 필요 (모델 간 의견 갈림)
- [ ] **tmux dependency**: hard dependency + brew 설치 안내 vs portable-pty fallback path 마련. (위 disagreement #1)
- [ ] **v1 권한 모델 강도**: DeepSeek 안(3개 denylist) vs Claude 안(worktree 한정 yolo + diff stat) vs 현 plan 유지(Phase 4까지 이연). (위 disagreement #2)
- [ ] **Raw log 저장 위치**: SessionEvent 테이블 유지 vs 별도 파일/DB 분리. (위 disagreement #3)
- [ ] **CLI 4종 Phase 배치**: 현 plan(Phase 2 일괄) vs Claude 안(Phase 1 Claude Code only, Phase 2.5 비-Claude). (위 disagreement #4)

### 권장 보강 (단일 의견이지만 강함)
- [ ] **Phase 0 게이트**: 전역 단축키 충돌 검증 + fallback 단축키 제시 (Claude).
- [ ] **3.1**: 터미널 오버레이 활성 시 `Shift+Tab` PTY 우선 규칙 (Claude).
- [ ] **3.5**: MCP 재검토 트리거 명시 — 어떤 신호가 보이면 Skill → MCP 전환을 검토할지 (Claude).
- [ ] **3.6**: worktree에서 git remote 조작 격리 — `git config --local` 또는 push URL 빈 값 설정 (DeepSeek).
- [ ] **Phase 0 또는 3.7**: session replay를 위한 terminal recording format(asciicast v2 등) 검토 (DeepSeek).
- [ ] **Phase 0**: Claude Code hook round-trip PoC 추가 (Claude).
- [ ] **5장**: phase 게이트 회귀 규칙 한 단락 추가 (Claude).
- [ ] **4.2**: Ticket/Comment에 `sync_origin` (device_id) 컬럼 추가 (Claude).
- [ ] **3.5 / 4.1**: `agentflow` CLI Unix domain socket 경로 정책 + workspace 단위 격리 명시 (Claude, Kimi, DeepSeek 공통 약지적).

## 모델별 리뷰 원본
- [Claude Opus 4.7](./plan-claude.md)
- [Gemini 3.1 Pro Preview](./plan-gemini.md)
- [Kimi K2.6](./plan-kimi.md)
- [DeepSeek V4 Pro](./plan-deepseek.md)

<!-- council-flow:review-complete -->
