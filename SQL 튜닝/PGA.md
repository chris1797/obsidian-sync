# PGA (Program Global Area)

오라클 서버 프로세스가 **자기 자신만 사용하는 전용 메모리 영역**. 세션별·프로세스별로 따로 잡힌다.

[[SGA]]가 모든 세션이 **공유**하는 메모리(버퍼 캐시, 라이브러리 캐시 등)라면, PGA는 **사적**이다. 그래서 PGA에 올라간 데이터에는 락(latch)이 필요 없다 — 다른 세션이 건드릴 수 없으니까.

## 왜 중요한가

[[소트 머지 조인]]의 정렬 단계, 해시 조인의 해시 테이블 빌드 단계처럼 **세션이 중간 결과를 쌓아둬야 하는 작업**은 전부 PGA를 쓴다. PGA가 부족하면 그 중간 데이터를 디스크(temp tablespace)로 흘려보내야 하고 — 이걸 **disk sort / one-pass / multi-pass** 라고 부른다. 성능이 급격히 나빠지는 지점.

즉, "이 쿼리가 왜 느린가"를 들여다볼 때 PGA에 다 못 담겨서 디스크로 새는 상황인지가 핵심 관전 포인트다.

## 구성 요소 (대략)

- **Sort Area** — ORDER BY, GROUP BY, 인덱스 빌드, [[소트 머지 조인]] 등에서 정렬용
- **Hash Area** — 해시 조인의 해시 테이블 빌드용
- **Bitmap Merge Area** — 비트맵 인덱스 머지
- **Stack Space, Session Memory** — 바인드 변수·커서 상태 등

## 사이즈 결정

`WORKAREA_SIZE_POLICY = AUTO` (요즘 기본값)이면 `PGA_AGGREGATE_TARGET`을 보고 오라클이 세션마다 적당히 분배. 수동(`MANUAL`)일 땐 `SORT_AREA_SIZE`, `HASH_AREA_SIZE` 같은 파라미터로 직접 지정.

운영 DB는 거의 AUTO인데, 그래도 한 세션이 잡을 수 있는 상한이 있어서 **대용량 정렬·해시 조인이 PGA에 다 못 들어가면 결국 디스크로 빠진다**. 튜닝 관점에서는 이 임계점을 의식하는 게 중요.

## 연결되는 개념

- [[SGA]] — 공유 메모리. PGA와 짝.
- [[소트 머지 조인]] — 정렬 단계가 PGA Sort Area에서 일어남.
- 해시 조인 — Build 단계가 PGA Hash Area에서 일어남.
