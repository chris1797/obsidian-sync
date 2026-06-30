# tmux 단축키

터미널 하나에서 여러 작업 공간을 띄우고, **접속을 끊어도 프로세스가 살아 있게** 하는 도구. 핵심 가치는 detach — SSH가 끊겨도 세션 안의 작업은 계속 돈다.

## 멘탈 모델: 중첩된 그릇

```
session (작업 공간 전체, 예: "프로젝트A")
└─ window (탭 같은 것)
   └─ pane (한 화면을 쪼갠 분할)
```

session > window > pane 순으로 큰 그릇 안에 작은 그릇이 들어간다. 이 위계를 잡고 있으면 단축키가 어느 층을 건드리는지 헷갈리지 않는다.

## prefix

거의 모든 단축키는 **prefix를 먼저 누르고 뗀 뒤** 다음 키를 누른다. 기본 prefix는 `Ctrl+b`.
아래 표에서 `prefix %` = `Ctrl+b` 누르고 떼고 → `%`.

## Session (작업 공간)

| 키 / 명령 | 동작 |
|---|---|
| `tmux new -s 이름` | 새 세션 생성 |
| `tmux ls` | 세션 목록 |
| `tmux attach -t 이름` | 기존 세션에 붙기 |
| `prefix d` | **detach** — 세션은 살려두고 빠져나옴 (tmux의 핵심) |
| `prefix s` | 세션 목록에서 골라 전환 |
| `prefix $` | 현재 세션 이름 변경 |
| `tmux kill-session -t 이름` | 세션 종료 |

## Window (탭)

| 키 | 동작 |
|---|---|
| `prefix c` | 새 윈도우 |
| `prefix ,` | 윈도우 이름 변경 |
| `prefix n` / `prefix p` | 다음 / 이전 윈도우 |
| `prefix 0~9` | 번호로 이동 |
| `prefix w` | 윈도우 목록에서 선택 |
| `prefix &` | 윈도우 닫기 |

## Pane (화면 분할)

| 키 | 동작 |
|---|---|
| `prefix %` | 세로 분할 (좌우로 쪼갬) |
| `prefix "` | 가로 분할 (위아래로 쪼갬) |
| `prefix 화살표` | 분할된 pane 간 이동 |
| `prefix o` | 다음 pane으로 순환 |
| `prefix z` | **zoom** — 현재 pane만 전체화면 토글 |
| `prefix {` / `prefix }` | pane 위치 swap |
| `prefix space` | 레이아웃 순환 변경 |
| `prefix x` | pane 닫기 |

## 자주 쓰는 흐름

- 작업 시작: `tmux new -s 프로젝트` → 안에서 `prefix %`로 좌우 분할(에디터 / 로그)
- 자리 비울 때: `prefix d`로 detach → 나중에 `tmux attach -t 프로젝트`로 그대로 복귀
- prefix가 손에 안 맞으면 `~/.tmux.conf`에서 `Ctrl+a` 등으로 재바인딩하는 게 흔한 첫 커스터마이징
