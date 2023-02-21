---
title: "[Airflow] 큰 DAG 분리하기 1. 코드 분리"
date: 2023-02-22 00:00:00
categories:
- airflow
tags: [airflow]

---

sql 튜닝, airflow 마이그레이션, 기타 등등을 진행하던 와중에 문제가 생겼습니다.

- graph UI를 도저히 볼 수가 없어요

- DAG 하나에 task가 너무 많고 코드가 너무 길어요

해당 DAG은 가이드만 잡아드린 상태로 DW 엔지니어 분들이 직접 사용하셨기 때문에 제가 직접 관리하진 않았는데, 최근 다른 서비스 마이그레이션을 진행하면서 Task가 많이 늘어서 문제가 생겼습니다.

<br/>

<img width="843" alt="image" src="https://user-images.githubusercontent.com/52685258/220400701-2526dbf3-09df-42a0-be3f-35ab9023b4ed.png">

> 살려줘....

<br/>

하나의 DAG에 640개가 넘는 Task가 엄청난 Dependency를 이루고 있었습니다.

UI 로딩 자체도 너무 느리고, 원하는 Task를 검색해 Upstream과 Downstream을 보는건 거의 불가능에 가까웠습니다. DAG 하나밖에 없는 스크립트의 길이가 7천줄이 훨씬 넘어 스크립트의 가독성 또한 상당히 떨어졌습니다.

당장 조치가 필요했습니다.

<br/>

# UI 개선

UI 개선은 사실 간단합니다. TaskGroup 단위로 원하는 Task들을 묶어놓으면 훨씬 보기가 편해집니다. 어떤 기준으로 TaskGroup을 정할 것이냐에 대한  기준만 세워진다면 어렵지 않은 작업입니다.

<img width="1239" alt="image" src="https://user-images.githubusercontent.com/52685258/220402196-63df5748-b50e-4134-8919-ded67ca32eef.png">

> 644개의 Task -> 16개의 TaskGroup

보지 않아도 되는 Task는 Group으로 접어놓고, 원하는 Task가 속한 TaskGroup만 펼쳐서 보면 됩니다. 2 버전 들어오면서 사용자가 체감하기에 가장 좋은 기능이 아닌가 싶습니다. 

<br/>



# 코드 분리

그래프 UI는 사실 Task만 잘 돈다면 조금 불편해도 어느정도는 감수할 수 있습니다. 트리 UI로도 필요한 작업은 모두 가능하기 때문입니다. 

그러나 코드가 길고 가독성이 떨어지게 되면 작업 효율에도 바로 영향을 미치고, 사실 코드 분리가 UI 개선과도 무관하지 않기 때문에 여기에 더 많은 생각을 했습니다.

import 구문이나 기타 function을 제외하면 제가 업무에서 사용하는 DAG 스크립트의 구조는 크게 3단계로 나눌 수 있습니다. 

![image](https://user-images.githubusercontent.com/52685258/220407681-e6fd7d56-3940-4c41-bcdb-d13021eb951c.png)

- 1번 DAG 선언부
- 2번 Task 선언부
- 3번 Task Dependency 선언부



일반적으로 사용하는 Airflow Example과 크게 다르지 않습니다. 문제의 DAG는 2번 구간이 약 5천줄, 3번 구간이 약 2천줄입니다.

<br/>

## 어디까지 나눌 것인가?

일단 DAG 자체를 분리하지는 않았습니다. 이유는 2가지입니다.

첫번째는, DAG를 나누면 그에 맞게 ExternalTaskSensor와 ExternalTaskMarker를 앞뒤로 넣어줘야 하는데, DAG와 DAG가 아닌 DAG에 속한 Task들과 DAG에 속한 Task들이 복잡하게 dependency를 이루고 있었기 때문에 해당하는 모든 Task들에 앞뒤로 sensor와 marker를 넣어주는 작업이 생각보다 오래 걸릴것 같았습니다.

두번째는, ExternalTaskMarker가 다른 DAG의 task를 Upstream task로 잡고 있을 때, Marker의 downstream task를 재귀적으로 clear시키지 못하는 상황이 간헐적으로 발생했습니다. MWAA의 문제인지 airflow 2.2.2 버전의 문제인지 제 문제인지는 알 수 없으나, 나름대로 여러번 테스트해보았음에도 이 문제를 해결하지 못해 DAG을 분리하는 작업은 하지 않았습니다. (예전에 stackoverflow에도 비슷한 이슈로 질문을 올렸으나... 외면받았습니다.)

DAG 선언부는 그대로 두었습니다. DAG을 분리하거나 한번에 여러개의 DAG을 생성하지 않는 이상 인스턴스 하나 생성하는 코드이기 때문에 길거나 가독성을 해치지도 않습니다.



그렇다면 

- Operator 인스턴스를 바깥에서 생성해놓고 task dependency를 설정하는 코드도 바깥으로 빼는 방법 (2번 3번 모두 바깥으로)
- Operator 인스턴스는 DAG 안에서 생성해 놓은 뒤 task dependency를 설정하는 코드를 바깥으로 빼는 방법 (3번만 바깥으로)

2가지의 선택지가 남았고, 저는 3번인 dependency 선언부만 바깥으로 빼기로 했습니다.

<br/>

## 왜?

Task 선언 자체를 자동화하거나 간소화하는 방법은 생각하지 못했기 때문에, 어딘가에는 인스턴스가 선언되어있어야 했습니다.

만약 외부에 별도 파이썬 스크립트나 yaml 파일로 필요한 Task들을 별도로 만들어놓는다면, 이에 대한 가장 큰 이점은 어떤 DAG에서든 해당 Task를 import하거나 정보를 읽어와 인스턴스를 만들 수 있다, 즉 재사용에 대한 이점이라 생각했습니다. 특정 메서드나 기능을 별도 클래스 / 메서드로 빼서 모듈화시키는 작업과 비슷한 맥락으로 생각했습니다.

일단 재사용에 대한 수요가 없었습니다. 이 Task에서 수행하는 작업을 다른 DAG에서 동일하게 실행한다거나 하는 구조가 없었고, 자연스럽게 Task 선언부를 분리한다고 해서 딱히 이득이 없을거라 판단했습니다. 또한 기존 작업 방식에서 최대한 위화감을 줄이고자 했기 때문에 Task 선언부는 최대한 건드리지 않으려 했던 것도 이유 중 하나입니다.

그리고 Airflow의 Task instance는 반드시 dag를 파라미터로 가지는, 마치 DAG에 종속적인 구조를 가지고 있는데 외부로 TaskInstance를 빼버리면 어디에선가는 DAG 관련 파라미터 설정을 해줘야하는 점도 불편함으로 다가왔습니다.

```python
# 이런 Task가 별도로 존재한다면
test_task_2 = BashOperator(
  task_id="test_task_1",
  bash_command='echo hello 1'
)

# DAG에서 Import를 하든 외부에서 dependency를 설정하든 아래와 같은 형태로 다시 만들어줘야 합니다.
test_task_2 = BashOperator(
  task_id="test_task_1",
  bash_command='echo hello 1',
  dag=dag
)
```

DAG의 숫자 자체가 많아진다면 DAG 정보 / Task 정보를 따로 관리하며 DAG 인스턴스가 생성되는 시점에 이를 매핑해주는 식으로 관리한다면 편리하겠지만 지금처럼 단일 DAG에 Task가 많은 경우에는 이점이 적다고 판단했습니다.

<br/>

현재 dependency 선언부를 분리해 job 테스트중에 있습니다. 다음 포스팅에서는 어떻게 dependency를 분리했고 그 과정에서 airflow 구조를 어디까지 살펴보았는지 적어보도록 하겠습니다 :)
