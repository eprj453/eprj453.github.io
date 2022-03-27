---
title: "[Airflow] Executor Deep Dive 1-1. LocalExecutor.UnLimitedParallelism"
date: 2022-02-24 00:00:00
categories:
- airflow
tags: [airflow]

---





먼저 `__init__` 입니다. Process를 상속받기 위한 super 메서드가 있고, result_queue를 초기화하네요. 

result_queue에는 각 Task Instance의 상태가 담겨져 있고 queue니까 FIFO 방식으로 먼저 실행되는 Task가 먼저 나가서 마지막에는 airflow webserver UI의 상태도 변경시키게 될겁니다. result_queue의 역할을 여기서 확인할 수 있습니다.



`run()` 입니다. Process 클래스에 있는 run을 실행시키고, 이는 필시 프로세스를 실행시키는 API가 호출될 것입니다. BaseProcess 클래스의 run 메서드입니다.

```python
def run(self):
  '''
        Method to be run in sub-process; can be overridden in sub-class
        '''
  if self._target:
    self._target(*self._args, **self._kwargs)
```

self.target이 지정되어 있는 경우 target으로 지정되어 있는 callable 객체, 즉 파이썬 1급 객체인 메서드를 실행시킵니다. 그렇다면 LocalWorkerBase 클래스의 target 메서드는? `do_work()` 로 지정되어 있습니다.

중간 정리를 해보자면 이렇습니다.

1. LocalExecutor는 내부 클래스인 UnLimitedParallelism / LimitedParallelism으로 구현을 한 뒤 실행
2. UnLimitedParallelism 클래스는 LocalWorkerBase를 상속하고 있고, LocalWorkerBase 클래스의 do_work 메서드를 실행 (super().__init__(target=self.do_work))
3. LocalWorkerBase 클래스에서 프로세스를 실행하기 위한 Process 클래스의 `run()` 메서드는 do_work를 타겟 메서드로 지정하고 있고, 그렇기 때문에 LocalWorker 클래스에서도 프로세스가 실행되면 `run()` 메서드가 실행
4. LocalWorker 클래스의 `run()` 메서드는 LocalWorkerBase 클래스의 `execute_work()` 메서드를 실행



자, LocalWorker가 프로세스를 실행하면 결국 LocalWorkerBase의 `execute_work()` 메서드가 실행된다는 것까지 파악했습니다. 이 메서드만 따로 떼서 보겠습니다.

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

`EXECUTE_TASKS_NEW_PYTHON_INTERPRETER` 설정값에 따라서 실행하는 방식이 달라집니다. [공식문서](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#execute-tasks-new-python-interpreter)에서는 EXECUTE_TASKS_NEW_PYTHON_INTERPRETER에 대해 다음과 같이 설명하고 있습니다. 

> 기본 값은 False입니다. False인 경우 상위 프로세스의 분기(fork)로 프로세스가 생성되고 속도가 빠릅니다. True의 경우 새로운 python process가 생성됩니다. 느리지만 plugins의 변경사항이 즉시 반영됩니다.



subprocess 라이브러리와 os.fork()를 사용해 프로세스를 생성하는 2가지 방법이 차이가 있다는걸 알 수 있습니다. 두 메서드 모두 자식 프로세스를 생성하는 메서드인 것은 같으나, 분명 os 라이브러리와 subprocess 라이브러리의 생성 방식은 차이가 있을 것입니다. 이 부분은 따로 OS 레벨에서 파헤쳐보는 포스팅을 해보려고 합니다. 

`_execute_work_in_subprocess / _execute_work_in_fork` 두 메서드 모두 프로세스 실행에 성공한다면 성공 상태를 반환합니다. 즉, result_queue에는 실행한 프로세스의 상태를 반환받은 뒤, result_queue에 넣어주면 `execute_work()` 메서드의 기능은 끝입니다. 











