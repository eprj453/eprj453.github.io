---
title: "[Airflow] Airflow의 하루"
date: 2023-04-02 00:00:00
categories:
- airflow
tags: [airflow]

---

Airflow를 처음 사용할 때 가장 헷갈리는 것 중 하나는 날짜의 개념일거라 생각합니다. 꼭 Airflow를 사용하지 않더라도, 배치 스케줄링의 멱등성과 정확성을 위해서라도 Airflow의 시간 개념을 알아두면 나쁘지 않을 것 같아 내용을 정리합니다. execution date와 data_interva_start  / end 등이 헷갈리는 분들께 도움이 되길 바랍니다.

<br/>

# 하루

하루의 사전적 의미는 24시간 중에 자정에서 다음 자정까지를 뜻한다고 합니다. 2023년 4월 1일 00시부터 2023년 4월 2일 00시까지가 사전적으로 정확한 하루가 지났다고 말할 수 있겠네요.

그렇다면, 우리는 이 말은 어떻게 받아들이나요?

> `밀린 일을 처리하느라 하루종일 걸렸다`

여기서 하루는 정확히 00시부터 다음 날 00시를 의미하나요? 그럴수도 있겠지만 그렇지는 않습니다. 많은 회사들의 근무시간인 오전 9시부터 오후 6시일수도 있고, 야근을 했다면 다음 날 00시가 아니라 새벽까지를 지칭하는 말이 될지도 모르겠습니다.

하루의 사전적 개념은 명확하지만, 하루라는 개념 자체는 인간들의 약속입니다. 지구에서 하루가 지날동안 수성에서는 76일이 지난다고 하니, 정확히는 지구에 사는 인간들의 약속이라고 봐야겠습니다. 생각해보니 강아지의 수명을 인간의 수명에 빗대어 강아지의 1년은 인간의 4~7년 정도로 나이를 환산하는 것 또한 하루라는 개념이 대상에 따라 달라질 수 있음을 나타내기도 합니다.

<br/>

# Airflow의 하루

갑자기 웬 하루 이야기를 하나 싶겠지만, 이야기하고자 하는 핵심은 `Airflow의 하루는 30분도, 한시간도, 하루도, 일주일도, 1년도 될 수 있다`입니다.

여기서 하루를 24시간이 흐르는 인간 개념의 하루로 보더라도, 00시부터 다음 날 00시가 아닌 9시부터 다음 날 9시까지일수도 있고 18시부터 다음 날 18시일수도 있다는 말입니다.

그렇다면 하루를 24시간이 아닌 23시간으로 정의하고 Airflow DAG을 구성할수도 있을까요? 아예 불가능하진 않겠습니다만, 매우 귀찮고 관리도 어렵습니다. 그리고 언젠가 원하지 않는 결과를 발생시킬 것입니다. 

배치 스케줄링에서 가장 중요한 것은 규칙입니다. 규칙적으로 정해진 시간에 맞춰 실행되어야 합니다. 예외조차도 규칙적으로 발생한다면 큰 문제는 되지 않습니다. 매 시간마다 도는 DAG인데 오전 9시에만 실행되지 않도록 하는 것은 그리 어렵지도, 문제될 것도 없는 배치입니다.

그런데 예외가 매번 발생하고 그 결과가 계속 누적된다면 이는 불행한 결과를 초래합니다. 하루가 24시간이 아니라 23시간이라면, 1시간 차이가 매번 발생하고 24번의 배치가 도는 시점에는 일부러 24시간을 더해줘야 그 차이가 맞을 것입니다. 윤달을 생각하면 이해가 바로 되실거라 생각합니다.

<br/>

매주 월요일마다 도는 DAG가 있다고 해보겠습니다. 이 DAG이 도는 날은 항상 월요일입니다. 그렇다면 이 DAG의 어제는 언제일까요.

월요일의 어제는 일요일이지만, 이 DAG의 어제는 또 월요일입니다. 매주 월요일마다 도는 DAG의 세상에는 일요일도 화요일도 없습니다. 오직 월요일만이 이 DAG의 시간 기준이 됩니다. 이전 월요일, 이번 월요일, 다음 월요일... 계속 월요일만 있을 뿐입니다.

매월 1일에 도는 DAG도 개념은 동일합니다. 매월 1일은 그 달에 오직 하나만 존재하는 유일한 존재입니다. 어제가 4월 1일이었는데, 매달 1일에 도는 DAG이 있었다면 어제도 돌았을겁니다. 이 DAG의 어제는 3월 31일이 아니라 3월 1일입니다.

매 시간 도는 DAG의 어제는 1시간 전, 5분마다 도는 DAG의 어제는 5분 전입니다. 

이처럼 배치에서 중요한 것은 `어제와 오늘이 어디부터 어디까지`인가?

<br/>

# data_interval의 등장

execution_date로 시간 개념을 지정하던 Airflow 1에서는 이러한 어제 / 오늘에 대한 정확한 구분을 하기 어려웠습니다. Airflow에서 execution_date는 매번 다를 수 있는 어제를 반환했는데, 이는 당연히 1년전일수도 하루 전일수도 1시간 전일수도 있습니다. 

Airflow 2버전에 들어오면서 생긴 개념이 `data_interval_start`와 `data_interval_end`입니다. data_interval_start는 배치 실행 시점의 어제, data_interval_end는 배치 실행 시점의 현재를 반환합니다.  

<br/>

# 그래서 DAG의 시간 세팅은 어떻게?

개인적으로 DAG을 원하는 시점에 동작하기 위해 고려하는 것은 크게 3가지입니다.

- 언제 도는지가 중요한 DAG인가
- catchup을 True로 둘 것인가 False로 둘 것인가
- 현재 기준 돌아야 하는 data_interval_start와 data_interval_end가 언제인지 알 수 있는가

<br/>

## 언제 도는지가 중요한 DAG인가

언제 도는지가 중요하지 않은 DAG이라기보단, 과거 데이터를 소급해야 하거나 BackFill이 필요하지 않는 DAG이 있을 수 있습니다. 예를 들면, 저는 매일 9시 30분에 팀원들이 돌아가면서 아침 커피를 사는데 이 알람에서 어제가 언제고 오늘이 언제인지는 크게 중요하지 않습니다. 그저 정해진 시간에 해당 인덱스에 걸려있는 팀원만 알려주면 되기 때문에, 이런 DAG에서는 시간 세팅이 크게 중요하진 않습니다.

<br/>

## catchup을 True로 둘 것인가 False로 둘 것인가

사실 이게 제일 중요할지도 모르겠습니다. catchup을 True로 두냐 False로 두는 것은 어제 / 오늘보다는 `해당 DAG의 개시를 언제 할 것이냐` 에 결정적입니다. 이는 당연히 `start_date`를 언제로 하느냐와도 직접적으로 연관됩니다.

> An Airflow DAG defined with a `start_date`, possibly an `end_date`, and a non-dataset schedule, defines a series of intervals which the scheduler turns into individual DAG runs and executes. The scheduler, by default, will kick off a DAG Run for any data interval that has not been run since the last data interval (or has been cleared). This concept is called Catchup.
>
> If your DAG is not written to handle its catchup (i.e., not limited to the interval, but instead to `Now` for instance.), then you will want to turn catchup off. This can be done by setting `catchup=False` in DAG or `catchup_by_default=False` in the configuration file. When turned off, the scheduler creates a DAG run only for the latest interval.
>
> - [Airflow Docs - catchup](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#catchup)

catchup을 False로 두면 해당 DAG의 `어제`를 기준으로 해당 DAG이 하나 실행되고, 이후 예정된 시간에 DAG이 동작합니다. 이전 과거를 소급해 Backfill로 동작하거나 미래를 기다리지 않습니다. 

```python
from datetime import datetime
from airflow.decorators import dag, task

@dag(
	schedule_interval="0 9 * * *",
	start_date=datetime(2023, 5, 2, 9, 0),
	catchup=False,
	tags=['parksw2', 'test'])
def CATCHUP_TEST():
	@task
	def start():
		print("hello start!")

	@task
	def end():
		print("hello end!")

	start >> end


CATCHUP_TEST_DAG = CATCHUP_TEST()
```

오늘은 2023년 4월 2일입니다. 매일 9시에 도는 DAG의 start_date를 5월 2일로 잡아두고 catchup을 False로 주면 
