> "FastAPI 는 싱글 스레드인가?" 한마디로 답하기 어려운 이유 — 세 층으로 나눠야 한다.

### 1. 이벤트 루프는 싱글 스레드다

FastAPI 는 ASGI(Starlette 위) 기반. 한 워커 프로세스 안에는 **asyncio 이벤트 루프 1개**가 돈다. `async def` 엔드포인트는 이 루프 위의 코루틴으로 실행되고, `await` 지점마다 다른 코루틴에 양보하며 동시성을 만든다. 이 메커니즘 자체는 [[이벤트 루프와 코루틴]] 노트에서 한 단계 더 파봤다.

→ 스레드를 늘려서 동시성을 얻는 게 아니라, **한 스레드 안에서 I/O 대기 시간을 겹쳐서** 동시성을 얻는 모델.

Spring Boot (Tomcat) 의 요청당 스레드 모델과 정반대다. Tomcat 은 요청이 오면 스레드풀에서 하나를 꺼내 할당하고, 그 스레드가 블로킹 I/O 도중에 잠들면 다른 스레드가 다른 요청을 처리한다. FastAPI 는 잠들지 않는다 — 코루틴이 그냥 양보한다.

### 2. `def` 엔드포인트는 자동으로 스레드풀로 빠진다

엔드포인트를 `async def` 가 아닌 그냥 `def` 로 선언하면, Starlette 가 알아서 `run_in_threadpool` (anyio 스레드풀) 로 던진다.

```python
@app.get("/a")
async def a():   # 이벤트 루프에서 직접 실행
    ...

@app.get("/b")
def b():         # 스레드풀로 위임 (멀티스레드 경로)
    ...
```

→ 동기 블로킹 코드가 이벤트 루프를 막지 못하게 하는 **안전장치**. 이 경로만 보면 멀티스레드.

⚠️ 그렇다고 "어차피 스레드풀로 빠지니 다 `def` 로 써도 되겠네" 는 함정 — 스레드풀 크기는 유한하다 (기본 anyio 40개). I/O 가 진짜 많은 서비스라면 `async def` + async 드라이버 조합이 본래 의도.

### 3. CPU 코어 활용은 프로세스로

운영에서는 `uvicorn --workers N` 또는 `gunicorn -k uvicorn.workers.UvicornWorker -w N` 으로 프로세스 N 개를 띄운다. 각 프로세스가 독립된 이벤트 루프를 가지므로, **멀티코어 활용은 여기서** 일어난다.

→ 파이썬 [[GIL]] 때문에 한 프로세스 안에서 스레드만 늘려서는 CPU 병렬을 못 얻는다. 그래서 어쩔 수 없이 프로세스로 확장. JVM 기반 Spring 이 한 프로세스로 코어를 전부 쓰는 것과 결정적으로 다른 지점.

### 가장 흔히 밟는 지뢰

`async def` 안에서 **동기 블로킹 함수** (`requests.get`, `psycopg2`, `time.sleep` 등) 를 그대로 부르면, 그 워커의 이벤트 루프 **전체가 멈춘다**. 한 사용자의 느린 쿼리가 같은 워커에서 처리 중인 모든 요청을 마비시키는 그림.

대응:
- HTTP → `httpx.AsyncClient` (대신 `requests`)
- DB → `asyncpg` / `aiomysql`, 또는 SQLAlchemy 의 [[AsyncSession]] (대신 동기 `Session`)
- 그것도 안 되면 차라리 엔드포인트를 `def` 로 둬서 스레드풀에 맡기기

### 한눈에

| | Spring Boot (Tomcat) | FastAPI |
|---|---|---|
| 동시성 단위 | 요청당 스레드 | 코루틴 (이벤트 루프) |
| 블로킹 I/O 대처 | 스레드는 잠들고 풀의 다른 스레드가 다른 요청 처리 | 이벤트 루프가 멈춤 → 같은 워커 전체 마비 |
| CPU 확장 | 스레드 수 ↑ (한 JVM 안에서) | 프로세스 수 ↑ ([[GIL]] 때문) |

### 의문점 / 더 파볼 것

- 스레드풀 크기 조정 (`anyio.to_thread.current_default_thread_limiter()`) 은 언제 건드릴 가치가 있나?
- async 컨텍스트에서 SQLAlchemy 를 쓸 때 세션 스코프 / 트랜잭션 경계는 JPA 의 영속성 컨텍스트와 어떻게 다른가? → [[AsyncSession]] 으로 따로 정리.
- uvicorn workers 와 gunicorn + uvicorn worker 의 실질적 차이 (헬스체크, 그레이스풀 리스타트 등)
