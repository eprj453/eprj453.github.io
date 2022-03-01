---
title: "[Airflow] Executor Deep Dive 1. LocalExecutor"
date: 2022-02-23 00:00:00
categories:
- airflow
tags: [airflow]

---



airflow에는 여러 구성요소가 있지만 그 중에서 Executor에 대한 공부를 가장 소홀히 했다는 생각이 듭니다. 코딩을 하며 가장 많이 만나는 각종 Operator은 그 기능이 직관적이기 때문에 내부 코드를 굳이 보지 않아도 사용에 큰 지장이 없고 보더라도 그 구조가 간단한 경우가 많습니다.

Executor는 각 Task를 어떻게 할당하는가에 대한 중요한 역할을 하고 있음에도 사용하고 파라미터를 어떻게 조작할까 정도만 생각했었는데, Executor의 내부 코드를 보고 어떤 식으로 Task를 할당하고 있는지 Executor별로 하나씩 살펴보고자 합니다.



# LocalExecutor

LocalExecutor보다 더 간단하게 사용할 수 있는 SequentialExecutor가 있긴 하지만, SequentialExecutor는 Task의 병렬 실행이 불가능하다는 치명적인 단점이 있어 상용 환경에 올리기에는 무리가 있습니다. 테스트 용도가 아니면 권장하지 않고 개인적으로는 테스트 용도로 사용하더라도 병렬 실행이 가능한 LocalExecutor를 사용하는 것이 좋다고 생각합니다.

LocalExecutor를 표현한 그림입니다.

![](https://user-images.githubusercontent.com/52685258/155315401-84bd9c25-f7b9-47cd-82c4-243d4b6626a2.png)

Executor는 Scheduler가 Task 실행방식을 결정하기 위한 것이기 때문에 논리적으로는 그림의 Scheduler 자리 어딘가에 Executor가 있다고 보면 되겠습니다. Executor는 여러 프로세스를 병렬로 실행하기 위해 `부모 프로세스 : 자식 프로세스 = 1 : N`의 구조를 가지고 있는 것을 볼 수 있습니다

그럼 LocalExecutor는 이렇게 정리할 수 있습니다.

- Airflow의 Task들을 병렬처리한다.
- 여러 subprocess를 두는 방식으로 병렬처리를 한다.



그럼 코드를 직접 까서 이게 사실인지 확인해보겠습니다.

<br/>

# Code

https://github.com/apache/airflow/blob/main/airflow/executors/local_executor.py

저는 MWAA의 2.0.2 버전을 보았으나 최신 버전의 LocalExecutor 코드와 큰 차이는 없습니다.

먼저 class LocalExecutor의 생성자(`__init__`)쪽을 보겠습니다.

```python
    def __init__(self, parallelism: int = PARALLELISM):
        super().__init__(parallelism=parallelism)
        self.manager: Optional[SyncManager] = None
        self.result_queue: Optional['Queue[TaskInstanceStateType]'] = None
        self.workers: List[QueuedLocalWorker] = []
        self.workers_used: int = 0
        self.workers_active: int = 0
        self.impl: Optional[
            Union['LocalExecutor.UnlimitedParallelism', 'LocalExecutor.LimitedParallelism']
        ] = None
```

일단 BaseExecutor를 상속하고 있기 때문에 super 메서드로 부모 클래스의 `__init__`을 호출해 사용하는 것을 볼 수 있습니다. 그 외에는 `self.impl` 쪽이 눈에 띄는데, 이름과 Optional 속성을 보면

- 뭔가 구현을 해서 사용하겠구나
- 구현을 하는 구현체는 `LocalExecutor.UnlimitedParallelism / LocalExecutor.LimitedParallelism` 둘 중 하나겠구나
- LocalExecutor 내부 클래스에 `UnlimitedParallelism, LimitedParallelism`이 있겠구나

정도를 생각해볼 수 있겠습니다.

<br/>

## UnLimitedParallelism / LimitedParallelism

parallelism은 airflow.cfg에서 갯수를 설정할 수 있습니다. parallelism 파라미터에 따라 UnLimitedParallelism / LimitedParallelism이 구분되는건 `class LocalExecutor`의 `def start(self)`에서 볼 수 있습니다.



```python
    def start(self) -> None:
        """Starts the executor"""
        self.manager = Manager()
        self.result_queue = self.manager.Queue()
        self.workers = []
        self.workers_used = 0
        self.workers_active = 0
        self.impl = (
            LocalExecutor.UnlimitedParallelism(self)
            if self.parallelism == 0
            else LocalExecutor.LimitedParallelism(self)
        )

        self.impl.start()
```

executor가 실행되면 이 start 메서드가 실행되고 이 때 impl도 구성이 됩니다. parallelism이 0일 때는 UnlimitedParallelism, 그렇지 않을때는 LimitedParallelism을 선택합니다. 그 뒤에 내부 클래스에서 구현된 start() 메서드가 실행됩니다. 본격적 기능은 구현 클래스의 start()를 봐야겠네요.

갑자기 궁금해졌는데 그럼 parallelism을 음수로 설정하면 어떻게 될까요? 결론부터 말씀드리면 컴파일에서 오류가 나지는 않지만 task가 실행되지 않습니다. 보다보니 음수일때는 UnlimitedParallelism이 선택되거나 Exception을 내는게 맞는게 아닌가 싶네요.

UnLimitedParallelism / LimitedParallelism의 코드를 전부 까보기 때문에 내용이 좀 길어서 다음 포스팅에서 이어가겠습니다. :)





