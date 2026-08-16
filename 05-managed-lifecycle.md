
컨테이너가 스스로의 수명주기를 직접 제어할 수 없는 환경이란?
일반 프로그램을 로컬에서 돌릴 때는
- 종료코드 리턴하고 끝내거나
- 무한 루프
- 어찌됐건 자가기 통제

쿠버네티스는 외부에서 강제로 신호를 보내는 것 같음
Warm up (시작 시)
Graceful Shutdown (종료 시)

시작 시점을 컨테이너가 못 정함
- 스케줄러가 지금 어떤 노드에 배치한다고 결정하면 그 순간 시작
- 컨테이너 코드 입장에선 자기가 언제 시작되는지 관여를 못함

종료 시점도 플랫폼이 관여함
- 노드 리소스 부족
- 롤링 업데이트로 새 버전 배포
- 스케일 다운
- 노드 자체가 문제 있어서 drain
- 사용자가 kubectl delete pod 실행
- liveness probe 실패

종료 시에 우선 `SIGTERM` 신호가 날아옴. 스스로 정리할 수 있는 유예를 줌 (30초, 뒤에 SIGKILL) 알아서 정리하고 죽던가 아니면 SIGKILL 이 들어옴 (강제적)
검색해보니,
새 요청 받기 중단, 처리 중인 요청 마저 완료, 자원 정리, 버퍼 flush, 종료
graceful shut

SIGKILL 상태에 들어오면
진행 중이던 작업은 그대로 유실
보통은 SIGTERM에서 스스로 정리하고 나가는 것이 정상, SIGKILL은 일종의 실패한 종료
kill -9

Poststart Hook
메인 컨테이너 프로세스와 병렬로 실행 (순서 보장 없음)
- 의도적으로 start up 상태를 지연
- 특정 조건 불충족시 종료시키는 역할


Prestop Hook
앱한테 곧 종료된다는 사실을 알려주는 블로킹 호출 (정리시간 확보)
블로킹 호출?
훅이 끝나야지만 다음 단계인 SIGTERM으로 넘어감. 

앱 코드를 수정할 수 없거나, 신호처리 로직을 넣기 어려운 경우 (타인이 만든 이미지를 쓰는경우)
이때 외부에서 정리 작업을 붙일 수 있음

`terminationGracePeriodSeconds` 시간 에 포함되기에 오래 잡으면 앱이 정리할 시간이 그만큼 줄어듦
실행 방식은은 exec, httpGet

[Pod 생성]
   ↓
Init Container 1 실행 → 완료
Init Container 2 실행 → 완료          ← 순차, 전부 성공해야 다음
   ↓
[메인 컨테이너 생성]
   ├─→ PID 1 프로세스 시작
   └─→ postStart 훅 (병렬)
   ↓
liveness probe 시작 → 실패하면 재시작
readiness probe 시작 → 통과해야 Service endpoint에 등록
   ↓
[정상 운영 - 트래픽 처리]
   ↓
[종료 결정]
   ├─→ endpoint에서 제거 (비동기)
   └─→ preStop 훅 실행 (블로킹)
   ↓
SIGTERM 전송
   ↓
grace period 대기 (기본 30초)
   ↓
SIGKILL (안 죽었으면)
   ↓
[Pod 삭제]
