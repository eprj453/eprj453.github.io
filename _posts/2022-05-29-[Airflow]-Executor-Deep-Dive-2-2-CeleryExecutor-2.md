---
title: "[Airflow] Executor Deep Dive 2-2. CeleryExecutor 2"
date: 2022-05-29 00:00:00
categories:
- airflow
tags: [airflow]

---

1장에서는 sorted_queue가 어디서부터 생겨나는가?를 보았습니다. 

# class CeleryExecutor.trigger_tasks()

```python
Cdef trigger_tasks(self, open_slots: int) -> None:
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

sorted_queue 그 밑으로는 간단합니다. `open_slot` 은 최대 병렬처리 가능한 프로세스 갯수에서 현재 running으로 표시되어 있는 task의 갯수를 뺀 갯수. 즉 현재 실행 가능한 프로세스의 갯수입니다. queue 앞에서부터 실행 가능한 프로세스 갯수만큼 pop(0)으로 빼서 `task_tuples_to_send` 에 넣은 뒤, `_process_tasks`에 인자로 넘겨줍니다.

<br/>

# _process_tasks() / _send_tasks_to_celery()

```python
def _process_tasks(self, task_tuples_to_send: List[TaskInstanceInCelery]) -> None:
    first_task = next(t[3] for t in task_tuples_to_send)

    # Celery state queries will stuck if we do not use one same backend
    # for all tasks.
    cached_celery_backend = first_task.backend

    key_and_async_results = self._send_tasks_to_celery(task_tuples_to_send)
    self.log.debug('Sent all tasks.')

    for key, _, result in key_and_async_results:
        if isinstance(result, ExceptionWithTraceback) and isinstance(
            result.exception, AirflowTaskTimeout
        ):
            if key in self.task_publish_retries and (
                self.task_publish_retries.get(key) <= self.task_publish_max_retries
            ):
                Stats.incr("celery.task_timeout_error")
                self.log.info(
                    "[Try %s of %s] Task Timeout Error for Task: (%s).",
                    self.task_publish_retries[key],
                    self.task_publish_max_retries,
                    key,
                )
                self.task_publish_retries[key] += 1
                continue
        self.queued_tasks.pop(key)
        self.task_publish_retries.pop(key)
        if isinstance(result, ExceptionWithTraceback):
            self.log.error(CELERY_SEND_ERR_MSG_HEADER + ": %s\n%s\n", result.exception, result.traceback)
            self.event_buffer[key] = (State.FAILED, None)
        elif result is not None:
            result.backend = cached_celery_backend
            self.running.add(key)
            self.tasks[key] = result

            # Store the Celery task_id in the event buffer. This will get "overwritten" if the task
            # has another event, but that is fine, because the only other events are success/failed at
            # which point we don't need the ID anymore anyway
            self.event_buffer[key] = (State.QUEUED, result.task_id)

            # If the task runs _really quickly_ we may already have a result!
            self.update_task_state(key, result.state, getattr(result, 'info', None))
            

            
def _send_tasks_to_celery(self, task_tuples_to_send: List[TaskInstanceInCelery]):
        if len(task_tuples_to_send) == 1 or self._sync_parallelism == 1:
            # One tuple, or max one process -> send it in the main thread.
            return list(map(send_task_to_executor, task_tuples_to_send))

        # Use chunks instead of a work queue to reduce context switching
        # since tasks are roughly uniform in size
        chunksize = self._num_tasks_per_send_process(len(task_tuples_to_send))
        num_processes = min(len(task_tuples_to_send), self._sync_parallelism)

        with ProcessPoolExecutor(max_workers=num_processes) as send_pool:
            key_and_async_results = list(
                send_pool.map(send_task_to_executor, task_tuples_to_send, chunksize=chunksize)
            )
        return key_and_async_results
```

first_task에서는 가장 첫번째 execute_command를 할당합니다. 



중간에 _send_tasks_to_celery()가 등장하는데, 실행할 수 있는 task가 1개밖에 없다면 task를 바로 map 함수에 send_task_to_executor를 적용시켜 executor로 보냅니다. `send_task_to_executor` 는 Celery의 task 클래스의 메서드인데, 이 안에 있는 메서드를 따라가다보면 execute_command를 OS 단의 API Call로 직접 호출하는 메서드까지 나오지 않을까?하는 추측을 해봅니다. 이것까지 들어가지는 않겠습니다.

어쨌든 여기서는 ProcessPoolExecutor를 사용한 병렬처리가 들어갑니다. 각자 executor에 task를 병렬로 할당하는 `execute_async()` 메서드는 CeleryExecutor에서 사용하지 않았지만, 실행 가능한 task의 list를 `한군데`에서 계속 갱신한 뒤 실행하는것만 병렬로 각각 executor에 뿌려주는 방식이라면 가능할것도 같습니다. 실행하는 task가 겹칠 일도 없고 성공하거나 실패한 task가 queue에 남아있을 일도 없습니다.



for loop 부터는 task의 상태를 갱신합니다. `_send_tasks_to_celery()`로 실행시킨 task는 queue에서 빼고 running 상태인 task는 `self.running`에 집어넣는 등의 동작이 실행됩니다. self.event_buffer에도 queue와 task id를 저장하는데 정확한 쓰임새는 잘 모르겠습니다. 

`_process_tasks()`에서는 task를 실제로 실행하는 단계로 넘기고 상태를 갱신하는 작업이 이루어지는 것을 알 수 있습니다.



<br/>

# 정리

<img width="962" alt="image" src="https://user-images.githubusercontent.com/52685258/171681196-a7cdea16-852d-4ae3-8e82-c0be4ee7a230.png">



이 외에도 CeleryExecutor에서는 실행되는 메서드가 더 많지만, `change_state / update_task_state` 등 task의 상태를 관리하는 메서드입니다. 이 메서드들은 더 깊게 보지는 않겠습니다.

