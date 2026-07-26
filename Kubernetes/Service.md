# Service

[[Pod]]는 죽고 살며 IP가 계속 바뀐다 — 소모품성의 대가다. 그 흔들리는 Pod들 앞에 **고정된 진입점** 하나를 세워, 다른 데서 안정적으로 찾고(discovery) 부하를 나누는(load balancing) 층이 Service. 한 문장: **"바뀌는 Pod IP를 고정된 이름·주소 뒤에 숨긴다."**

## 왜 필요한가 — Pod IP는 못 믿는다

[[Pod]] 노트에서 본 대로, 재배포·장애로 Pod가 교체되면 **새 IP**를 받는다. 클라이언트가 Pod IP를 직접 알고 있으면 교체 때마다 연결이 깨진다. Service는 이걸 두 개로 푼다.

- **고정 가상 IP(ClusterIP) + DNS 이름** — 절대 안 바뀌는 진입점.
- **라벨 셀렉터** — "label이 `app=aim`인 Pod들"처럼 *조건*으로 대상을 고른다. Pod가 죽고 새로 떠도, 조건에 맞으면 자동으로 뒤에 편입. 그래서 IP를 하나하나 알 필요가 없다.

즉 Service는 "지금 살아있는, 조건에 맞는 Pod 집합"을 항상 가리키는 **안정된 별명**이다.

## 3가지 타입 — 노출 범위가 다르다

- **ClusterIP** (기본) — 클러스터 *내부* 전용 고정 IP. 내부 서비스끼리 부를 때.
- **NodePort** — 각 노드의 특정 포트를 열어, 외부에서 `노드IP:포트`로 접근.
- **LoadBalancer** — 클라우드 로드밸런서를 붙여 외부 진입점 하나로 묶음. 실무 외부 노출은 대개 이것.

위로 갈수록 노출 범위가 넓어진다. 사실 상위 타입은 하위를 품는다 — LoadBalancer가 내부적으로 NodePort를, NodePort가 ClusterIP를 깔고 앉는 식.

## kube-proxy — 추상 뒤의 실제 라우팅

Service의 고정 IP는 *추상*일 뿐, 실제 패킷은 아무도 안 듣는 가상 IP로 간다. "그 IP로 온 트래픽을 살아있는 Pod 중 하나로" 실제로 넘기는 건 각 노드의 **kube-proxy**가 iptables/IPVS 규칙으로 한다. [[Kubernetes]] 지도에서 kube-proxy를 "노드 네트워크 규칙 관리"라 했던 게 바로 이 일이다.

> 상태: #wip — 발견·로드밸런싱의 큰 그림. 다음 가지: Ingress(L7 라우팅, 여러 Service를 도메인·경로로 묶기), headless Service, 클러스터 내부 DNS.
