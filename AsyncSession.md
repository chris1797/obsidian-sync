> SQLAlchemy 2.0+ 에서 **async 컨텍스트**용으로 만들어진 세션. 기본 `Session` 의 모든 I/O 메서드를 코루틴으로 노출한다. [[FastAPI 동시성 모델]] 의 "지뢰 피하기" 가 시작되는 곳.

### 왜 별도의 세션인가

기본 `Session` 의 메서드 (`execute`, `commit`, `flush`, lazy loading 접근 등) 는 전부 **동기 블로킹**. async 엔드포인트에서 그대로 부르면 이벤트 루프가 멈춘다.

```python
async def get_user(session: AsyncSession, user_id: int):
    result = await session.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one()
```

→ 핵심은 **드라이버 교체**. `psycopg2` (동기) 대신 `asyncpg` (async), `pymysql` 대신 `aiomysql`. 엔진은 `create_async_engine("postgresql+asyncpg://...")` 식으로 만든다. 동기 드라이버 위에 `AsyncSession` 을 얹어도 안쪽에서 블로킹이라 의미가 없다.

### JPA 영속성 컨텍스트와 비교

SQLAlchemy 세션과 JPA 의 영속성 컨텍스트는 둘 다 **Unit of Work** 패턴이라 닮은 점이 많다.

| | JPA `EntityManager` | SQLAlchemy `Session` |
|---|---|---|
| 1차 캐시 (identity map) | 있음 | 있음 |
| dirty checking | 있음 (자동) | 있음 (flush 시점) |
| flush 트리거 | 커밋 / 쿼리 직전 / 명시 호출 | 명시 호출 위주 (`autoflush` 기본 ON 이지만 더 보수적) |
| lazy loading | 프록시로 트랜잭션 안에서 가능 | **async 에서는 기본 금지** |

→ 가장 큰 함정이 **lazy loading**. 동기 `Session` 에서는 `user.posts` 접근 시 알아서 쿼리를 또 날린다. 그런데 `AsyncSession` 에서는 lazy 트리거가 **`await` 없는 곳에서** 일어나므로 기본 차단. 모르고 쓰면 `MissingGreenlet` 같은 폭탄을 맞는다.

→ 해결: **explicit eager loading**. 쿼리 시점에 미리 다 끌어온다.
```python
result = await session.execute(
    select(User).options(selectinload(User.posts))
)
```

이 지점은 JPA 의 [[N+1 문제]] 와 정확히 같은 문제를 다른 디폴트로 푸는 것. JPA 는 "lazy 가 디폴트, 필요할 때 `JOIN FETCH` 로 풀어라" 라면, async SQLAlchemy 는 "lazy 가 사실상 금지, 항상 eager 를 명시해라" — 같은 문제의 입장이 정반대다.

💡 어떻게 보면 async SQLAlchemy 가 더 정직한 모델일 수도. JPA 처럼 "내가 알아서 쿼리 또 날려줄게" 가 멀티스레드/단일 트랜잭션 환경에선 자연스러웠지만, async 에서는 그 자동 마법이 통째로 깨진다.

### 세션 스코프 — 요청당 1개

FastAPI 에서는 의존성 주입으로 요청당 세션을 만들고 닫는 패턴이 표준.

```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_factory() as session:
        yield session

@app.get("/users/{uid}")
async def read_user(
    uid: int,
    session: AsyncSession = Depends(get_session),
):
    ...
```

→ 세션은 **스레드 세이프도 아니고, async-task 세이프도 아니다**. 요청 경계 = 세션 경계로 두는 게 안전. Spring 의 `@Transactional` 처럼 메서드 단위로 자동 관리해 주는 어노테이션은 없고, 컨텍스트 매니저 (`async with session.begin():`) 로 직접 트랜잭션을 짠다.

→ 이게 처음에 어색한 부분. Spring 에서는 트랜잭션이 "어노테이션 한 줄" 의 인프라적 존재였는데, SQLAlchemy 에서는 트랜잭션이 **코드 흐름의 일부** 로 드러난다. 더 명시적인 만큼 더 책임이 사용자에게.

### 의문점 / 더 파볼 것

- `expire_on_commit` 의 기본값이 `True` 라서 커밋 직후 객체 접근 시 또 lazy load 가 일어나 async 에서 폭발. async 에서는 거의 디폴트로 `False` 같은데, 왜 기본이 `True` 일까?
- `selectinload` vs `joinedload` 선택 기준 (one-to-many → selectin, to-one → joined 가 통설)
- SQLAlchemy 의 이벤트 훅 (`before_flush` 등) 은 async 컨텍스트에서 어떻게 불리나? sync 콜백을 async 에서 부르면 또 블로킹 문제?
