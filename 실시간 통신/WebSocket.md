---
up: "[[실시간 전송 방식 선택]]"
---
# WebSocket

**HTTP로 시작해서 HTTP를 버리는** 프로토콜. 다른 두 방식([[Long Polling]]·[[SSE]])이 끝까지 HTTP 안에 머무는 것과 근본적으로 다른 지점이 여기다. HTTP를 사다리로 쓰고, 올라간 뒤 걷어찬다.

그래서 얻는 게 둘이다 — **양방향**(클라이언트도 말할 수 있음)과 **2바이트 오버헤드**(헤더가 없으니까). 잃는 것도 둘인데, 그게 이 노트의 후반부다.

## 핸드셰이크 — 101 Switching Protocols

딱 한 번 HTTP를 쓴다.

```
→ GET /ws HTTP/1.1
  Host: example.com
  Upgrade: websocket                            ← ① 갈아타자
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==   ← ② 무작위 16바이트의 base64
  Sec-WebSocket-Version: 13
  Sec-WebSocket-Protocol: trades.v1             ← ③ 선택: 앱 레벨 프로토콜 협상

← HTTP/1.1 101 Switching Protocols              ← ④ 200 이 아니다
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

④ 이후로는 **같은 TCP 소켓 위에서 완전히 다른 규칙**으로 통신한다. 상태 코드도 헤더도 없다.

②→`Sec-WebSocket-Accept` 계산이 처음엔 보안 장치인 줄 알았는데 아니었다.

```js
crypto.createHash('sha1')
  .update(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11')  // RFC 6455 고정 GUID
  .digest('base64');
```

GUID가 규격에 박혀 있는 공개 상수다. 비밀이 없으니 인증일 리 없다. 실제 목적은 **"나는 진짜 WebSocket 서버지, 우연히 101을 뱉은 캐시나 프록시가 아니다"를 증명**하는 것이다. 중간 장비가 이 응답을 오해해서 캐싱하거나 잘못 중계하는 걸 막는다. 경계에 낀 중간 장비가 프로토콜을 오해한다는 문제의식은 [[네트워크 경계 (ingress·egress·NAT)]]의 프록시 이야기와 같은 뿌리다.

## 프레이밍 — 오버헤드 2바이트의 정체

```
바이트 0:  [FIN=1][RSV 000][opcode=0001]   ← 0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong
바이트 1:  [MASK=0][payload len=117]        ← 125 이하면 여기서 끝
바이트 2~: 페이로드 117바이트
```

**117바이트 본문 + 헤더 2바이트 = 119바이트.** 1,000건이면 119,000바이트고, 실측값 119,692에 핸드셰이크까지 딱 들어맞는다. 계산이 맞아떨어지는 걸 보고서야 "오버헤드 2바이트"가 비유가 아니라 문자 그대로였다는 게 실감났다.

규칙 둘이 더 있다.

- 페이로드가 126~65535면 길이 필드 2바이트 추가, 그 이상이면 8바이트 추가.
- **클라이언트→서버 프레임은 반드시 마스킹**(4바이트 키로 XOR). 올려보내는 쪽은 항상 +4바이트다. 이것도 보안이 아니라 **프록시 캐시 오염 방지**용이다 — 공격자가 페이로드에 가짜 HTTP 요청처럼 보이는 바이트열을 넣어 중간 캐시를 속이는 걸 막는다. 위의 `Sec-WebSocket-Accept`와 동기가 정확히 같다: **WebSocket 규격의 이상해 보이는 장치들은 대체로 중간 장비를 상대로 한 방어다.**

opcode에 `ping`(0x9)/`pong`(0xA)이 **프로토콜 레벨로 들어가 있다**는 것도 중요하다. [[SSE]]가 주석 줄이라는 편법으로 하던 일을 여기선 규격이 직접 지원한다 → [[heartbeat 와 유휴 커넥션]].

## 서버

```js
import { WebSocketServer } from 'ws';
const wss = new WebSocketServer({ server, path: '/ws' });

wss.on('connection', (ws, req) => {
  const since = Number(new URL(req.url, 'http://x').searchParams.get('since') || 0);
  for (const e of store.since(since)) ws.send(JSON.stringify(e));   // 재개는 직접 구현

  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
  ws.on('message', (buf) => handleClientMessage(ws, buf));   // ← 이게 SSE 에 없는 능력
});
```

`ws.on('message')` 한 줄이 WebSocket을 고르는 유일하게 정당한 이유다. 이 줄을 안 쓸 거면 [[SSE]]가 낫다.

## 잃은 것 — 브라우저가 해주던 걸 전부 직접 짠다

[[SSE]]의 두 줄짜리 클라이언트가 여기선 이렇게 된다.

```js
let lastId = 0;
let delay = 500;

function connect() {
  const ws = new WebSocket(`wss://example.com/ws?since=${lastId}`);

  ws.onopen    = () => { delay = 500; };                 // ① 백오프 리셋
  ws.onmessage = (e) => {
    const m = JSON.parse(e.data);
    lastId = m.id;                                       // ② 재개 커서 직접 관리
    render(m);
  };
  ws.onclose   = () => {                                 // ③ 재연결 직접
    setTimeout(connect, delay);
    delay = Math.min(delay * 2, 30_000) + Math.random() * 300;
  };
  ws.onerror   = () => ws.close();                       // ④ error 는 close 를 보장하지 않는다
}
```

①~④가 전부 [[SSE]]에선 `retry:`와 `Last-Event-ID`로 공짜였던 것들이다. ③의 `Math.random()`을 장식으로 보면 안 되는 이유는 [[지수 백오프와 지터]]에 따로 적었다 — 없으면 서버 재시작 때 전 클라이언트가 **동시에** 달려든다.

④도 직관에 반한다. `onerror` 뒤에 `onclose`가 항상 온다고 가정하면 재연결이 영영 안 걸리는 경우가 생긴다. 그래서 명시적으로 닫아 `onclose`를 강제한다.

**두 번째로 잃은 것은 인증이다.** 브라우저 `WebSocket` 생성자도 [[SSE]]의 `EventSource`와 똑같이 커스텀 헤더를 못 붙인다. 쿠키를 쓰거나([[CORS]] 제약 동반), 연결 직후 첫 메시지로 토큰을 보내고 서버가 인증 전까진 아무것도 안 보내는 방식을 짜야 한다. URL 쿼리에 토큰을 실으면 프록시 접근 로그에 그대로 남으니 피한다.

## 언제 고르나

[[실시간 전송 방식 선택]]의 결론은 짧다: **양방향이거나 바이너리일 때만.** 2바이트 대 14.2바이트는 이유가 못 된다.

> 상태: #wip — 다음 가지: `permessage-deflate` 확장(압축), 서브프로토콜 협상을 실제로 쓰는 사례, 여러 서버 인스턴스에 흩어진 소켓에 브로드캐스트하는 법.

> 왜 여기 붙나: HTTP 로 시작해 HTTP 를 버린다 — 양방향과 2바이트 오버헤드

#sprout
