# SSH

원격 머신에 **암호화된 채널로 접속**하는 프로토콜. 단순히 "원격 로그인"이라기보다, 핵심은 *비밀번호 없이도 안전하게 신원을 증명하는 방식*에 있다 — 그래서 공개키 인증의 작동 원리를 이해하는 게 SSH를 이해하는 것과 거의 같다.

## 왜 공개키인가 — 비대칭 키의 핵심

비밀번호 인증은 "공유 비밀"을 매번 네트워크로 보낸다. 가로채이면 끝. SSH 공개키 인증은 **비밀을 절대 보내지 않는다**는 게 발상의 전환이다.

키는 한 쌍으로 만들어진다:

- **private key** (`~/.ssh/id_ed25519`) — 내 손을 절대 떠나지 않음. 이게 곧 나.
- **public key** (`~/.ssh/id_ed25519.pub`) — 서버에 미리 올려둠 (`~/.ssh/authorized_keys`). 공개돼도 안전.

접속 흐름:

1. 서버가 "이 랜덤 데이터에 네 private key로 서명해봐"라고 던진다 (challenge).
2. 클라이언트가 private key로 서명해서 보낸다.
3. 서버는 올려둔 public key로 **서명이 맞는지 검증만** 한다.

private key는 한 번도 전송되지 않는다. public key로는 서명을 *검증*만 할 수 있을 뿐 *위조*할 수 없다 — 이 비대칭성이 보안의 뿌리다. 비밀번호 방식과 가장 크게 갈리는 지점.

## ssh-agent — private key를 매번 풀지 않으려고

private key는 보통 passphrase로 암호화돼 있다. 매 접속마다 passphrase를 치기 싫으니, **ssh-agent**가 메모리에 복호화된 키를 들고 있다가 대신 서명한다. `ssh-add`로 키를 agent에 등록.

## known_hosts — 서버를 의심하는 쪽

위는 "서버가 나를 믿는" 방향. 반대로 **나도 서버를 믿어야** 중간자 공격(MITM)을 막는다. 처음 접속 시 뜨는

```
The authenticity of host '...' can't be established. ... (yes/no)?
```

이게 서버의 호스트 키 지문을 `~/.ssh/known_hosts`에 기록할지 묻는 것. 다음부터 지문이 바뀌면 경고한다 — 누가 그 주소를 가로챘다는 신호일 수 있으니.

## ~/.ssh/config — 반복을 줄이는 곳

호스트별 설정을 별칭으로 묶어둔다.

```
Host myserver
    HostName 203.0.113.10
    User ijaehun
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

이러면 `ssh myserver` 한 줄로 끝. 긴 옵션을 외울 필요가 없다.

## 자주 쓰는 명령

| 명령 | 동작 |
|---|---|
| `ssh-keygen -t ed25519` | 키 쌍 생성 (ed25519 권장, RSA보다 짧고 안전) |
| `ssh-copy-id user@host` | public key를 서버 `authorized_keys`에 등록 |
| `ssh user@host` | 접속 |
| `ssh-add ~/.ssh/id_ed25519` | agent에 key 등록 |
| `scp 파일 user@host:경로` | 파일 복사 |

## tmux와의 관계

SSH 세션은 연결이 끊기면 그 안에서 돌던 프로세스도 같이 죽는다. 그래서 원격에서 긴 작업을 돌릴 땐 접속하자마자 [[tmux 단축키|tmux]]로 세션을 띄우고 그 안에서 작업한다 — SSH가 끊겨도 tmux 세션은 서버에 살아 있어, 재접속 후 `attach`로 그대로 복귀할 수 있다. 둘이 자주 짝지어 쓰이는 이유.

## 곁가지 — git 인증

GitHub 등에 SSH 키를 등록하면 `git push`마다 토큰을 치지 않아도 된다. 다만 **기기마다 key가 따로**라, 새 기기(예: 다른 mac)에서 clone하려면 그 기기에서 키를 새로 만들어 등록해야 한다.
