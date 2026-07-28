# 동시성 MOC

이 영역의 입구. 핵심 축은 **"GIL이 막은 길과, 그 우회로"** — 파이썬은 스레드로 CPU 병렬을 못 얻으니(=GIL), 한 스레드 안의 *협력적 동시성*으로 I/O 동시성을 얻는 쪽으로 갔다. 이 한 문장이 아래 네 노트를 관통한다.

## 읽는 순서

1. [[GIL]] — 제약의 뿌리. 왜 스레드를 N개 띄워도 CPU 병렬이 안 되는가. "CPU 병렬은 스레드 말고 프로세스"라는 결론이 다음 챕터들의 전제.
2. [[이벤트 루프와 코루틴]] — 그래서 택한 협력적 동시성의 *메커니즘*. `await`는 기다림이 아니라 **양보 요청**. 이 한 줄이 async의 80%.
3. [[FastAPI 동시성 모델]] — 그 원리가 실제 웹 프레임워크에서 세 층(이벤트 루프 / `def`→스레드풀 / 프로세스 N개)으로 드러남. 밟는 지뢰까지.
4. [[AsyncSession]] — 그 모델이 DB 계층에서 만드는 함정. lazy loading 차단이 JPA [[N+1 문제]]와 *정반대 디폴트*로 같은 문제를 푸는 지점 — 여기서 DB 영역과 이어진다.

## 다음 가지 후보

`run_in_executor` / `run_in_threadpool`(동기 코드를 async로 끌어들이는 다리), [[asyncio.gather]] / TaskGroup(Task 예외가 silent해지는 함정), async generator(`async for`, 스트리밍), `multiprocessing` IPC 비용, Python 3.13 free-threaded 빌드가 refcount를 어떻게 푸는가.
