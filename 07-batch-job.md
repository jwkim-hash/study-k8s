
- 백업
- 대용량 파일 변환
같이 한번 실행되고 끝나야 되는 것들에 사용. 끝나면 다시 띄우지 않음

Pod만 쓰면 노드가 죽었을때 이를 실행 해줄게 없음.
ReplicaSet을 쓰면
pod가 잘 성공해서 종료해도 다시 살려버림

완료를 다루는 컨트롤러

- 실패하면 재시도
- 성공하면 더 이상 만들지 않음
- 여러개를 병렬로 실행 가능
- 끝난 뒤 결과와 로그 확인 가능

```
sepc:
  template:
    spec:
      restartPolicy: OnFailure
```
OnFailure - 실패시 같은 노드에서 컨테이너를 재시작
Never - 재시작하지 않고, Job이 새 Pod를 만들어서 재시도

- completions: 몇 번 성공해야 Job이 완료되는지
- parallelism: 동시에 몇개 Pod?
- backoffLimit: 몇 번까지 재시도

1. Single Pod Job
```
completions: 1
parallelism: 1
```
Pod 하나 실행하고 성공하면 끝,
데이터 마이그레이션, 단발성 스크립트

2. Fixed Completion Count Job
```
completions: 5
parallelism: 1
```
정해진 횟수만큼 순차 실행. x번 성공할때까지 하나씩

3. Work Queue Job
```
completions: 1
parallelism: 5
```
여러 Pod가 동시에 돌면서 공유 큐를 소비하는 형태.
Pod 하나라도 성공하면 나머지도 곧 종료되고 Job이 완료. -> 이부분 좀 이해가 안됨.어쩔 때 쓰는지도

4. Parallel Fixed Completion Count Job
```
completions: 10
parallelism: 3
```
정해진 횟수를 병렬로 처리, 총 10번 성공이면 동시에 3개씩 돌아감

정리하면,
Job은 완료되는 작업이라는 개념을 알고있는 리소스

나중에 질문.
Work Queue 패턴 잘 모르겠음. 외부 큐를 통해 스스로 조절?

정리를 잘 해야됨
Job이 완료되도 Pod가 자동으로 사라지지 않음. 로그 확인할 수 있지만 Job을 많이 돌리면 Pod가 쌓여서 etcd에 부담

멱등성을 고려해야함.
실패시 재시도를 할텐데, 같은 작업이 두번실행될 수 있다는 뜻임.
Job Podeh SIGTERM받을 수 있고, 중간에 끊겼을때 데이터 남는거 고려해야됨.

