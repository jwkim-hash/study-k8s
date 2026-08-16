
컨테이너 요구사항 + 노드 자원 고려해서 어느 노드에 pod 배치할지 결정

마이크로 기반 시스템에서 컨테이너 수는 수백 이걸 일일히 노드에 나눠주는건 어려움
노드의 용량 상태라던가
노드가 죽으면? 다시 배치를 수동으로?
SSD, GPU등 이런것들까지 사람이 못함

자동화 기준
- 컨테이너 리소스 요구 충족
- 노드 자원들을 잘 활용
- 사람이 의도한 정책들도 반영

노드의 자원
- OS,커널
- kubelet, 컨테이너 런타임
- eviction을 위해 남여두는 여유
- Allocatable

이것저것, 시스템 몫을 뺀 Allocatable이 실제 스케줄러가 계산함

보통 pod request를 보고 들어감
cpu는 압축 가능
memory는 OOM 발생. 주의해야함

QoS
- guaranteed
- burstable
- besteffort

priorityclass -> 잘 안씀

실제 스케줄러가 노드를 고르는 기준
- Filtering
	- cpu, 메모리
	- hostport 비어있는지
	- 노드 정상?
	- nodeSelector 조건?
	- taint
- Scoring
	- 자원이 고르게 분산

Node Affinity - 사람이 직접 배치 정책에 개입
nodeSelector
nodeAffinity
	- `requiredDuringScedulingIgnoredDuringExecution` 조건을 만족하는 노드가 없으면 Pending
	- `preferredDuringSchedulingIgnoredDuringExecution` 조건 맞는 노드를 우선, 없어도 다른 노드에라도 배치

Pod Affinity, Anti-Affinity
Pod기준으로 어떤 Pod 옆에 놓을지 말지

Pod Affinity
특정 레이블을 가진 Pod가 있는곳에 배치
- 예: 웹 서버와 캐시를 같은 노드에 두면 네트워크 지연 줄어듦.
- 서로 자주 통신하는 컴포넌트 가까이

Pod Anti-Affinity
특정 레이블을 가진 Pod가 없는 곳에 배치
- 같은 앱의 replica들을 서로 다른 노드에 분산 -> 노드 하나 죽더라도 전부 죽는걸 방지
- 리소스를 많이 잡아먹는 Pod끼리 한 노드에 몰리는걸 방지

가용성 확보 목적

Pod affinity,anti-affinity는 계산비용이 큼

Taints, Tolerations
Taint - 노드에다가 붙힘
`kubectl taint nodes node01 gpu=true:NoSchedule` 

NoSchedule - 새 Pod를 배치하지 않음. 이미 떠있는 Pod는 그대로 둠
PreferNoSchedule - 가능하면 피하지만, 다른데 없으면 배치
NoExecute - 새 배치도 막고, 기존 Pod도 쫒아냄


Toleration - Pod 스펙에 달림
```
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

- 그 노드로 가고싶을때 nodeAffinity가 있어야 되고
- 거기서 거부당하지 않으려면 toleration
저 두 조합으로 특정 노드에 특정한 pod만 들어올 수 있음 (GPU 노드에 GPU pod만 배치)


Descheduler
배치 당시엔 최적이었지만 시간이 지날수록 이상해짐
- 노드가 추가됐는데 기존 Pod들은 옛 노드
- Pod가 특정 노드에만 쌓임
- 노드 레이블이 바뀌었는데 이미 떠있는 Pod 그대로
- 어떤 노드는 꽉 차있고, 어떤 노드는 비어있음

스케줄러는 이미 배치된 Pod를 다시 검토하지 않음

Descheduler? Pod를 배치하는게 아니라 쫓아냄
1. 여기 있으면 안되는 Pod를 찾고
2. 그 Pod가 Evict
3. Pod가 사라지면 ReplicaSet 이 새 Pod 생성
4. 새 Pod를 스케줄러가 현재 상황 기준으로 다시 배치
쫓아내면, 스케줄러가 다시 배치해주는 방식

CronJob 형태로 주기적으로 돌리는게 일반적이라 함. 상시 실행하면 클러스터가 요동칠 수 있음

