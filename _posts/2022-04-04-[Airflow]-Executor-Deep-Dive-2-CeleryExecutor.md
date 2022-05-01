---
title: "[Airflow] Executor Deep Dive 2. CeleryExecutor"
date: 2022-05-01 00:00:00
categories:
- airflow
tags: [airflow]

---



Executor Deep Dive 2번째 파트 CeleryExecutor입니다. 사실 LocalExecutor는 production 환경에서 사용을 권장하지 않습니다. 그러나 CeleryExecutor부터는 production 환경 사용도 권장하고 있습니다.

> CeleryExecutor is recommended for production use of Airflow. It allows
> distributing the execution of task instances to multiple worker nodes. 
>
> Celery is a simple, flexible and reliable distributed system to process
> vast amounts of messages, while providing operations with the tools
> required to maintain such a system.
>
> -- CeleryExecutor Docstring



LocalExecutor보다 내용이 조금 많습니다. 달려보겠습니다!

<br/>

# Class CeleryExecutor

<img width="946" alt="image" src="https://user-images.githubusercontent.com/52685258/166137148-ae2cc30a-2352-458f-8411-b02e7218d39e.png">

CeleryExecutor는 LocalExecutor처럼 Worker 프로세스에 task가 하나씩 subprocess로 붙는 방식과는 조금 다릅니다. 스케줄러는 Queue로 Task를 보내고, 쌓여있는 Task들을 떠있는 Worker들이 경쟁적으로 가져가 자신의 subprocess로 만드는 모습을 볼 수 있습니다.

<br/>

## init

```python
class CeleryExecutor(BaseExecutor):
    def __init__(self):
        super().__init__()

        # Celery doesn't support bulk sending the tasks (which can become a bottleneck on bigger clusters)
        # so we use a multiprocessing pool to speed this up.
        # How many worker processes are created for checking celery task state.
        self._sync_parallelism = conf.getint('celery', 'SYNC_PARALLELISM')
        if self._sync_parallelism == 0:
            self._sync_parallelism = max(1, cpu_count() - 1)
        self.bulk_state_fetcher = BulkStateFetcher(self._sync_parallelism)
        self.tasks = {}
        # Mapping of tasks we've adopted, ordered by the earliest date they timeout
        self.adopted_task_timeouts: Dict[TaskInstanceKey, datetime.datetime] = OrderedDict()
        self.task_adoption_timeout = datetime.timedelta(
            seconds=conf.getint('celery', 'task_adoption_timeout', fallback=600)
        )
        self.task_publish_retries: Dict[TaskInstanceKey, int] = OrderedDict()
        self.task_publish_max_retries = conf.getint('celery', 'task_publish_max_retries', fallback=3)
```

CeleryExecutor 역시 BaseExecutor를 부모 클래스로 가지고 있습니다. super()로 부모 클래스의 속성 또한 초기화시켜 사용하는 모습도 볼 수 있습니다.

`self._sync_parallelism` 변수는 각 Worker가 병렬로 실행할 수 있는 최대 subprocess 수를 결정합니다. default가 32개인걸로 알고 있는데, 각자 사용환경마다 다르겠지만 요즘 사용하는 일반적인 서버에서는 32개는 너무 적지 않나 생각합니다. Worker를 너무 적게 띄워 부모 프로세스가 너무 적게 할당되어 있지 않는 이상은 좀 더 여유롭게 갯수를 잡는 것이 사용하는 측면에서도 좋을 것 같습니다. 만약 0이 할당되어 있다면 자동으로 1개 or 코어 수 -1개 중 더 큰 갯수를 선택합니다. 코어 수에서 1개를 빼는 것은 아직 이유는 잘 모르겠으나, 최대 코어 수를 사용하면 문제가 될만한 상황이 뒤에 나오지 않을까 생각합니다.

나머지 인스턴스 변수들은 실행에 필요한 사항들을 담고 있습니다. 이것들을 전부 살펴보지는 않겠습니다.

이제 task를 실제로 실행시키는 메서드 execute_async를 보겠습니다.

<br/>

## def execute_async()

```python
def execute_async(
    self,
    key: TaskInstanceKey,
    command: CommandType,
    queue: Optional[str] = None,
    executor_config: Optional[Any] = None,
):
    """Do not allow async execution for Celery executor."""
    raise AirflowException("No Async execution for Celery executor.")
```

CeleryExecutor는 LocalExecutor와 달리 execute_async를 사용하지 않습니다. LocalExecutor와는 달리 각 task를 비동기로 Worker에 할당하지 않는 이유는 뭘까요?

LocalExecutor와 CeleryExecutor의 아키텍처를 비교해보겠습니다.

![image](https://user-images.githubusercontent.com/52685258/166137194-faf04667-72f4-48da-b89b-c7615fb72607.png)

<img width="946" alt="image" src="https://user-images.githubusercontent.com/52685258/166137148-ae2cc30a-2352-458f-8411-b02e7218d39e.png">

LocalExecutor와 달리 CeleryExecutor는 N개의 Worker가 각각 subprocess를 가질 수 있습니다. 또한 scheduler가 subprocess를 할당한다기보다는, 각 Worker가 queue에 들어있는 task를 polling해가는 형태의 구조를 갖고 있는 것을 볼 수 있습니다(`Executor가 task를 실행시킨다는 표현도 틀린 표현은 아닙니다`). queue가 꼭 1개일거란 보장도 없습니다.

n개의 Executor가 각자 독립적으로 queue에서 task를 빼가는 방식인데, 비동기 방식으로 구동된다면 어떻게 될까요?

- 1번 executor가 queue에서 Task를 인지하고 실행합니다.
- task의 상태는 실행되었다고 DB에 기록합니다.
- 변경된 task는 queue에서 빠지게 되고, 그 다음 task를 executor가 실행합니다. 어떤 executor일지는 따로 설정하지 않으면 알 수 없습니다.

여기서 비동기 방식을 지원한다면 task의 상태가 변경되기 전에 다른 Executor에서 동일한 task를 실행할 수 있게 됩니다. queue의 F
IFO 방식이 원활하게 보장되려면 task의 상태 변경이 된 이후에 그 다음 task를 실행한다는 순서가 보장되어야 합니다.

그렇기 때문에 task 할당으로 Queue를 사용하는 CeleryExecutor는 비동기 방식으로 task를 실행하지 않습니다.

<br/>

Queue에는 어떤 것들이 들어갈까요?

```python
# Task instance that is sent over Celery queues
# TaskInstanceKey, Command, queue_name, CallableTask
TaskInstanceInCelery = Tuple[TaskInstanceKey, CommandType, Optional[str], Task]
```

Queue에는 Tuple이 들어가고,`task instance key / command, queue_name, task(Callable)` 이 들어있습니다. 여기에서는

- task가 실행 가능한 무언가(command, python function..)이겠구나
- queue 이름을 지정할 수 있는걸로 보아 queue는 여러개일수도 있겠구나

정도를 생각해볼 수 있겠습니다.



중간 정리를 해보자면,

- 사용자가 Webserver에 올린 Task를 DB에서 읽어들인다.
- scheduler는 실행 전 상태인 Task를 DB에서 읽어 queue에 넣는다.
- queue에 있는 task를 worker가 실행시킨다.
- task 상태를 변경한다.



CeleryExecutor의 task 실행 순서는 이렇게 정리해볼 수 있습니다.

DB에서 실행 전 상태의 task를 스캔하는건 모든 Executor의 공통사항일테니 넘어가고, queue에 task를 append하는 메서드부터 살펴보겠습니다.



## CeleryExecutor.trigger_tasks()

```python
def trigger_tasks(self, open_slots: int) -> None:
    """
    Overwrite trigger_tasks function from BaseExecutor

    :param open_slots: Number of open slots
    :return:
    """
    sorted_queue = self.order_queued_tasks_by_priority()

    task_tuples_to_send: List[TaskInstanceInCelery] = []

    for _ in range(min(open_slots, len(self.queued_tasks))):
        key, (command, _, queue, _) = sorted_queue.pop(0)
        task_tuple = (key, command, queue, execute_command)
        task_tuples_to_send.append(task_tuple)
        if key not in self.task_publish_retries:
            self.task_publish_retries[key] = 1

    if task_tuples_to_send:
        self._process_tasks(task_tuples_to_send)
```

queue는 특정 우선순위를 가지고 Out 순서가 결정되는 우선순위 Queue인가봅니다. 정렬 기준은 무엇이며,  `self.order_queued_tasks_by_priority()` 는 어디에 있을까요?



## BaseExecutor.order_queued_tasks_by_priority()

```python
def order_queued_tasks_by_priority(self) -> List[Tuple[TaskInstanceKey, QueuedTaskInstanceType]]:
    """
    Orders the queued tasks by priority.

    :return: List of tuples from the queued_tasks according to the priority.
    """
    return sorted(
        self.queued_tasks.items(),
        key=lambda x: x[1][1],
        reverse=True,
    )
```

Queue는 부모 클래스인 BaseExecutor에 있습니다. CeleryExecutor 말고도 Queue 방식으로 task를 할당하는 Executor들이 꽤 있나봅니다.

queued_tasks는 BaseExecutor의 인스턴스가 가지고 있습니다. items()로 key와 value를 가져오는걸 보니 Dict 형태로 task를 보관하고 있음을 알 수 있습니다.

정렬 key는 아이템의 `x[1][1]` 입니다. CeleryExecutor의 queue 아이템은 `(key, command, queue, execute_command)` 이니, command 부분에 정렬할 수 있는 무언가가 있을거라 추측할 수 있습니다.



`trigger_tasks()` 에서는 최소 open_slots, 최대 `len(self.queued_tasks)` 만큼 for loop를 돌며  `sorted_queue` 에서 아이템을 빼 task_tuples_to_send라는 빈 list에 task를 집어넣고 실행시킵니다.



그런데 이건 queueing이 완료된 task들을 실행시키는 메서드이고, queueing이 되는 곳은 다른 곳인것 같습니다. `self.order_queued_tasks_by_priority()` 에서 sorted_queue를 가져왔고, `self.order_queued_tasks_by_priority()` 는 `self.queued_tasks` 를 가져왔으니 `self.queued_tasks` 딕셔너리에 Task를 넣어주는 곳이 어딘지 보면 될 것 같습니다.

<br/>

## BaseExecutor.queue_command()

```python
def queue_command(
    self,
    task_instance: TaskInstance,
    command: CommandType,
    priority: int = 1,
    queue: Optional[str] = None,
):
    """Queues command to task"""
    if task_instance.key not in self.queued_tasks and task_instance.key not in self.running:
        self.log.info("Adding to queue: %s", command)
        self.queued_tasks[task_instance.key] = (command, priority, queue, task_instance)
    else:
        self.log.error("could not queue task %s", task_instance.key)

def queue_task_instance(
    self,
    task_instance: TaskInstance,
    mark_success: bool = False,
    pickle_id: Optional[str] = None,
    ignore_all_deps: bool = False,
    ignore_depends_on_past: bool = False,
    ignore_task_deps: bool = False,
    ignore_ti_state: bool = False,
    pool: Optional[str] = None,
    cfg_path: Optional[str] = None,
  ) -> None:
    """Queues task instance."""
    pool = pool or task_instance.pool

    # TODO (edgarRd): AIRFLOW-1985:
    # cfg_path is needed to propagate the config values if using impersonation
    # (run_as_user), given that there are different code paths running tasks.
    # For a long term solution we need to address AIRFLOW-1986
    command_list_to_run = task_instance.command_as_list(
      local=True,
      mark_success=mark_success,
      ignore_all_deps=ignore_all_deps,
      ignore_depends_on_past=ignore_depends_on_past,
      ignore_task_deps=ignore_task_deps,
      ignore_ti_state=ignore_ti_state,
      pool=pool,
      pickle_id=pickle_id,
      cfg_path=cfg_path,
    )
    self.log.debug("created command %s", command_list_to_run)
    self.queue_command(
      task_instance,
      command_list_to_run,
      priority=task_instance.task.priority_weight_total,
      queue=task_instance.task.queue,
    )

```

CeleryExecutor 환경 airflow에서 DAG를 실행시켰을 때 로그에 `Adding to queue: ~ ` 가 뜨는걸 많이 봤었는데, 여기 있었네요! `queue_task_instance()` 메서드가 실행되면 `queue_command()`에 의해 task 인스턴스가 Executor의 인스턴스 변수인 queued_tasks에 저장됩니다.



그런데 `queue_task_instance()` 는 그럼 어디에서 실행되고 있는걸까요? 여태까지의 메서드들은 전부 실제 기능을 수행하고 있었고, 이것들을 모니터링하며 계속 실행시킬 주체는 아직 발견하지 못했습니다.

<br/>

## airflow/www/views.py/def run()

```python
@action_logging
def run(self):
    """Runs Task Instance."""
    dag_id = request.form.get('dag_id')
    task_id = request.form.get('task_id')
    origin = get_safe_url(request.form.get('origin'))
    dag = current_app.dag_bag.get_dag(dag_id)
    task = dag.get_task(task_id)

    execution_date = request.form.get('execution_date')
    execution_date = timezone.parse(execution_date)
    ignore_all_deps = request.form.get('ignore_all_deps') == "true"
    ignore_task_deps = request.form.get('ignore_task_deps') == "true"
    ignore_ti_state = request.form.get('ignore_ti_state') == "true"

    executor = ExecutorLoader.get_default_executor()
    valid_celery_config = False
    valid_kubernetes_config = False
    valid_celery_kubernetes_config = False

    ...
    ...
    ...

    executor.start()
    executor.queue_task_instance(
        ti,
        ignore_all_deps=ignore_all_deps,
        ignore_task_deps=ignore_task_deps,
        ignore_ti_state=ignore_ti_state,
    )
    executor.heartbeat()
    flash(f"Sent {ti} to the message queue, it should start any moment now.")
    return redirect(origin)


```

찾았습니다. 생각보다 긴 시간을 헤메다 발견했는데, 위의 아키텍처 그림을 꽤 많이 봤었음에도 Webserver에서 DB로 화살표를 보내고 있는걸 단번에 생각해내지 못했습니다. 역시 머리가 나쁘면 몸이 고생합니다.

airflow의 Web은 flask로 구현되어 있습니다. flask app에서 HTTP request를 받으면 executor에서 queue_task_instance를 하도록 실행하고 heartbeat 확인까지 이 곳에서 담당하고 있는 것을 볼 수 있습니다.

<br/>

원래 CeleryExecutor는 포스팅 하나에 전부 담으려고 했는데, 이 역시 쓰다보니 투머치토킹이 되어버려서 2장으로 나누도록 하겠습니다. 이후에는 실제로 task를 실행시키는 단계와 상태 변경 정도가 남아있는데, 코드는 길지는 설명이 길지 않아 나머지 1장에 전부 담아낼 수 있을것 같습니다.

