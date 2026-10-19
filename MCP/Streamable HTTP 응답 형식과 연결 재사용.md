---
up: "[[MCP]]"
---
# Streamable HTTP 응답 형식과 연결 재사용

[[MCP]]의 각주에서 Streamable HTTP 를 "POST 하나로 합치고, 스트리밍이 필요할 때만 [[SSE]] 로 켠다" 고 정리했다. 그 "필요할 때만" 이 연결 재사용에서 발목을 잡는다.

## 응답은 두 형식 중 하나

Streamable HTTP 에서 서버는 POST 응답을 `application/json` 으로도, `text/event-stream` 으로도 줄 수 있다. 클라이언트가 `Accept` 에 둘 다 넣어야 하는 이유가 이것([[MCP 서버 로컬 개발 루프]]의 406 함정).

옛 HTTP+SSE(2024-11 스펙)와 헷갈리지 않는 감별법: **같은 URL 로 보낸 POST `initialize` 에 200 이 오고, 이후 `Mcp-Session-Id` 헤더로 세션을 이어가면** Streamable HTTP 다. 옛 방식은 먼저 GET 으로 SSE 채널을 열고 거기서 POST 할 주소를 받아 온다.

## 왜 공유 클라이언트가 연결 수를 못 줄이나

HTTP keep-alive 의 전제는 "응답 본문을 **끝까지 읽어야** 그 연결이 깨끗해져 풀로 돌아간다" 는 것. 수도관으로 치면 물을 다 빼야 다음 사람이 쓸 수 있다.

event-stream 응답은 원리상 끝이 열려 있는 스트림이다. 클라이언트 SDK 가 원하는 메시지(JSON-RPC 응답 하나)만 받고 **스트림을 끝까지 읽지 않은 채 닫으면**, 그 연결은 물이 남은 관이라 풀로 못 돌아가고 버려진다.

```
공유 클라이언트(연결 풀) 사용
  → POST tools/call → 응답 Content-Type: text/event-stream
    → SDK 가 결과 메시지만 읽고 close
      → 연결 폐기 → 다음 요청은 새 TCP + TLS
```

그래서 풀을 공유해도 새 연결 수가 거의 안 준다. **재사용을 막는 건 전송 방식(Streamable HTTP)이 아니라 응답 형식(event-stream)이다.** 같은 전송이라도 서버가 `application/json` 으로 답하면 정상적으로 재사용된다.

## 세션 하나가 치르는 왕복 수

호출마다 세션을 새로 열면 툴 한 번 부르는 데 이만큼 오간다.

| | stateful | stateless |
|---|---|---|
| `initialize` | ✓ | ✓ |
| GET 스트림(서버→클라 알림 채널) | ✓ | — |
| `notifications/initialized` | ✓ | ✓ |
| `tools/call` | ✓ | ✓ |
| `tools/list` (출력 스키마 검증용) | ✓ | ✓ |
| `DELETE` (세션 종료) | ✓ | — |
| **합계** | **6** | **4** |

실제 일은 `tools/call` 하나인데 나머지가 프로토콜 의례다. 호출마다 세션을 새로 열면 이 왕복이 지연의 상당 부분을 차지하고, 위 연결 폐기와 겹치면 왕복마다 새 연결까지 붙는다. [[SSL 컨텍스트 생성 비용 (truststore)]]와 같은 교훈 — "재사용하면 된다" 를 덩어리로 믿지 말고, 무엇이 재사용되고 무엇이 안 되는지 떼어 본다.

> 왜 여기 붙나: MCP 전송 방식(Streamable HTTP)의 각주 — "필요할 때만 SSE" 가 연결 재사용과 세션 왕복 비용으로 드러나는 지점

## 가지
- [[MCP 세션 재사용]]

#sprout
