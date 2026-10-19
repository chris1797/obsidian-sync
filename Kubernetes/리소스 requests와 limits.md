---
up: "[[Pod]]"
---
# 리소스 requests와 limits

[[Pod]] 스펙의 `resources` 에는 숫자가 두 개 들어간다. 처음엔 "최소·최대" 정도로 읽었는데, 둘은 **보는 주체도, 커널에서 바뀌는 모양도 다르다.** 한 문장으로: **requests 는 바닥(최소 보장), limits 는 천장(상한)** 이고, 파드는 그 사이에서 노드 여유만큼 쓴다.

## 단위부터

`m` 은 1/1000 CPU. `1000m = 코어 1개`, `350m = 0.35 CPU`. 절대량이다 — 노드가 몇 코어든 350m 은 350m.

## requests — 장부상 예약 + 경합 때의 몫

"CPU 예약량" 이라고 부르면 반만 맞다. 역할이 둘이다.

**1. 스케줄러의 장부.** [[컨트롤 플레인]]의 스케줄러는 "이 노드의 requests 합이 **allocatable**(노드 용량에서 시스템 예약분을 뺀 것)을 넘지 않는가" 만 본다. **실제 사용량은 안 본다.**

```
allocatable 4000m
- 이미 올라간 파드들의 requests 합 3000m
= 남은 자리 1000m  → 350m 파드는 들어감, 1200m 파드는 Pending
```

CPU 가 놀고 있어도 장부가 차 있으면 못 들어간다. 그리고 파드는 쪼개지지 않으니, 두 노드에 400m 씩 남아 합이 800m 이어도 700m 파드는 Pending 이다.

**2. 실행 중엔 가중치.** 노드에서 requests 는 cgroup 의 `cpu.shares`(v1) / `cpu.weight`(v2) 가 된다. 몇 ms 같은 절대량이 아니라 **상대 비율**이다.

| 상황 (2코어 노드, 둘 다 CPU 최대로 원함, limits 없음) | A (req 350m) | B (req 700m) |
|---|---|---|
| 1:2 비율로 나눔 | ≈ 667m | ≈ 1333m |

둘 다 자기 requests 보다 많이 받는다 — 남는 CPU 까지 비율대로 나누니까. B 가 한가해지면 A 가 거의 다 쓴다. **requests 는 아무도 멈추게 하지 않는다.** 붐빌 때 "최소 이만큼은 돌려받는다" 는 보장일 뿐.

그래서 도서관 좌석 예약에 가깝다. 예약자가 없으면 남이 앉아도 되지만, 예약자가 오면 자리를 돌려받는다. 다만 열람실 장부가 차면 빈 좌석이 있어도 새 예약은 안 받는다.

## limits — 넘을 수 없는 천장

CPU limits 는 CFS 쿼터가 된다. "100ms 마다 70ms" 같은 **시간 예산**이고, 다 쓰면 구간 끝까지 멈춘다. 이 멈춤이 [[CPU throttling]] — 메커니즘은 그쪽에 따로 적었다.

스케줄러는 limits 를 **보지 않는다.** 그래서 노드의 limits 합이 용량의 2~3배인 것(overcommit)도 허용된다. "다들 동시에 최대로 쓰진 않겠지" 라는 도박이고, 다들 동시에 바빠지면 결국 requests 비율 싸움으로 돌아간다.

## 메모리는 다르게 끝난다

CPU 는 시간을 쪼개 쓰는 자원이라 "잠깐 멈춰" 로 제한할 수 있다. 메모리는 이미 차지한 공간을 뺏을 방법이 없다 → limits 를 넘으면 **OOM kill 후 재시작.** CPU limits 는 빼는 선택지가 있어도([[CPU throttling]] 참고) 메모리 limits 는 빼면 안 되는 이유가 이 비대칭이다.

## QoS — 쫓겨나는 순서

requests·limits 조합으로 파드 등급이 매겨지고, 노드가 메모리 압박을 받을 때 쫓아내는(eviction) 순서가 된다.

- **Guaranteed** — 모든 컨테이너가 CPU·메모리 모두 requests = limits
- **Burstable** — 그 외 하나라도 requests 가 있으면
- **BestEffort** — 아무것도 없음. 제일 먼저 쫓겨남

CPU requests 350m / limits 700m 이면 메모리를 같게 줘도 Burstable 이다.

## 올리기 전에 볼 것

requests 를 올리면 장부상 자리를 더 차지한다. 배포 중 surge 로 뜨는 파드까지 자리가 필요하다([[Deployment]]). 그래서 확인할 건 "노드 CPU 사용률" 이 아니라 **"후보 노드들의 allocatable − requests 합"**, 그리고 한 노드에 파드 하나가 통째로 들어가는가.

[[HPA]] 는 이 requests 를 사용률의 **분모**로 쓴다 — requests 를 바꾸면 같은 부하라도 사용률 숫자가 바뀐다.

> 상태: #wip — 다음 가지: LimitRange·ResourceQuota(네임스페이스가 이 값들을 강제·기본값 주입하는 방식), Pod 레벨 리소스(1.34+).

> 왜 여기 붙나: Pod 스펙 resources 의 두 숫자 — requests 는 바닥, limits 는 천장

## 가지
- [[CPU throttling]]
- [[LimitRange·ResourceQuota]]

#sprout
