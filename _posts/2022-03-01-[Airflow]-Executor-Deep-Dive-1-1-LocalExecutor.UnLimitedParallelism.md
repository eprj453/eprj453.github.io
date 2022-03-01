---
title: "[Airflow] Executor Deep Dive 1-1. LocalExecutor.UnLimitedParallelism"
date: 2022-03-01 00:00:00
categories:
- airflow
tags: [airflow]

---



## UnLimitedParallelism

```python
    class UnlimitedParallelism:
        """
        Implements LocalExecutor with unlimited parallelism, starting one process
        per each command to execute.

        :param executor: the executor instance to implement.
        """
```

프로세스가 들어오면 들어오는대로 하나씩 하나씩 실행시킨다는 docstring이 있습니다. 조금 길고 귀찮긴 하지만 포스팅의 목적에 맞게 init 메서드를 제외한 모든 메서드를 까서 살펴보겠습니다.

<br/>

### def start

```python
        def start(self) -> None:
            """Starts the executor."""
            self.executor.workers_used = 0
            self.executor.workers_active = 0
```

executor가 시작되면 변수 2개가 초기화됩니다. 사용된 worker와 실행중인 worker의 상태를 체크하기 위한 용도인것 같습니다.

<br/>

### def execute_async

```python
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
            :param queue: Name of the queue
            :param executor_config: configuration for the executor
            """
            if not self.executor.result_queue:
                raise AirflowException(NOT_STARTED_MESSAGE)
            local_worker = LocalWorker(self.executor.result_queue, key=key, command=command)
            self.executor.workers_used += 1
            self.executor.workers_active += 1
            local_worker.start()
```

이름만 봐도 비동기로 task를 실행시키는, 즉 subprocess를 만드는 메서드임을 알 수 있습니다. 

위의 LocalExecutor 설명에서는 언급하지 않았지만 하나의 process는 하나의 task만을 실행시키고, 각각 독립적인 process이기 때문에 queue나 pipe와 같은 IPC 기법이 없는 이상 task간의 통신은 원칙적으로 불가능합니다. 

단, airflow는 전역 저장소 형태의 xcom을 이용해 task 간에도 데이터를 주고받을 수 있는 IPC 기법을 제공하고 있습니다.



제일 먼저 result_queue가 없다면 Exception을 일으킵니다. result_queue는 LocalExecutor의 start 메서드에서 생성되기 때문에 당연히 없다면 오류입니다.

그 다음에는 local_worker를 생성합니다. queue와 task의 고유 key, 그리고 task를 실행할 command를 전달합니다. executor.workers_used와 executor.workers_active에 각각 1을 더해줍니다. 

local_worker의 start 메서드는 파이썬 빌트인 라이브러리인 Process의 start입니다. LocalWorker <- LocalWorkerBase <- Process 순으로 상속관계를 가지고 있습니다. 여기서 인자로 주어진 Key와 command를 기반으로 task 프로세스가 실행되는 것을 확인할 수 있습니다. Worker Process를 메인 프로세스로 하고, Task를 자식 프로세스로 할당합니다.

<br/>

### def sync

```python
      # pylint: enable=unused-arndment # pragma: no cover
      def sync(self) -> None:
        """Sync will get called periodically by the heartbeat method."""
        if not self.executor.result_queue:
          raise AirflowException("Executor should be started first")
          while not self.executor.result_queue.empty():
            results = self.executor.result_queue.get()
            self.executor.change_state(*results)
            self.executor.workers_active -= 1
```

heartbeat에 의해 주기적으로 호출되는 메서드라고 docstring에 명시되어 있습니다. result_queue가 없다면 LocalExecutor의 start 메서드가 아직 실행되지 않은것이기 때문에 Exception을 일으킵니다.

result_queue가 완전히 빌 때까지 queue에 담겨있는 프로세스 상태를 가져와서 상태를 변경합니다. `change_state()` 메서드는 BaseExecutor 클래스에 있는데, 주어진 Task 정보에 따라 executor에서 실행중인 task를 없애거나 상태를 변경합니다. 이에 대한 건 BaseExecutor의 코드를 보면 이해에 도움이 될 것입니다.

https://github.com/apache/airflow/blob/main/airflow/executors/base_executor.py

<br/>

### def end

```python
        def end(self) -> None:
            """
            This method is called when the caller is done submitting job and
            wants to wait synchronously for the job submitted previously to be
            all done.
            """
            while self.executor.workers_active > 0:
                self.executor.sync()
```

`self.executor.workers_active` 가 0 이하가 될 때까지 sync 메서드를 호출합니다. sync 메서드의 while문에서 `self.executor.workers_active` 를 1씩 깎는 것을 확인할 수 있습니다.



<br/>

여기까지 UnLimitedParallelism 클래스의 모든 메서드를 봤습니다. 그런데 가장 중요한 기능이 전혀 보이지 않습니다. Task Process를 실행시키는 메서드입니다. UnLimitedParallelism 클래스의 메서드들은 Worker 프로세스를 띄우고 상태를 확인하는 메서드들뿐입니다. task process가 실행되어야 result_queue에도 task 정보가 찰텐데 그런 것도 코드에서 전혀 보이지 않습니다.



이를 확인하기 위해서는 LocalWorker와 그 부모 클래스 LocalWorkerBase를 볼 필요가 있습니다.

<br/>

# LocalWorker

```python
class LocalWorker(LocalWorkerBase):
    """
    Local worker that executes the task.

    :param result_queue: queue where results of the tasks are put.
    :param key: key identifying task instance
    :param command: Command to execute
    """

    def __init__(
        self, result_queue: 'Queue[TaskInstanceStateType]', key: TaskInstanceKey, command: CommandType
    ):
        super().__init__(result_queue)
        self.key: TaskInstanceKey = key
        self.command: CommandType = command

    def do_work(self) -> None:
        self.execute_work(key=self.key, command=self.command)
```

여긴 별게 없습니다. 부모 클래스 상속받아서 부모 메서드 execute_work를 실행하는 것 말고는 별다른 기능이 없네요. `UnlimitedParallelism.execute_async()` 에 있는 `local_worker.start()`의 `start()` 메서드도 LocalWorker 클래스에서는 찾아볼 수 없습니다.

부모 클래스인 LocalWorkerBase를 봐야겠습니다.

<br/>

# LocalWorkerBase

```python
class LocalWorkerBase(Process, LoggingMixin):
    """
    LocalWorkerBase implementation to run airflow commands. Executes the given
    command and puts the result into a result queue when done, terminating execution.

    :param result_queue: the queue to store result state
    """

    def __init__(self, result_queue: 'Queue[TaskInstanceStateType]'):
        super().__init__(target=self.do_work)
        self.daemon: bool = True
        self.result_queue: 'Queue[TaskInstanceStateType]' = result_queue

    def run(self):
        # We know we've just started a new process, so lets disconnect from the metadata db now
        settings.engine.pool.dispose()
        settings.engine.dispose()
        return super().run()

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

    def _execute_work_in_subprocess(self, command: CommandType) -> str:
        try:
            subprocess.check_call(command, close_fds=True)
            return State.SUCCESS
        except subprocess.CalledProcessError as e:
            self.log.error("Failed to execute task %s.", str(e))
            return State.FAILED

    def _execute_work_in_fork(self, command: CommandType) -> str:
        pid = os.fork()
        if pid:
            # In parent, wait for the child
            pid, ret = os.waitpid(pid, 0)
            return State.SUCCESS if ret == 0 else State.FAILED

        from airflow.sentry import Sentry

        ret = 1
        try:
            import signal

            from airflow.cli.cli_parser import get_parser

            signal.signal(signal.SIGINT, signal.SIG_DFL)
            signal.signal(signal.SIGTERM, signal.SIG_DFL)
            signal.signal(signal.SIGUSR2, signal.SIG_DFL)

            parser = get_parser()
            # [1:] - remove "airflow" from the start of the command
            args = parser.parse_args(command[1:])
            args.shut_down_logging = False

            setproctitle(f"airflow task supervisor: {command}")

            args.func(args)
            ret = 0
            return State.SUCCESS
        except Exception as e:  # pylint: disable=broad-except
            self.log.error("Failed to execute task %s.", str(e))
        finally:
            Sentry.flush()
            logging.shutdown()
            os._exit(ret)  # pylint: disable=protected-access
            raise RuntimeError('unreachable -- keep mypy happy')

    @abstractmethod
    def do_work(self):
        """Called in the subprocess and should then execute tasks"""
        raise NotImplementedError()
```

airflow 커맨드를 실행하기 위한 클래스입니다. 파이썬 빌트인 클래스인 Process를 상속하고 있네요. target을 do_work로 지정했습니다. LocalWorker의 do_work는 execute_work를 실행시키기 때문에, 이 부분이 프로세스 실행과 연관이 있는 것은 맞아보입니다.

target은 BaseProcess 클래스의 초기화 변수입니다. `BaseProcess.run()`에서 target이 존재할 경우 실행됩니다.

```python
    def run(self):
        '''
        Method to be run in sub-process; can be overridden in sub-class
        '''
        if self._target:
            self._target(*self._args, **self._kwargs)
```



LocalWorkerBase의 run이 실행되면, super().run()에 의해 BaseProcess.run()이 실행되고 결국 do_work가 실행됩니다. LocalWorker가 초기화되면 LocalWorkerBase가 초기화되고, target이 지정되어 있기 때문에 `BaseProcess.run()` 에 의해 target이 실행되는 구간이 내부에 있을거라 추측합니다. (이 부분은 코드 내부에서 찾을 수 없었습니다. ㅜㅜ)



어쨌든 LocalWorkerBase 클래스가 초기화되면 자동으로 do_work()를 실행하고, 이를 상속받고 있는 LocalWorker 클래스의 `do_work()`는 execute_work()를 실행하기 때문에, 결국 LocalWorker 클래스가 초기화되면 LocalWorkerBase 클래스의 execute_work가 실행됩니다.



execute_work()와 그 내부 메서드만 보면 result_queue와 OS 단계에서 프로세스를 생성하는 코드도 있습니다. 이 부분만 보면 LocalExecutor 내부는 한 번 훝은게 아닌가 생각합니다.

그런데 쓰다보니 너무 길어져서, LocalExecutor는 다음 포스팅에서 마무리하도록 하겠습니다.









