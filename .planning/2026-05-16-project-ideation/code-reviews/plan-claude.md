---
title: "Plan review — agentflow-project-ideation — claude-opus-4-7"
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
reviewer: claude-opus-4-7
cli: claude
verdict: ship-after-minor-edits
prompted_against:
  - /Users/jh/Codes/radial-app/.planning/2026-05-16-project-ideation/plan.md
---

## 강점 (Strengths)

- **데이터 모델과 UI 컬럼의 분리(3.2)**가 v2 대비 가장 큰 개선이다. `Ticket.status`/`Ticket.type`/`Ticket.assignee_type` 3축과 `AgentSession.state` 별도 상태머신은 칸반 lane이 늘어나거나 줄어도 데이터 모델이 흔들리지 않는다. "UI 컬럼은 데이터 status를 그대로 노출하지 않아도 된다"는 명시는 좋은 설계 의사결정이다.
- **`AgentSession.state`에 `orphaned`/`recovered`를 1급 상태로 포함**한 것은 local-first + tmux 조합의 가장 흔한 실패 모드(앱 강제종료 후 재시작)를 정면으로 다룬다. 4.3의 "tmux 세션은 있는데 DB state가 불명확하면 orphaned로 표시"는 구체적이고 측정 가능하다.
- **Phase 0 게이트(414)에서 "wterm 사용 여부를 결정한다. 실패 시 xterm.js를 v1 기본값으로 확정"** 이라고 단정한 것이 좋다. 대부분 plan은 fallback을 "준비한다" 수준에서 끝나지만, 이 plan은 spike 결과로 결정을 내리고 v1을 잠그는 지점이 명확하다.
- **"Raw log always wins" 원칙(44)이 3.3 파이프라인, 4.3 상태 전이, 6장 hook 파싱 실패 대응까지 일관되게 적용**된다. hook 실패만으로 세션을 failed 처리하지 않는다(344)는 규칙은 실제 dogfood에서 흔히 발생하는 false-failed를 막아준다.
- **SessionEvent와 raw log를 sync 대상에서 제외(338)**한 결정. 이는 Turso 비용/충돌/PII 문제를 한 번에 해결한다. Phase 3 게이트에서 "Agent raw log는 sync되지 않는다"가 검증 항목으로 박혀 있어 후속 결정 번복도 어렵다.
- **6장의 리스크 목록이 영향/대응 쌍으로 짜여 있고, 모두 plan 본문의 다른 섹션과 연결**된다. 예: wterm 위험 → Phase 0 게이트, hook 위험 → 4.3 상태 전이, secrets 위험 → 3.6.

## 위험 및 누락 (Gaps and risks)

- **"3초 내 실행 큐 또는 running 상태로 전환"(445)이 측정 가능한 듯 보이지만 정의가 모호하다.** worktree 생성, instruction 주입, `post_worktree`/`pre_agent` 스크립트 실행이 포함된 3초인지, 큐 진입만 3초인지 불명. 6장에서도 "worktree 초기화 지연이 3초 기준 실패의 원인"이라 적었는데, 이 시점에서 이미 3초 기준이 깨질 수 있음을 자인한 셈이다. v1 게이트 통과 여부가 측정 정의에 좌우된다.
- **8장 v1 Dogfood 검증 기준이 측정 가능하지 않은 항목을 포함**한다. 10번째 항목 "1주일 dogfood 후 기존 단일 터미널 워크플로우보다 작업 추적이 개선됐다고 판단된다"는 self-report 기반이라 ship/no-ship 결정에 쓰기 어렵다. "주당 만든 Agent 티켓 수", "orphaned 복구 성공률", "raw log 유실 0건" 등 정량 기준이 빠졌다.
- **4개 CLI 동시 지원(3.4)을 Phase 2에 묶은 것은 v1 부담은 줄였지만 Phase 2 자체를 비대화**시킨다. Phase 2 통과 기준(468)에 "Code CLI/Gemini CLI/OpenCode 추가" 자체에 대한 검증이 없다. 동시 3개 실행과 CLI Bridge 검증은 Claude Code 단일로도 가능하므로, 비-Claude CLI는 Phase 2 후반 또는 별도 phase로 분리하는 편이 안전하다. 특히 OpenCode는 다른 3개 대비 표준화 정도가 낮아 capability registry 추상화 비용이 비대칭적으로 클 위험이 있다.
- **Skill/Instruction 패키지가 MCP를 대체한다는 결정(197, 586)의 근거가 약하다.** "지원 대상 CLI들이 동일한 protocol/tooling 모델을 공유한다고 가정하기 어렵고"는 사실이지만, Claude Code와 Gemini CLI는 둘 다 MCP를 지원한다. v2+에서 양 CLI가 MCP를 표준 도구 인터페이스로 굳히면 Skill 접근은 다시 뒤집힐 가능성이 있다. v3에서 "MCP 도입 조건(어떤 신호가 보이면 다시 검토할지)"이 명시되어 있지 않다.
- **전역 단축키 충돌 처리가 누락**됐다. `Cmd+Shift+Space`는 macOS의 입력 소스 전환 기본 단축키와 충돌할 수 있다. Phase 0 통과 기준에 "단축키가 다른 시스템/앱 단축키와 충돌하지 않는지 확인" 항목이 없다. 사용자 변경은 가능(66)하지만 첫 실행에서 막히면 dogfood 자체가 시작되지 않는다.
- **모드 토글 UX(`Shift+Tab`, 74)가 Agent → Terminal 전환 흐름과 충돌**할 수 있다. 3.1 마지막 단락에서 "Agent 실행 시 입력창이 터미널 오버레이로 확장"된다고 했는데, 터미널 오버레이에서 `Shift+Tab`은 일반적으로 텍스트 인풋이다. 모드 토글과 터미널 입력의 입력 컨텍스트 분리 규칙이 plan에 없다.
- **`AGENTFLOW_SOCKET` 사용(202)이 etymology만 있고 어떤 transport인지 미정**이다. 4.1에서 "Unix domain socket"이라 했지만 multi-instance(여러 AgentFlow 인스턴스가 동시 실행) 또는 동일 ticket을 여러 agent가 참조하는 경우의 socket discovery/locking 정책이 빠져 있다.
- **Conflict policy "last-write-wins + conflict notification"(482)의 정의가 추상적이다.** 개인용 single-user 가정인데도 Web/Mobile에서 같은 ticket의 status를 동시에 변경하는 시나리오에서 "conflict notification"이 무엇을 보여주는지(diff? two-way? overwrite 후 toast?) 미정. Phase 3 통과 기준에 "충돌이 silent overwrite가 아님"만 있어 검증이 모호하다.
- **권한 모델 고도화(Phase 4)와 yolo mode 기본값(46) 사이의 긴장**. yolo는 v1부터 기본값인데 권한 모델은 v4로 미뤘다. v1~v2에서 yolo로 인한 사고가 발생하면 게이트가 뒤로 이동할 수 있는데, 이 risk가 6장 yolo 리스크 대응("workspace 단위 표시, worktree 격리, diff review")으로 충분히 덮이는지 의문이다. diff review는 Phase 4 작업이다.
- **dependency check(앱 시작 시 tmux/git 확인, 526)와 onboarding flow의 책임 소재 누락**. Phase 0의 "tmux 설치 확인"은 spike이고, Phase 1 범위에 "최초 실행 시 의존성 부재 처리"가 명시되지 않았다. v1 dogfood 첫 분의 실패 원인 1순위가 될 가능성이 높다.
- **데이터 모델에 user/identity 컬럼이 없다.** 개인용 + local-first 가정이라 단기적으로 문제는 없지만, Phase 3에서 Web/Mobile이 추가될 때 "어느 디바이스에서 만든 ticket"을 추적할 sync_origin 필드가 빠져 있다. Conflict resolution과 audit에서 곧바로 필요해진다.
- **Phase 0 "Claude Code 단일 spawn PoC"(407)에 hook 수신 검증이 빠져 있다.** 3.5의 CLI Bridge와 3.3의 hook 기반 상태 업데이트가 작동하려면 Phase 0에서 최소한의 hook round-trip 검증이 필요하다. 이게 없으면 Phase 1에서 raw log만 보이고 상태 전이가 모두 수동이 될 위험.
- **Phase 게이트 통과 후 "되돌아갈 조건"이 없다.** 예를 들어 Phase 2에서 동시 3개 실행이 안정화됐다고 통과했는데 Phase 3 sync 도입 후 회귀가 발생하면 어떻게 처리하는지(이전 phase 게이트 재실행) 규칙이 없다.

## 구체적 제안 (Concrete suggestions)

1. **3.3 Agent 실행 파이프라인에 "3초"의 측정 정의를 추가**. 예: "Agent 모드 submit → Ticket row 생성 + UI에 queued 또는 running 표시까지 3초 이내. worktree 생성과 instruction 주입은 이 3초에 포함되지 않으며 별도 'preparing' substate로 표시." 그리고 8장 5번 항목을 이 정의에 맞춰 재작성.
2. **8장 v1 Dogfood 검증 기준에 정량 항목 3개 추가**. (a) 1주일간 raw log 유실 0건, (b) 앱 강제 종료 후 orphaned 세션 복구 성공률 100%(샘플 5회 이상), (c) Agent 티켓 생성에서 첫 stdout 출력까지 p50 ≤ 10초. 10번째 항목(self-report)은 보조 지표로 격하.
3. **3.4 지원 CLI 우선순위 재조정**. Phase 1: Claude Code only. Phase 2 전반: 동시 실행/큐/watchdog/Skill 패키지(Claude Code 기반). Phase 2 후반(혹은 Phase 2.5 신설): Code CLI/Gemini CLI 추가. OpenCode는 capability registry 안정화 후 Phase 4 또는 별도 spike. Phase 2 통과 기준에 "비-Claude CLI 1개 이상이 capability registry로 동작" 항목 추가.
4. **3.5에 "MCP 재검토 트리거" 명시**. "Skill 패키지로는 표현 불가능한 양방향 도구 호출이 2회 이상 누적되거나, 지원 대상 CLI 중 2개 이상이 동일한 MCP profile을 stable로 표기하면 v2+ 시점에 MCP 도입을 재검토한다." 후속 의사결정 지점을 닫아준다.
5. **Phase 0 통과 기준에 단축키 충돌 검증 추가**. "기본값 `Cmd+Shift+Space`가 macOS Sonoma/Sequoia 기본 설정과 충돌하지 않거나, 충돌 시 즉시 fallback 단축키(예: `Cmd+Opt+Space`)를 제시한다."
6. **3.1 모드 토글 규칙 보강**. "터미널 오버레이가 활성화된 동안 `Shift+Tab`은 모드 토글로 가로채지 않고 PTY로 전달한다. 모드 토글은 오버레이 비활성 상태에서만 활성." 입력 컨텍스트 우선순위를 명시.
7. **4.1 Agent control 항목에 socket discovery/locking 정책 추가**. "Unix domain socket 경로는 `${XDG_RUNTIME_DIR:-$TMPDIR}/agentflow-<workspace-id>.sock`. 동시 실행 시 충돌하지 않도록 workspace 단위로 격리. multi-instance는 v1 비지원으로 명시."
8. **4.2 데이터 모델에 `sync_origin`(device_id) 컬럼 추가**. Ticket/Comment에 한정해 추가하고, AgentSession은 desktop-only이므로 생략. Phase 3 conflict UI의 기반.
9. **Phase 3 통과 기준에 conflict UI 정의 추가**. "동일 ticket의 동일 필드가 다른 device에서 갱신된 경우, last-write-wins 적용 전에 두 버전을 사이드바이사이드로 보여주는 conflict toast가 트리거된다. 사용자는 keep mine / accept theirs 중 선택한다." 추상 표현 대신 UI 명세 한 줄.
10. **Phase 1 범위에 "최초 실행 의존성 체크 + 가이드"를 명시**. tmux, git, 단축키 권한(Accessibility)을 onboarding에서 체크하고 missing 시 액션 버튼을 표시한다. 6장 "tmux 외부 의존성" 대응을 Phase 1으로 당겨야 v1 dogfood가 시작될 수 있다.
11. **Phase 0 spike에 "Claude Code hook round-trip PoC" 추가**. status update 1회를 hook → CLI Bridge → DB까지 끝까지 보내는 실험. 실패 시 Phase 1에서 상태 전이 전략을 polling 또는 watchdog 기반으로 조정.
12. **5장에 "게이트 회귀 규칙" 한 단락 추가**. "이후 phase에서 이전 phase 게이트 항목이 회귀하면 해당 phase 진행을 중단하고 회귀 항목을 우선 복구한다. 회귀 복구가 phase plan의 일부가 된다." 게이트의 unidirectional 가정 보강.
13. **6장 yolo 리스크 대응에 v1 한정 안전장치 추가**. diff review는 Phase 4이므로, v1에서는 최소한 "worktree 안에서만 yolo 허용, repo root에서 yolo 불가"와 "yolo 세션의 git status 변경 파일 수를 ticket 상세에 표시"를 의무화. 표면적 격리만으로 부족한 경우의 보강선.

## Verdict

ship after minor edits — v3는 v2 대비 데이터 모델 분리, phase 게이트, orphaned 상태, raw-log 보존 같은 핵심 결정 지점을 잘 닫았다. 다만 "3초"의 정의, dogfood 검증 기준의 정량화, CLI 우선순위, MCP 재검토 트리거, conflict UI 명세 등 측정·결정 지점에서 모호함이 남아 있어 plan 자체로는 일부 게이트가 통과 판정을 내리기 어렵다. 위 13개 제안 중 1, 2, 3, 9, 10번은 구현 전 plan에 반영해야 v1 dogfood가 의미 있는 검증이 된다.

<!-- council-flow:review-complete -->
