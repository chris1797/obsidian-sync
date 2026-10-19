---
up: "[[실시간 전송 방식 선택]]"
---
# Long Polling

**응답을 일부러 늦게 보내는 HTTP.** 새 기술이 아니라 [[REST API]]의 평범한 `GET` 하나를 서버가 손에 쥔 채 리턴하지 않는 것뿐이다. 그래서 "실시간 통신 방식"이라기보다 **무상태 HTTP로 실시간을 흉내 내는 편법**에 가깝다.

이름 때문에 오해했던 것부터 적는다. polling은 "주기적으로 물어보기"라 자원 낭비의 대명사인데, long polling은 **주기가 없다.** 물어보고 → 답이 올 때까지 기다리고 → 답이 오면 즉시 다시 물어본다. 빈손으로 돌아오는 요청이 없다는 게 short polling과의 결정적 차이다.

## 서버 — 리턴하지 않는 핸들러

```js
app.get('/events', (req, res) => {
  const since = Number(req.query.since || 0);

  // ① 밀린 게 있으면 즉시 응답 (요청 사이에 놓친 이벤트 복구)
  const backlog = store.since(since);
  if (backlog.length) return res.json({ cursor: store.cursor, events: backlog });

  // ② 없으면 res 를 쥔 채 리턴하지 않는다 = 요청이 열린 상태로 대기
  const onEvent = (evt) => {
    clearTimeout(timer);
    bus.off('event', onEvent);
    res.json({ cursor: evt.id, events: [evt] });   // ← 여기서 비로소 응답 종료
  };
  bus.once('event', onEvent);

  // ③ 25초 안에 아무 일 없으면 빈 응답으로 스스로 끊는다
  const timer = setTimeout(() => {
    bus.off('event', onEvent);
    res.json({ cursor: since, events: [] });
  }, 25_000);

  req.on('close', () => { clearTimeout(timer); bus.off('event', onEvent); }); // ④ 누수 방지
});
```

③이 제일 중요하고 제일 자주 빠뜨리는 부분이다. **서버가 프록시보다 먼저 양보해야 한다.** Nginx 기본 타임아웃이 60초인데 서버가 90초를 버티면, 클라이언트가 보는 건 이벤트가 아니라 504다. 서버 로직은 멀쩡한데 인프라가 먼저 끊는 이 구도는 [[CORS]]에서 "막는 주체가 서버가 아니라 브라우저"였던 것과 같은 종류의 착시다 — **문제를 일으킨 층과 증상이 보이는 층이 다르다.**

④도 그냥 예의가 아니다. 클라이언트가 탭을 닫아도 `bus` 리스너와 타이머가 남으면 커넥션당 누수가 쌓인다. 유휴 커넥션 500개에 21KB씩 쓰는 건 정상 동작이지만, 리스너 누수는 그 위에 얹히는 진짜 비용이다.

## 클라이언트 — while 루프가 전부

```js
let cursor = 0;
async function loop() {
  while (true) {
    try {
      const r = await fetch(`/events?since=${cursor}`);   // 여기서 최대 25초 멈춤
      const { cursor: c, events } = await r.json();
      cursor = c;
      events.forEach(render);
    } catch {
      await sleep(1000);        // 네트워크 오류 → 잠깐 쉬고 재시도
    }
    // 응답을 받자마자 while 이 돌아 다음 요청을 즉시 쏜다
  }
}
```

여기 `catch` 안의 `sleep(1000)`은 사실 [[지수 백오프와 지터]]로 바뀌어야 한다. 서버가 죽었을 때 초당 1회씩 전 클라이언트가 두드리면 살아나려는 서버를 다시 눕힌다.

## 대가 — 봉투가 편지보다 크다

이벤트 1건당 실제로 오간 것을 뜯어보면 이 방식의 성격이 드러난다.

```
→ 요청  GET /events?since=41 HTTP/1.1
        Host / User-Agent / Accept / Cookie ...        ≈ 600 B
← 응답  HTTP/1.1 200 OK
        Content-Type / Content-Length / Date ...       ≈ 170 B
        {"cursor":42,"events":[...]}                     117 B
                                              합계    ≈ 885 B
```

**본문 117바이트에 봉투가 770바이트.** 이걸 1,000번 하면 884,698바이트, [[WebSocket]]의 7.4배가 된다. 그런데 이 낭비는 long polling의 죄가 아니라 HTTP/1.1이 헤더를 매번 원문으로 보내는 성질 때문이고, HTTP/2에서 182,475바이트(1.5배)로 떨어진다 → [[HTTP 헤더 오버헤드]].

## 그래도 이게 이기는 자리

버릴 방식으로 취급했다가 생각을 바꾼 이유가 둘 있다.

**끊김이 정상 동작이다.** [[SSE]]와 [[WebSocket]]은 "연결이 살아 있어야 정상, 끊기면 복구"라는 상태를 관리해야 한다. long polling은 매 사이클이 새 요청이라 복구할 상태가 없다. `cursor`만 들고 있으면 재개도 공짜다. [[heartbeat 와 유휴 커넥션]]이 필요 없는 유일한 방식이기도 하다 — 25초마다 응답이 오가니 유휴 상태 자체가 안 생긴다.

**[[REST API]]의 무상태를 깨지 않는다.** 서버가 커넥션을 장기 보유하지 않으니 아무 인스턴스나 요청을 받을 수 있다. [[MCP]]가 상시 SSE 채널을 버리고 Streamable HTTP로 간 이유와 같은 동기다. 다만 요청이 열린 25초 동안은 그 인스턴스가 잡혀 있으므로, 무상태는 맞아도 "무비용"은 아니다.

정리하면 **이벤트가 드물고(초당 1건 미만) HTTP/2를 쓴다면** 여전히 합리적 선택이다. 판단 기준 전체는 [[실시간 전송 방식 선택]].

> 상태: #wip — 다음 가지: 서버 인스턴스가 여럿일 때 이벤트를 어느 인스턴스가 받아 누구에게 넘길 것인가(팬아웃/백플레인), `cursor` 를 DB에 둘지 메모리에 둘지의 트레이드오프.

> 왜 여기 붙나: 응답을 일부러 늦게 보내 무상태 HTTP 로 실시간을 흉내 낸다

#sprout
