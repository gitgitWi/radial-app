---
title: "Plan review — agentflow-project-ideation — kimi-k2.6"
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
reviewer: opencode-go/kimi-k2.6
cli: opencode
verdict: ship-after-minor-edits
prompted_against:
  - /Users/jh/Codes/radial-app/.planning/2026-05-16-project-ideation/plan.md
---

## 강점 (Strengths)

**1. 상태 모델의 분리 원칙**
`Ticket.status`, `Ticket.assignee_type`, `AgentSession.state`를 계층별로 분리한 것은 데이터 모델 설계의 핵심 결정이다. UI 컬럼과 데이터 상태를 디커플링하면 칸반 뷰가 변경되어도 실행 파이프라인에 영향을 주지 않는다. 특히 "UI drag/drop은 Ticket.status를 변경한다. Agent runtime은 AgentSession.state를 변경한다"는 원칙은 경계가 명확하다.

**2. Raw log 불변성**
"hook 파싱이나 상태 추론이 실패핲도 원본 stdout/stderr와 session metadata는 남긴다"는 원칙은 디버깅과 감사 추적의 기반이 된다. 에이전트 도구가 실패하거나 버전이 변경되어도 사실(fact) 계층이 유지된다.

**3. Phase gate 구조**
기술 선택 → Desktop MVP → Multi-agent Runtime → Sync → Polish 순서에 검증 게이트를 배치한 것은 리스크 기반 우선순위 설정이다. 특히 wterm 실패 시 xterm.js fallback을 Phase 0에서 결정하고, Turso sync를 Phase 3까지 이연한 것은 local-first 전략과 일치한다.

**4. Skill/Instruction 패키지 접근**
MCP 서버를 조기 도입하지 않고 `agentflow` CLI + 환경변수 + instruction 파일로 대체한 결정은 개인용 local-first 앱의 디버깅성과 배포 복잡도 측면에서 타당하다. 에이전트 CLI들의 프로토콜 표준화가 아직 불확실한 상황에서 과도한 abstraction보다 실용적이다.

**5. yolo mode와 권한 모델의 초기 고려**
"yolo 실행은 기본값으로 두되, 위험 모드 표시와 workspace 단위 설정은 v1부터 둔다"는 접근은 dogfood 단계에서 사용자 실수를 방지하면서도 반복 입력 마찰을 줄인다.

---

## 위험 및 누락 (Gaps and risks)

**1. 상태 머신 간 매항 불명확 (Ticket ↔ AgentSession ↔ QueueItem)**

`AgentSession.state`에는 `worktree_preparing`, `starting`이 있지만, 이 상태일 때 `Ticket.status`는 무엇인지 명시되지 않았다. 예를 들어 worktree 생성 중에는 `Ticket.status`가 `running`인가, 아니면 여전히 `queued`인가? 다이어그램에서는 `worktree_preparing`이 큐 통과 직후에 위치하므로 사용자는 칸반에서 "running"으로 볼 것으로 예상되지만, 사실은 아직 프로세스가 시작되지 않았다.

또한 `QueueItem.state`의 열거값이 정의되지 않았다. `queued` 상태의 티켓이 큐에서 빠지면 QueueItem은 삭제되는가, 아니면 `done` 상태로 남는가? 만약 삭제된다면 동시 실행 제한이 해제된 시점의 이력이 사라진다.

`Ticket.status`의 `todo`는 언제 사용되는지도 불명확하다. Task 티켓은 `backlog`에서 바로 `done`으로 가는가? 아니면 `todo`를 거치는가?

**2. tmux + portable-pty 이중 capture의 동시성/유실 위험**

3.7절에서 "`tmux pipe-pane` 또는 PTY capture"으로 raw log를 append한다고 했는데, "또는(or)"이 아니라 두 메커니즘을 동시에 사용할 가능성이 있다. 특히 portable-pty로 PTY를 열고 그 안에서 tmux를 띄우는 구조라면:
- portable-pty는 에뮬레이터 수준에서 바이트 스트림을 캡처한다
- tmux pipe-pane은 tmux 내부에서 pane 출력을 파이프로 보낸다
- 둘 다 동일한 로그 파일에 append하면 race condition과 중복 기록, 순서 뒤섞임이 발생할 수 있다

계획은 Phase 0에서 "중복 문제"를 검증한다고 하지만, 검증 실패 시 어떤 메커니즘을 primary로 삼을지 결정 기준이 없다. 만약 portable-pty가 primary라면 tmux는 세션 복구용(lifecycle 관리)으로만 사용하고 pipe-pane은 사용하지 않아야 한다. 반대로 tmux가 primary라면 portable-pty는 왜 필요한지 의문이다.

또한 "raw stdout/stderr가 유실 없이 append된다"는 Phase 1 통과 기준인데, 파일 시스템 append의 원자성 보장, 디스크 부족 시 graceful degradation, 로그 파일 회전(rotation) 정책이 전혀 없다.

**3. CLI Bridge + Skill 패키지의 확장성과 버전 관리**

`agentflow` CLI의 명령 집합(`subtask add`, `status update`, `comment add`)은 v1 기준이지만, 향후 필드가 추가되거나 의미가 바뀌면 기존 Skill 패키지와의 호환성이 깨진다. Skill 패키지는 "각 CLI가 읽을 수 있는 instruction 파일 템플릿"을 포함한다고 하는데, 템플릿의 버전 관리 전략이 없다.

더 중요한 것은 "CLI별 동작 방식 차이" 리스크(6.3절)에서 언급한 capability registry가 Phase 2에나 등장한다는 점이다. v1에서는 Claude Code만 지원하므로 registry 없이도 가능하지만, v2에서 Code CLI/Gemini CLI/OpenCode를 추가할 때 `agentflow` CLI의 인터페이스가 달라지면 Skill 패키지도 함께 수정되어야 한다. 계획에는 Skill 패키지가 에이전트별로 분리되어 있는지, 아니면 단일 패키지로 관리되는지 명시되지 않았다.

또한 `AGENTFLOW_SOCKET`이 Unix domain socket이라면, 다중 에이전트 동시 실행 시 소켓 경로 충돌 방지(예: 티켓별 소켓 네이밍)와 소켓 파일 정리가 필요하다.

**4. Hook 파싱 실패 대응의 충분성**

"hook 실패만으로 세션을 실패 처리하지 않는다"는 원칙은 올바르지만, 다음 세 가지 edge case가 누락되었다:

- **부분적 hook 성공**: hook이 상태를 `done`으로 업데이트했지만, 이후 exit code가 0이 아닌 경우 어떤 상태가 최종 승자인가? exit code 우선이라면 hook 성공 후 잠시 `done`으로 표시되었다가 `failed`로 되돌아가는 UI 깜빡임이 발생한다.
- **hook 형식 검증**: 에이전트 출력에 `agentflow status update ...`와 유사한 문자열이 우연히 포함되면(예: 에이전트가 문서를 작성할 때) 의도하지 않은 상태 변경이 발생할 수 있다. hook의 시작/종료 마커나 체크섬 검증이 필요하다.
- **hook 타임아웃**: 무한 루프에 빠진 에이전트가 hook을 계속 출력하면 상태 머신이 과도하게 전이될 수 있다. hook 처리 간 최소 간격(debounce)이나 최대 처리 빈도 제한이 없다.

**5. 동시 실행 N=3과 watchdog/OS 알림의 미스매치**

N=3은 동시 실행 슬롯 수이지만, 메모리 watchdog은 개별 세션 단위로 동작하는 것으로 보인다. 다음 시나리오가 계획에서 다뤄지지 않았다:

- 3개 세션이 각각 메모리 임계치 미만이지만, **합산 메모리**로 시스템에 부담을 주는 경우. watchdog이 per-session이면 이를 감지하지 못한다.
- 메모리 watchdog이 임계치를 초과한 세션을 `failed` 처리할 때, 해당 슬롯이 즉시 해제되어 queued 세션이 실행되면 메모리 상황이 더 악화될 수 있다. cooldown period나 시스템 전체 메모리 체크 없이 바로 dequeue하면 연쇄 실패가 발생할 수 있다.
- OS 알림은 완료/실패/대기/메모리 임계치 4가지를 포함하지만, **queue에서 오래 대기 중인 세션**(`queued` 상태가 10분 이상 지속)에 대한 알림은 없다. 사용자는 티켓이 queued인지 모르고 기다릴 수 있다.

**6. 기타 누락**

- **worktree_preparing 실패 시 정리**: 3.3절 다이어그램에서는 worktree 준비 실패 시 `Ticket: failed`로 가지만, 이미 생성된 불완전한 worktree나 branch가 남는 경우 정리 정책이 없다. 누적되면 `.agentflow/worktrees/`가 쓰레기로 채워진다.
- **SessionEvent 순서 보장**: `SessionEvent`에는 `ts`(타임스탬프)가 있지만, 동일 밀리초 내 여러 이벤트의 순서를 보장할 `sequence` 번호나 `local_event_id`의 정확한 생성 전략이 없다. 특히 앱 크래시 후 이벤트 재생 시 순서가 중요하다.
- **압축/회전 없는 무한 로그 증가**: raw log append가 무제한으로 누적될 경우 디스크 공간 고갈 리스크가 있다. 계획 어디에도 로그 회전, 압축, 자동 삭제 정책이 없다.
- **Phase 1 통과 기준의 비현실성**: "3초 내 실행 큐 또는 running 상태로 전환"은 worktree 생성 + branch 생성 + tmux 세션 생성 + 초기화 스크립트 실행을 포함한다. worktree가 큰 repo라면 git 작업만으로 3초를 초과할 수 있다. 이 기준은 실패 가능성이 높으므로 "3초 내 queued 상태"와 "실행 시작 시 queued → running"을 분리하여 기술해야 한다.
- **Ticket.status = `archived`의 전이 규칙**: `done`에서 `archived`로 가는 조건(수동? 자동? 일정 기간 후?)이 없다.

---

## 구체적 제안 (Concrete suggestions)

**1. 상태 매항 표 추가 (섹션 3.2 또는 4.3)**

`AgentSession.state` → `Ticket.status` → UI 표시의 명시적 매항 표를 추가하라.

| AgentSession.state | Ticket.status | UI 표시 |
|---|---|---|
| created | backlog | Backlog |
| worktree_preparing | queued | Agent lane — preparing |
| queued | queued | Agent lane — queued |
| starting | running | Agent lane — running |
| running | running | Agent lane — running |
| waiting_input | waiting | Agent lane — waiting |
| exited (code 0) | done | Done |
| exited (code != 0) | failed | Attention lane |
| failed | failed | Attention lane |
| orphaned | (이전 상태 유지) | Attention lane — orphaned |
| recovered | running | Agent lane — recovered |

이 표가 없으면 Phase 1 구현 시 매번 추론해야 하고, UI와 데이터 불일치 버그가 발생한다.

**2. QueueItem 상태 열거값 정의 및 수명주기 명시 (섹션 4.2)**

```text
QueueItem
  id, ticket_id, workspace_id, priority
  state: pending, active, completed, cancelled
  enqueued_at, started_at, completed_at
```

- `pending`: 슬롯 대기 중
- `active`: 실행 시작됨 (dequeue된 시점)
- `completed`: 정상 종료 또는 실패로 인해 큐에서 제거됨
- `cancelled`: 사용자가 큐에서 제거

QueueItem은 실행 완료 후에도 `completed` 상태로 24시간 보관하여 디버깅과 throughput 측정에 사용한다.

**3. Log capture 전략 단일화 및 fallback 명시 (섹션 3.7)**

다음과 같이 명확히 결정하라:

> **Primary**: tmux `pipe-pane`으로 세션별 로그 파일에 append. 이 메커니즘은 tmux 세션이 살아있는 한 로그가 유실되지 않는다.
> **Secondary (fallback)**: 앱이 pipe-pane 출력 파일을 tail -f하거나 inotify/kqueue로 모니터링하여 UI에 스트리밍한다. portable-pty는 tmux가 설치되지 않은 환경의 Phase 0 PoC용으로만 사용하고, v1에서는 tmux를 필수 의존성으로 한다.
> **중복 방지**: 두 메커니즘을 동시에 운영하지 않는다. 만약 portable-pty가 필요하다면(향후 Windows 지원 등) 로그 작성자를 단일화하는 adapter를 먼저 설계한다.

**4. Hook 처리 규칙 강화 (섹션 4.3 또는 신규 섹션)**

다음 규칙을 추가하라:

- **exit code 우선**: hook이 어떤 상태를 선언했든, 프로세스 종료 시 exit code가 최종 승자다. exit code != 0이면 최종 상태는 무조건 `failed`.
- **hook 마커**: `agentflow` CLI 호출은 JSON Lines 형식으로 stdout에 출력하되, 시작 마커 `<<<AGENTFLOW_HOOK>>>`과 종료 마커 `<<<AGENTFLOW_HOOK_END>>>`로 감싼다. 마커 불일치 시 해당 줄은 무시한다.
- **debounce**: 동일 세션에서 1초 이내 중복 상태 변경은 마지막 것만 적용한다.
- **hook 무시 모드**: `--dry-run` 또는 `AGENTFLOW_HOOK_DRY_RUN=1` 환경변수로 에이전트가 hook 출력을 테스트할 수 있게 한다.

**5. N=3 + watchdog 통합 개선 (섹션 3.3 및 5.2)**

다음 동작을 명시하라:

- **시스템 전체 메모리 체크**: per-session watchdog 외에, 시스템 전체 사용 가능 메모리가 500MB 미만으로 떨어지면 새로운 dequeue를 일시 중단한다. 이미 실행 중인 세션은 영향을 받지 않는다.
- **failed 후 cooldown**: watchdog에 의해 세션이 종료되면, 해당 슬롯은 10초간 cooldown 상태가 되어 queued 세션이 즉시 메모리 부족 환경에 진입하지 않도록 한다.
- **Queue 대기 알림**: `queued` 상태가 5분 이상 지속되면 OS 알림("N개의 에이전트가 실행 대기 중입니다")을 선택적으로 보낸다.

**6. 로그 회전 정책 추가 (섹션 3.3 또는 6.x 리스크)**

신규 리스크로 추가하라:

> **raw log 무한 증가**: 장기 실행 에이전트의 로그가 디스크를 채울 수 있다.
> - 대응: 세션당 10MB 초과 시 자동 순환(`.log.1`, `.log.2`), 최대 5개 파일 보관. 완료된 세션의 로그는 30일 후 자동 압축(gzip), 90일 후 삭제. 정책은 workspace 설정으로 override 가능.

**7. Phase 1 통과 기준 수정 (섹션 5.1)**

> "floating input에서 Agent 티켓을 만들고 3초 내 실행 큐 또는 running 상태로 전환된다"

를 다음으로 수정하라:

> "floating input에서 Agent 티켓을 만들고 3초 내 `Ticket.status = queued` 또는 `AgentSession.state = worktree_preparing`으로 전환된다. 실제 `running` 상태는 worktree 준비 및 tmux 세션 생성 완료 후 도달하며, 이는 repo 크기에 따라 변할 수 있다."

**8. `agentflow` CLI 프로토콜 버전 관리 (섹션 3.5)**

Skill 패키지 문서에 다음을 추가하라:

> `agentflow` CLI는 `AGENTFLOW_PROTOCOL_VERSION` 환경변수를 읽는다. v1 명령 집합은 하위 호환성을 보장하며, v2 필드 추가 시에도 v1 클라이언트는 무시할 수 있는 선택적 필드만 사용한다. Breaking change가 필요한 경우, Skill 패키지 메이저 버전을 함께 올린다.

---

## Verdict

ship after minor edits — 기술 방향성과 phase 구조는 견고하지만, 상태 머신 매항과 로그 capture 단일화 결정은 Phase 0~1 구현 전에 반드시 문서화되어야 한다. 위 제안 1~5를 계획에 반영하면 리스크가 크게 줄어든다.

<!-- council-flow:review-complete -->