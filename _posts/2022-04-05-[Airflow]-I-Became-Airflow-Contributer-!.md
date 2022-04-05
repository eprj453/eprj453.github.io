---
title: "[Airflow] Airflow 컨트리뷰터가 되었습니다!"
date: 2022-04-05 00:00:00
categories:
- airflow
tags: [airflow]

---

이전 포스팅 [[Airflow] Executor Deep Dive 1-2. LocalExecutor.LimitedParallelism](https://eprj453.github.io/airflow/2022/03/26/Airflow-Executor-Deep-Dive-1-2-LocalExecutor.LimitedParallelism/)에서, LimitedParallelism 클래스가 시작되면 worker가 parallelism 옵션에서 지정한 숫자(`self.executor.parallelism`)만큼 생성되는 것을 보았습니다.

```python
def start(self) -> None:
            """Starts limited parallelism implementation."""
            if not self.executor.manager:
                raise AirflowException(NOT_STARTED_MESSAGE)
            self.queue = self.executor.manager.Queue()
            if not self.executor.result_queue:
                raise AirflowException(NOT_STARTED_MESSAGE)
            self.executor.workers = [
                QueuedLocalWorker(self.queue, self.executor.result_queue)
                for _ in range(self.executor.parallelism)
            ]
```

<br/>

그런데 여기서 self.executor.parallelism이 음수라면 어떻게 될까요?

airflow 빌드에는 에러가 발생하지 않습니다. DAG를 실행시켜도 에러는 발생하지 않습니다. 단, LocalExecutor를 사용하는 모든 DAG의 Task가 Scheduled 상태에서 멈춰있습니다. Scheduler에서 task를 당겨갈 Worker가 생성되지 않았기 때문입니다.



물론 에러가 발생하지 않는 것이 가장 좋지만, 만약 에러 상황이라면 확실하게 에러를 발생시키는 것이 맞다고 생각합니다. 그렇지 않으면 개발자는 에러 상황을 인지하지 못한 채 서비스가 돌아갈 것이고 이는 더 큰 문제로 되돌아올 수 있기 때문입니다.

이렇게 거창하게 말하긴 했지만... 결론적으로 제가 airflow에 커밋한 내용은 LocalExecutor가 초기화될 때 해당 옵션이 음수라면 AirflowException을 발생시키는 코드를 추가한 것이 전부입니다. 코드로 치면... 2줄이네요 ㅎㅎ

Commiter의 피드백을 받아 총 2번의 커밋을 했고 제 소스가 main 브랜치에 병합됐습니다. 

[https://github.com/apache/airflow/pull/22711](https://github.com/apache/airflow/pull/22711)

그래도 ETL 업계에서 공룡 프레임워크인 Airflow에 두 줄이라도 기여했다는 뿌듯함, 전 세계인이 사용하는 오픈소스에 컨트리뷰터라는 자신감도 생겨서 기분이 좋습니다.

<img width="1043" alt="image" src="https://user-images.githubusercontent.com/52685258/161777402-e87a211c-6c53-4a31-a617-a81a928bcd1b.png">























