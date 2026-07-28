# FastMCP 대규모 도구의 토큰 절약

핵심은 에이전트가 **적게 읽고, 적게 호출하고, 작은 결과만 받게 하는 것**이다.

도구 500개를 연결하는 건 아이에게 공구 500개의 설명서를 매번 전부 읽힌 뒤 망치 하나를 고르게 하는 것과 같다. 실제 작업 전에 도구 이름·설명·입력 형식을 LLM에 알려주는 데도 토큰이 든다. 도구 실행 결과가 길거나 같은 호출을 반복하면 토큰은 더 늘어난다.

## 1순위 — 사용할 도구만 보여주기

**Allowlist(허용 목록)** 는 "이 에이전트가 사용할 수 있는 도구 목록"이다.

```
전체 MCP 도구 500개
    ↓ 배포 업무에 필요한 것만 허용
배포 에이전트가 보는 도구 20개
```

FastMCP 3.0 이상에서는 도구에 태그를 붙인 뒤 필요한 태그만 보이게 할 수 있다.

```python
from fastmcp import FastMCP

mcp = FastMCP("Company MCP")

@mcp.tool(tags={"deployment"})
def get_deployment_status(deployment_id: str) -> dict:
    """배포 상태를 조회한다."""
    ...

@mcp.tool(tags={"admin"})
def delete_deployment(deployment_id: str) -> dict:
    """배포 기록을 삭제한다."""
    ...

# deployment 태그가 붙은 도구만 목록과 호출에서 허용
mcp.enable(tags={"deployment"}, only=True)
```

`only=True`가 중요하다. 이것이 없으면 일부 도구를 켜고 끄는 방식이고, 있으면 명시적으로 선택한 도구만 허용하는 방식이 된다. 꺼진 도구는 목록에서 사라질 뿐 아니라 호출도 할 수 없다.

업무가 여러 종류라면 배포·Slack·GitHub·위키처럼 서버나 도구 묶음을 나누고, 에이전트 역할에 맞는 묶음만 제공한다. 사용자마다 권한이 다르면 단순 화면 숨김으로 끝내지 말고 서버의 인증·권한 검사에서도 같은 제한을 강제해야 한다.

## 2순위 — 필요한 도구 설명만 나중에 읽기

**지연 로딩(lazy loading)** 은 도구 설명서를 창고에 넣어두고 필요할 때만 꺼내는 방식이다.

FastMCP 3.1 이상에서는 Tool Search transform을 붙이면 500개 도구 대신 기본적으로 다음 두 도구만 보인다.

- `search_tools`: 필요한 도구를 검색
- `call_tool`: 검색해서 찾은 실제 도구를 호출

```python
from fastmcp import FastMCP
from fastmcp.server.transforms.search import BM25SearchTransform

mcp = FastMCP("Company MCP")

# 관련도가 높은 도구 설명을 검색할 때 최대 3개만 반환
mcp.add_transform(BM25SearchTransform(max_results=3))
```

흐름은 다음과 같다.

```
사용자: "어제 실패한 배포 원인을 찾아줘"
  ↓
LLM이 search_tools로 "배포 실패 로그" 검색
  ↓
관련 도구 3개의 설명만 받음
  ↓
call_tool로 필요한 도구 호출
```

항상 자주 쓰는 `help`, `status` 같은 도구는 검색 없이 바로 보이도록 고정할 수도 있다.

```python
mcp.add_transform(BM25SearchTransform(
    max_results=3,
    always_visible=["help", "status"],
))
```

Allowlist와 지연 로딩은 서로 대신하는 기능이 아니다.

```
500개 전체 도구
    ↓ Allowlist: 배포 도구만 허용
20개
    ↓ 지연 로딩: 이번 요청과 관련된 설명만 검색
3개
```

## 3순위 — 반복 업무를 스킬로 만들기

스킬에는 다음 내용을 적는다.

- 어떤 요청에서 사용할지
- 어떤 MCP 도구 묶음을 선택할지
- 호출 순서
- 재시도 횟수
- 작업을 끝내는 조건

스킬은 잘못된 도구 선택과 불필요한 재호출을 줄인다. 하지만 스킬만 추가한다고 500개 도구 설명이 자동으로 사라지지는 않는다. **스킬은 길 안내**, Allowlist와 지연 로딩은 **실제로 보여주는 설명서의 양을 줄이는 장치**다.

## 4순위 — 반복 호출을 하나로 묶기

항상 같은 순서로 실행되는 작업은 서버 내부에서 처리하고 결과만 돌려준다.

```
기존:
배포 조회 → 로그 조회 → 커밋 조회 → 장애 조회

개선:
diagnose_deployment_failure 한 번 호출
```

이런 도구를 **복합 도구** 또는 **서버 측 워크플로**라고 한다. 각 단계 사이에서 LLM이 결과를 읽고 다음 호출을 생각하지 않아도 되므로 토큰과 호출 횟수가 줄어든다. 모든 작업을 억지로 합치지 말고 호출 빈도가 높은 상위 10~20개 흐름부터 묶는다.

## 5순위 — 도구 결과를 작게 돌려주기

조회 도구는 기본적으로 작은 결과를 반환해야 한다.

```json
{
  "limit": 10,
  "fields": ["id", "status", "summary"],
  "detail_level": "summary"
}
```

- 목록은 기본 10개만 반환
- 필요한 필드만 반환
- 로그 전체 대신 오류 앞뒤만 반환
- 원문 대신 요약과 문서 ID 반환
- 자세한 내용은 추가 요청이 있을 때만 반환
- 긴 오류 메시지와 스택 트레이스를 그대로 반환하지 않음

FastMCP는 Python 함수의 반환 타입을 이용해 구조화된 결과를 만들 수 있다. 반환 형식을 명확히 하고 불필요한 문자열 설명을 반복하지 않는다.

## 함께 적용할 작은 규칙

- 도구 이름과 설명에서 반복 문장을 제거한다.
- 동일 호출을 반복하지 않는다.
- 재시도는 최대 1~2회로 제한한다.
- 이미 얻은 결과를 다시 조회하지 않는다.
- 원하는 정보를 찾으면 즉시 끝낸다.
- 오래된 MCP 결과는 원문 대신 짧은 요약만 남긴다.

서버 캐시는 외부 API나 DB를 다시 조회하는 비용과 시간을 줄인다. 하지만 같은 긴 결과를 LLM에 다시 전달한다면 LLM 토큰은 줄지 않는다. 캐시와 토큰 절약은 같은 말이 아니다.

## FastMCP 적용 순서

1. 현재 FastMCP 버전을 확인한다.
2. 도구에 `deployment`, `slack`, `github`, `wiki` 같은 업무 태그를 붙인다.
3. FastMCP 3.0 이상이면 `enable(..., only=True)`로 업무별 Allowlist를 적용한다.
4. FastMCP 3.1 이상이면 `BM25SearchTransform(max_results=3)`을 시험한다.
5. 조회 도구에 `limit`, `fields`, `summary` 방식을 적용한다.
6. 반복 호출 상위 10~20개를 스킬이나 복합 도구로 만든다.
7. 변경 전후의 도구 설명 토큰, 결과 토큰, 호출 횟수, 성공률을 비교한다.

> 한 줄 결론: **500개 중 필요한 도구만 허용하고, 그중 이번 요청에 필요한 설명만 검색하며, 호출 결과는 작게 돌려준다.**

## 관련 노트

- [[MCP]]
- [[MCP 서버 조합 (프록시·집계)]] — 여러 매체 MCP를 붙이면 툴이 폭증하는데, 여기 allowlist가 그 대응책

## 참고

- [FastMCP Component Visibility](https://gofastmcp.com/servers/visibility) — FastMCP 3.0+, Allowlist와 도구 표시 제어
- [FastMCP Tool Search](https://gofastmcp.com/servers/transforms/tool-search) — FastMCP 3.1+, 대규모 도구 목록의 검색 방식
- [FastMCP Transforms](https://gofastmcp.com/servers/transforms/transforms) — 도구가 클라이언트에 전달되기 전 변환하는 구조
- [FastMCP Tool Transformation](https://gofastmcp.com/servers/transforms/tool-transformation) — 긴 도구 이름·설명·입력 형식 정리
- [MCP Tools 명세](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) — `tools/list`, `tools/call`, 구조화 결과

> 상태: #wip — 대규모 도구의 토큰 절약 전략과 FastMCP 적용 방법까지.
