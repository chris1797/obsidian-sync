# asyncio.gather — 여러 코루틴을 동시에 기다리기, 그리고 예외가 삼켜지는 함정

**여러 코루틴을 한 이벤트 루프에 동시에 올려놓고 "다 끝나면 깨워줘"라고 기다리는 도구.** [[이벤트 루프와 코루틴]]에서 `await`는 "기다림이 아니라 양보 요청"이라고 했는데, 그건 코루틴 *하나*를 기다리는 얘기였다. gather는 그걸 *여러 개*로 확장한다 — 하나씩 순서대로 `await`하면 직렬인데, gather로 묶으면 셋이 동시에 진행되다 마지막 하나가 끝날 때 함께 돌아온다.

## 왜 필요한가 — 순차 await의 낭비

심부름 세 개를 시킨다 치자. 각각 3초 걸리는 네트워크 요청이다.

```python
# 직렬 — 9초. 하나 끝나야 다음을 시작
a = await fetch("A")
b = await fetch("B")
c = await fetch("C")

# gather — 3초. 셋을 동시에 루프에 올리고 한꺼번에 기다림
a, b, c = await asyncio.gather(fetch("A"), fetch("B"), fetch("C"))
```

직렬 버전이 느린 건 CPU가 바빠서가 아니다. `fetch("A")`가 응답을 기다리며 **양보한 그 시간에 B를 시작하지 않고 놀기 때문**이다. gather는 그 노는 시간에 형제들을 밀어넣는다. I/O 동시성([[GIL]]이 막은 CPU 병렬 말고, 기다림을 겹치는 쪽)의 전형적 무기.

## 함정 1 — 하나가 터지면 나머지는? (`return_exceptions`)

여기가 gather의 진짜 배울 점이다. 기본값 `return_exceptions=False`일 때:

```
gather(A, B, C)   ← B가 예외를 던짐
  ↓
gather는 즉시 그 예외를 위로 던진다 (await 지점에서 raise)
  ↓
하지만 A, C는? → 취소되지 않고 백그라운드에서 계속 돈다 (!)
```

즉 **예외 하나가 gather를 뚫고 나와도 나머지 형제는 자동으로 멈추지 않는다.** 호출자는 "실패했구나" 하고 넘어가는데 A·C는 좀비처럼 살아 리소스를 쥐고 있을 수 있다. `return_exceptions=True`로 바꾸면 예외를 *던지는 대신 결과 리스트에 값처럼 담아* 돌려준다 — 그러면 세 개 결과를 다 받아 하나하나 "이건 성공, 이건 예외" 골라낼 수 있다.

## 함정 2 — fire-and-forget의 silent 예외

[[이벤트 루프와 코루틴]]에서 예고했던 그 함정이 이것. `create_task`로 던져놓고 **`await`하지 않으면**, 그 Task가 안에서 터진 예외는 아무도 안 받는다.

```python
asyncio.create_task(may_fail())   # 던지고 잊음
# may_fail()이 예외를 던져도 — await하는 순간까지 조용하다.
# 최악: 루프 종료 때 "Task exception was never retrieved" 경고만 뒤늦게.
```

`await`가 예외를 꺼내오는 통로인데, 그 통로를 안 열면 예외가 Task 안에 갇힌다. gather나 TaskGroup으로 반드시 거둬들여야 하는 이유.

## TaskGroup (3.11+) — gather의 현대적 후계

gather의 "형제가 안 멈춘다" 문제를 구조적으로 푼 게 **`asyncio.TaskGroup`**. 이름이 *structured concurrency*(구조적 동시성)다.

```python
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch("A"))
    tg.create_task(fetch("B"))
    tg.create_task(fetch("C"))
# with 블록을 나갈 때 셋이 다 끝나 있음이 보장됨
```

- 블록 안의 형제 중 **하나가 터지면 나머지를 자동 취소**하고, 예외들을 `ExceptionGroup`으로 묶어 던진다.
- "이 블록을 벗어나면 여기서 띄운 Task는 전부 끝났거나 정리됐다"가 문법으로 보장 — gather의 좀비 형제 문제가 사라진다.

정리하면: **동시에 여럿 기다리기 = gather가 입문, TaskGroup이 정답**(3.11+). gather는 여전히 "결과만 모으면 되고 실패는 각자 처리" 같은 단순 케이스에 쓴다.

## 관련 노트

- [[이벤트 루프와 코루틴]] — `await`=양보라는 뿌리. gather/TaskGroup은 그 양보를 여러 개로 겹치는 것
- [[FastAPI 동시성 모델]] — 웹 요청 하나 안에서 외부 호출 여럿을 겹칠 때 실제로 이 도구를 씀
- [[GIL]] — 여기서 얻는 건 CPU 병렬이 아니라 I/O 대기의 중첩임을 가르는 경계

> 상태: #wip — gather의 예외 함정과 TaskGroup 대비까지. 다음 가지: `ExceptionGroup`/`except*` 문법, 취소(`CancelledError`) 전파 규칙.
