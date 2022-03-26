---
title: "[Airflow] Executor Deep Dive 1-2. LocalExecutor.LimitedParallelism"
date: 2022-03-27 00:00:00
categories:
- airflow
tags: [airflow]

---

생각보다 UnlimitedParallelism이 길어졌습니다. 지난번 포스팅에서 LocalWorker 클래스가 초기화되면 LocalWorkerBase 클래스의 execute_work가 실행된다는 것까지 살펴봤습니다.





# class UnLimitedParallelism



## LocalWorkerBase.execute_work()

```python
def execute_work(self, key: TaskInstanceKey, command: CommandType) -> None:
        """
        Executes command received and stores result state in queue.

        :param key: the key to identify the task instance
        :param command: the command to execute
        """
        if key is None:
            return

        self.log.info("%s running %s", self.__class__.__name__, command)
        if settings.EXECUTE_TASKS_NEW_PYTHON_INTERPRETER:
            state = self._execute_work_in_subprocess(command)
        else:
            state = self._execute_work_in_fork(command)

        self.result_queue.put((key, state))


```

EXECUTE_TASKS_NEW_PYTHON_INTERPRETER 라는 설정값에 따라 자식 프로세스를 만드는 모듈이 subprocess / os로 구분됩니다. airflow 2.0 버전에서 추가된 설정사항입니다.

https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#execute-tasks-new-python-interpreter

> Should tasks be executed via forking of the parent process (“False”, the speedier option) or by spawning a new python process (“True” slow, but means plugin changes picked up by tasks straight away)



1 버전에 없던 os.fork()가 추가된 이유는 다음과 같습니다.

https://github.com/apache/airflow/commit/4839a5bc6ed7af7d0f836360e4ea3c6fd421e0fa

> ```
> Spawning a whole new python process and then re-loading all of Airflow
> is expensive. All though this time fades to insignificance for long
> running tasks, this delay gives a "bad" experience for new users when
> they are just trying out Airflow for the first time.
> 
> For the LocalExecutor this cuts the "queued time" down from 1.5s to 0.1s
> on average.
> ```

기본 설정값은 False입니다.  `subprocess.check_call()` 은 새로운 python process를 생성하는데, 그것보다 현재 프로세스를 복제해 바로 자식 프로세스로 내려보내는 fork()를 기본적으로 사용해 프로세스 로딩 시간을 줄이자는게 committer의 요지입니다. 

result_queue에 해당 task instance의 key와 command를 넣어줌으로써 해당 프로세스가 관리하는 task를 추가합니다.



이제 UnlimitedParallelism 클래스는 한바퀴 훑었습니다. LimitedParallelism 클래스를 보겠습니다.

<br/>

# class LimitedParallelism

```python
    class LimitedParallelism:
        """
        Implements LocalExecutor with limited parallelism using a task queue to
        coordinate work distribution.

        :param executor: the executor instance to implement.
        """

        def __init__(self, executor: 'LocalExecutor'):
            self.executor: 'LocalExecutor' = executor
            self.queue: Optional['Queue[ExecutorWorkType]'] = None

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

            self.executor.workers_used = len(self.executor.workers)

            for worker in self.executor.workers:
                worker.start()

        def execute_async(
            self,
            key: TaskInstanceKey,
            command: CommandType,
            queue: Optional[str] = None,
            executor_config: Optional[Any] = None,
        ) -> None:
            """
            Executes task asynchronously.

            :param key: the key to identify the task instance
            :param command: the command to execute
            :param queue: name of the queue
            :param executor_config: configuration for the executor
            """
            if not self.queue:
                raise AirflowException(NOT_STARTED_MESSAGE)
            self.queue.put((key, command))

        def sync(self):
            """Sync will get called periodically by the heartbeat method."""
            while True:
                try:
                    results = self.executor.result_queue.get_nowait()
                    try:
                        self.executor.change_state(*results)
                    finally:
                        self.executor.result_queue.task_done()
                except Empty:
                    break

        def end(self):
            """Ends the executor. Sends the poison pill to all workers."""
            for _ in self.executor.workers:
                self.queue.put((None, None))

            # Wait for commands to finish
            self.queue.join()
            self.executor.sync()
```

구조는 UnlimitedParallelism와 크게 다르지 않습니다. start 메서드만 보면 UnLimitedParallelism과 차이점이 확연히 보일 것입니다.



## def start()

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

        self.executor.workers_used = len(self.executor.workers)

        for worker in self.executor.workers:
          worker.start()
```



다른 것들은 크게 중요하지 않고, `self.executor.workers` 를 보겠습니다. UnLimitedParallelism에서는 LocalWorker 인스턴스를 제한 없이 생성하고 `workers_used`와 `workers_active` 로 증감기록만 해줬습니다. 그런데 LimitedParallelism의 `self.executor.workers`는  QueuedLocalWorker 인스턴스를 `self.executor.parallelism` 만큼만 생성해 List로 가지고 있습니다. 아무리 task가 많아져도 worker의 갯수의 제한이 있기 때문에 병렬처리 또한 제한적으로 실행될 것입니다.

물론 다른 구현사항들도 차이점이 있겠지만 이 내용을 기반으로 실행하는 방법이 달라지는 수준이기 때문에 UnLimitedParallelism과 중복 아닌 중복이 많이 됩니다. 생략하겠습니다 :)

<br/>

# 결론

클래스 하나를 보고 분석하는데에도 꽤 많은 시간이 걸렸습니다. 그렇지만 이득은 확실합니다.

LocalExecutor를 상용에 올릴 일은 없겠지만, 만약 문제가 생긴다면 이렇게 생각할 수 있게 되었습니다.

1. task가 Queueing도 되지 않네? 그럼 worker의 execute_work() 이전부터 살펴봐야겠다!
2. LimitedParallelism에서 result_queue까지 제대로 생성되었네?
3. `self.executor.parallelism` 이 -1로 설정되는 오류가 있었네? `for _ in range(-1)` 일테니 workers 자체가 생성되지 않았을테고, 그래서 task가 아예 queueing도 안되고 scheduled 상태로 멈춰있었구나!



내부 코드를 알게 되니 단지 UI로만 접하던 airflow를 좀 더 잘 알게 되었습니다.

다음은 실무에서도 많이 쓰는 CeleryExecutor 해부입니다!!





















