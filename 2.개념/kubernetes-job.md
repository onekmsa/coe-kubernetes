
# Job
Job 리소스를 생성하면 즉시 manifest에 명시한 Pod을 실행시킵니다. Pod의 프로세스가 성공적으로 종료되면 상태를 Complete로 변경하고 종료됩니다.  
실행 중 노드가 다운될 경우 가용한 다른 노드에 재 스케쥴링 됩니다.  
Pod의 프로세스가 실행 중 실패할 경우에는 manifest에 명시된 설정값(spec.template.spec.restartPolicy)에 따라 재실행 여부를 결정합니다.  

**RestartPolicy**
  - OnFailure
  - Never


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions : 5
  parallelism: 2
  activeDeadlineSeconds: 60
  backoffLimit: 6
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure # default restartPolicy인 Always는 Job에 적용 불가능하므로 OnFailure 나 Never를 명시적으로 주어야 합니다.
      containers:
      - name: main
        image: batch-job
```
## Completions

Job의 Pod을 여러번 실행시키고 싶을 때 사용합니다. Job의 Pod 실행이 끝나면 Pod을 다시 재 실행하여 명시된 카운트만큼 반복됩니다.   
```yaml
spec:
  completions : 5
```

#### Why?
리소스가 100이 필요한, 한 시간이 소요되는 Job이 있다면 중간에 실패할 가능성도 높고 실패하면 처음부터 다시 수행되어야 합니다.  
하지만, 10번 나눠서 실행하면 중간에 실패하더라도 중간부터 다시 실행할 수 있습니다.

#### Use case
코드 레벨에서 컨트롤 해야되나?


## Parallelism
Job의 Pod을 병렬로 동시에 수행합니다. 명시한 개수의 Pod 인스턴스를 동시에 실행시킵니다.

```yaml
spec:
  parallelism: 2
```

#### Why?
순서가 상관 없고 여러 번 수행되더라도 결과가 달라지지 않는 경우(idempotent) 병렬 수행을 통해 시간을 단축할 수 있습니다.

#### Use case
map reduce
파일 옮기기

## Job retry count / execution time

```yaml
spec:
  activeDeadlineSeconds: 60 # Job이 수행되는 시간. 명시한 시간을 넘어가도 종료되지 않으면 종료하고 상태를 실패로 변경한다
  backoffLimit: 6 # 실패로 상태를 변경하기 전에 명시 된 횟수만큼 retry. (default:6)
```
https://medium.com/google-cloud/kubernetes-running-background-tasks-with-batch-jobs-56482fbc853

# Cron Job
Job은 리소스 생성 후 바로 실행되지만 CronJob을 통해 특정한 시간에 특정 주기로 반복되도록 설정할 수 있습니다. 스케쥴링은 Cron Job 포맷을 따릅니다.

아래 예시의 경우 매 15분마다 수행됩니다.  
스케쥴링된 시간보다 Pod이 너무 늦게 실행되는 경우를 막기 위해 spec.startingDeadlineSecond에 데드라인 시간을 정할 수 있습니다.
10:30에 실행될 예정이나, 10:30:15까지 Pod이 시작하지 않는 경우 상태를 fail로 변경하고 Cron Job을 종료시킵니다.   
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
    name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: batch-job
```

## Concurrency Policy
.spec.concurrencyPolicy 필드를 통해 동시에 실행되는 Job을 제어할 수 있다.
- Allow(default)
- Forbid : Job이 실행될 시간에 이전 Job이 완료되지 않았으면 새로운 Job을 실행하지 않는다.
- Replace : Job이 실행될 시간에 이전 Job이 완료되지 않았으면 기존 Job을 새로운 Job으로 대체한다.

## Suspend
.spec.suspend

## Jobs History Limits
실행된 Job에 대한 히스토리 카운트 설정.
.spec.successfulJobsHistoryLimit (default 3)
.spec.failedJobsHistoryLimit (default 1)
