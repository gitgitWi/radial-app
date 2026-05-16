---
title: "Plan review — agentflow-project-ideation — gemini-3.1-pro-preview"
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
reviewer: gemini-3.1-pro-preview
cli: gemini
verdict: ship-after-minor-edits
prompted_against:
  - /Users/jh/Codes/radial-app/.planning/2026-05-16-project-ideation/plan.md
---

## 강점 (Strengths)
- **명확한 Phase Gate**: 각 Phase별 진입 및 통과 기준(Dogfood 검증 등)이 구체적이고 측정 가능하게 정의되어 있어 프로젝트 진행의 기준점이 명확함.
- **실용적인 점진적 설계**: 복잡도가 높은 동기화(Turso/libSQL)를 Phase 3으로 미루고, Local-first(SQLite) 데스크톱 환경 구축을 우선한 점은 리스크 관리에 탁월함.
- **견고한 데이터/세션 분리**: Ticket의 상태(status)와 AgentSession의 상태(state)를 분리하여 칸반 UI의 직관성과 시스템 백그라운드 관리의 정확성을 모두 확보함.
- **Fail-safe 원칙**: "Raw log always wins" 기조 아래, hook이나 상태 추론 실패 시에도 tmux 세션과 원본 로그를 남기도록 한 설계는 에이전트 실행의 불확실성을 잘 대비함.

## 위험 및 누락 (Gaps and risks)
- **Tauri + tmux + PTY 캡처 구조의 복잡성 (3.7 터미널/PTY 전략)**: Tauri v2, wterm(혹은 xterm.js), tmux attach, `portable-pty`가 맞물리는 구조는 입력 스트림 충돌 및 제어권 상실 위험이 큼. 특히 tmux와 `portable-pty`가 동시에 stdout을 캡처하려 할 때 퍼포먼스 저하와 로그 꼬임이 발생할 가능성이 높음.
- **Worktree 초기화 지연으로 인한 V1 게이트 실패 위험 (3.6 Worktree와 workspace 준비)**: `post_worktree` 스크립트 실행(예: `npm install` 등)이 길어질 경우, "3초 내 실행 큐 또는 running 상태 전환"이라는 V1 Dogfood 통과 기준(8. v1 Dogfood 검증 기준)을 만족시키기 불가능해 보임.
- **Watchdog 구현의 모호성 (3.3 Agent 실행 파이프라인)**: 메모리 초과나 런타임 행(hang)을 감지할 Watchdog이 구체적으로 어느 레이어(Rust Core, OS 프로세스 레벨, 혹은 분리된 데몬)에서 동작하고 판정하는지 명시되지 않음.
- **분산 환경의 낙관적 동기화 정책 (4.2 데이터 모델 보강)**: last-write-wins 정책 하나만으로는 오프라인 상태에서 길게 진행된 AgentSession이나 복잡한 Ticket 전이 상태가 병합될 때 논리적 모순이 발생할 수 있음.

## 구체적 제안 (Concrete suggestions)
- **PTY 캡처 및 터미널 렌더링 파이프라인 일원화 (3.7 섹션)**: `portable-pty`와 tmux의 역할을 겹치지 않게 분리. 런타임 로그는 전적으로 `tmux pipe-pane`을 통해 파일로 내보내고, Rust Core가 이 파일을 tailing하여 UI 프론트엔드로 전달하는 방식을 제안. 
- **Worktree 종속성 캐싱 전략 명시 (3.6 섹션)**: Worktree 생성 시 단순히 브랜치만 따는 것이 아니라, 빠른 초기화를 위해 `node_modules` 등의 글로벌 캐싱 계층 참조 혹은 심링크(symlink) 사용 전략을 문서에 추가하여 3초 제한 목표를 현실화.
- **Watchdog 아키텍처 구체화 (4.1 레이어)**: Backend core(Rust) 레벨에서 주기적으로 `ps` 또는 OS 시스템 콜을 이용해 Agent CLI의 PID 트리 리소스를 수집하고 평가하는 구조임을 레이어 명세에 추가.
- **Phase 1 게이트 조건 현실화 (8. 섹션)**: "3초 내 실행 큐 또는 running 상태로 전환된다"에서 초기화(Preparing) 시간을 예외로 두거나, Queue 진입까지만 3초 내로 제한하도록 조건을 수정.

## Verdict
ship after minor edits: 전반적인 아키텍처와 실행 계획은 매우 견고하나, Tauri/tmux/PTY 통합 파이프라인의 명확화와 Worktree 초기화 시간(3초 제한 조건)에 대한 캐싱/게이트 수정이 V1 성공을 위해 선행되어야 함.
<!-- council-flow:review-complete -->