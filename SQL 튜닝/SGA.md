# SGA (System Global Area)

오라클 인스턴스 전체가 **공유**하는 메모리 영역. 인스턴스 기동 시 한 번 잡히고, 모든 서버 프로세스·세션이 같이 들여다본다.

[[PGA]]가 세션 사적 메모리라면, SGA는 공용 — 그래서 SGA 안의 데이터는 여러 세션이 동시에 읽고 쓰기 때문에 **동시성 제어(latch, mutex)** 가 필수다. 이게 SGA에서 일어나는 대부분의 경합·대기 이벤트의 뿌리.

## 왜 중요한가

DB가 디스크 I/O를 줄이는 핵심 장치가 SGA다. 한 번 읽은 블록을 **버퍼 캐시**에 올려두면 다른 세션이 같은 블록을 또 읽을 때 디스크를 안 가도 된다. 같은 SQL을 다시 실행할 때 **라이브러리 캐시**에 파싱된 실행계획이 남아 있으면 hard parse를 피할 수 있다.

즉, "이 시스템이 왜 빠른가/느린가"의 절반은 SGA가 얼마나 잘 히트되느냐다.

## 주요 구성 요소

- **Database Buffer Cache** — 디스크에서 읽은 데이터 블록을 캐시. Buffer Cache Hit Ratio가 여기서 나옴. LRU 기반으로 관리.
- **Shared Pool**
    - **Library Cache** — 파싱된 SQL·실행계획·PL/SQL 코드 보관. 바인드 변수를 안 쓰면 여기가 hard parse로 폭발.
    - **Data Dictionary Cache (Row Cache)** — 딕셔너리 정보 캐시.
- **Redo Log Buffer** — 커밋 전 변경사항(redo 엔트리)을 임시로 모아두는 영역. LGWR이 redo 로그 파일로 내려씀.
- **Large Pool** — RMAN 백업, 병렬 쿼리 등 큰 메모리가 필요한 작업용 별도 영역. Shared Pool 단편화 방지.
- **Java Pool, Streams Pool** — 각각 Java VM, 스트림 복제용.

## 사이즈 결정

요즘은 `MEMORY_TARGET` 또는 `SGA_TARGET`을 설정하면 오라클이 구성요소(Buffer Cache, Shared Pool 등) 사이즈를 **자동으로 분배**한다(ASMM·AMM). 옛날엔 `DB_CACHE_SIZE`, `SHARED_POOL_SIZE`를 하나하나 잡았는데, 지금은 거의 자동.

다만 자동이라도 **전체 SGA 상한**은 사람이 정해줘야 하고, 이 값이 작으면 버퍼 캐시 히트율이 떨어지거나 Shared Pool에서 ORA-04031(메모리 부족)이 터진다.

## PGA와의 대비

| | SGA | [[PGA]] |
|---|---|---|
| 범위 | 인스턴스 공유 | 세션·프로세스 사적 |
| 동시성 | latch/mutex 필수 | 없음 |
| 주 용도 | 디스크 I/O 절감 (캐시) | 정렬·해시 작업 공간 |
| 부족하면 | 캐시 미스 ↑, ORA-04031 | disk sort / multi-pass |

캐싱 효율을 위한 메모리가 SGA, 작업 공간을 위한 메모리가 PGA — 이렇게 역할이 다르기 때문에 둘 중 하나만 키운다고 해결되지 않는다.

## 연결되는 개념

- [[PGA]] — 세션 전용 메모리. SGA와 짝.
- [[소트 머지 조인]] — 정렬은 PGA에서 일어나지만, 정렬 대상 블록을 읽어오는 단계는 SGA Buffer Cache를 거침.
- 해시 조인 — 마찬가지로 Build/Probe 입력은 Buffer Cache에서.
- 라이브러리 캐시 ↔ 바인드 변수·hard parse — 캐시 효율의 핵심.
