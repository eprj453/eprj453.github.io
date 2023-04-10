---
title: "[Airflow] Airflow의 하루(Airflow DAG 시간 지정하기)"
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
- 내가 이 DAG을 언제부터 실행시키고 싶은가

<br/>

## 언제 도는지가 중요한 DAG인가

언제 도는지가 중요하지 않은 DAG이라기보단, 과거 데이터를 소급해야 하거나 BackFill이 필요하지 않는 DAG이 있을 수 있습니다. 예를 들면, 저는 매일 9시 30분에 팀원들끼리 돌아가면서 정하는 아침 커피 당번을 알려주는 알람을 airflow에 올려놓고 사용하고 있습니다. 이 알람에서 어제가 언제고 오늘이 언제인지는 크게 중요하지 않습니다. 그저 정해진 시간에 해당 인덱스에 걸려있는 팀원만 알려주면 되기 때문에, 이런 DAG에서는 시간 세팅이 크게 중요하진 않습니다.

<br/>

## catchup을 True로 둘 것인가 False로 둘 것인가

사실 이게 제일 중요할지도 모르겠습니다. catchup을 True로 두냐 False로 두는 것은 어제 / 오늘보다는 `해당 DAG의 개시를 언제 할 것이냐` 에 결정적입니다. 이는 당연히 `start_date`를 언제로 하느냐와도 직접적으로 연관됩니다.

> An Airflow DAG defined with a `start_date`, possibly an `end_date`, and a non-dataset schedule, defines a series of intervals which the scheduler turns into individual DAG runs and executes. The scheduler, by default, will kick off a DAG Run for any data interval that has not been run since the last data interval (or has been cleared). This concept is called Catchup.
>
> If your DAG is not written to handle its catchup (i.e., not limited to the interval, but instead to `Now` for instance.), then you will want to turn catchup off. This can be done by setting `catchup=False` in DAG or `catchup_by_default=False` in the configuration file. When turned off, the scheduler creates a DAG run only for the latest interval.
>
> - [Airflow Docs - catchup](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#catchup)

catchup을 False로 두면 해당 DAG의 `어제`를 기준으로 해당 DAG이 하나 실행되고, 이후 예정된 시간에 DAG이 동작합니다. 이전 과거를 소급해 Backfill로 동작하거나 미래를 기다리지 않습니다. 아래와 같이 DAG 세팅을 해보겠습니다.

```python
from datetime import datetime
from airflow.decorators import dag, task

@dag(
	schedule_interval="0 9 * * *",
	start_date=datetime(2023, 4, 2, 9, 0),
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

이 DAG을 실행시키는 시간은 2023년 4월 5일입니다. 매일 9시에 도는 DAG의 start_date를 4월 2일로 잡아두었으니, catchup을 생각하지 않는다면 data_interval_start가 4월 2일인 DAG부터 돌아야 할 것 같이 생겼습니다.

<img width="489" alt="image" src="https://user-images.githubusercontent.com/52685258/230120655-fecd0393-67ad-429b-9af5-98ba82b2eb5a.png">

그러나 실제로는 start_date를 4월 4일로 두는 DAG 인스턴스 하나가 실행되었습니다. schedule_interval을 매일 9시로 지정했으니 이 DAG의 어제는 24시간 전이고, 4월 5일의 24시간 전인 4월 4일을 start_date로 두는 DAG 인스턴스 하나가 실행된 것입니다. 내일(4월 6일) 오전 9시에는에는 4월 5일을 data_interval_start로 두는 DAG이 실행될 것입니다.

<br/>

catchup을 True로 바꿔보겠습니다.

<img width="498" alt="image" src="https://user-images.githubusercontent.com/52685258/230122302-122b8adf-0c9b-4ea6-8f7a-e080e04f9d4f.png">

start_date가 4월 2일인 DAG부터 차례대로 실행되어 4월 4일인 DAG까지 실행되었습니다. Backfill의 개념을 알고 계시다면 어렵지 않게 이해하실 수 있겠습니다만, catchup을 True로 설정하고 start_date를 과거로 지정했다면 과거의 시간까지 모두 실행시킨다는 것을 알 수 있습니다.

여기서 중요한 점은 start_date는  `DAG이 이 날부터 돌았으면 좋겠다`하는 날짜가 아니라 `내가 DAG이 이 날부터 돌았으면 좋겠다하는 날짜의 어제` 를 지정해야 한다는 점입니다. 여기서 어제는 인간의 어제가 아니라 airflow의 어제입니다.

<br/>

이해를 돕기 위해 스케줄을 조금 바꿔보겠습니다.

```python
from datetime import datetime
from airflow.decorators import dag, task

@dag(
	schedule_interval="0 9 * * MON",
	start_date=datetime(2023, 4, 5, 9, 0),
	catchup=False,
	tags=['parksw2', 'test'])
def CATCHUP_TEST():
	@task
	def start():
		print("hello start!")

	@task
	def end():
		print("hello end!")

	start() >> end()


CATCHUP_TEST_DAG = CATCHUP_TEST()
```

매주 월요일 오전 9시에 도는 DAG의 start_date를 2023년 4월 5일로 잡아놨습니다. 4월 5일은 수요일입니다. 이 DAG의 catchup을 False로 잡았을 때, 이 DAG은 어떻게 실행될까요?

<img width="459" alt="image" src="https://user-images.githubusercontent.com/52685258/230125054-a539cb28-dbc2-41ee-9d8e-f56b2338e873.png">

아무런 실행도 일어나지 않습니다. 왜일까요?

이 DAG의 start_date를 수요일로 지정했습니다. 4월 5일 수요일을 기준으로 가장 직전의 월요일은 4월 3일입니다. 이 DAG의 하루는 1주일, 그것도 월요일만이 존재하는 1주일입니다. 4월 3일, 4월 10일, 4월 17일... 이 DAG의 세상에서는 월요일만 존재해야 합니다. 그렇다면 이 DAG이 data_interval_start로 잡을 수 있는 날짜는 4월 10일이고, data_interval_end는 4월 17일이기 때문에 최초 실행은 4월 17일이어야 맞습니다.

그러나 catchup을 False로 주면 상황이 조금 달라집니다. 이 DAG의 data_interval_start를 start_date로 바로 잡아버립니다. 이 DAG의 세상에서는 존재하지 않는 수요일이라는 날이 data_interval_start로 잡히게 되는 것이죠. 바로 다가오는 airflow의 내일, 여기서는 4월 10일을 data_interval_end로 DAG이 실행됩니다. 여기서는 data_interval_start -> 4월 5일, data_interval_end -> 4월 10일이 될 것입니다. DAG의 실행 또한 4월 17일이 아닌 4월 10일이 될 것입니다.

jinja template 등으로 data_interval_start를 인자로 넘기거나 할 때, 이 부분을 특히 조심해야 합니다.



<br/>

혹시, catchup을 True로 준다면 어떨까요?

<img width="537" alt="image" src="https://user-images.githubusercontent.com/52685258/230130453-d869a8e7-fec7-4c32-80de-9a6bc6826f4f.png">

catchup이 True이든 False이든 start_date 이전의 날짜는 임의로 backfill을 실행시키지 않는 이상 실행되지 않습니다. catchup을 True로 준다고 하더라도 start_date 이전의 스케줄링은 실행되지 않습니다.

이 DAG의 세상에서는 start_date 4월 5일은 없는 날짜를 지정한 것이나 다름없기 때문에 지정할 수 있는 날짜가 생길때까지 기다립니다. start_date를 4월 5일로 지정했지만 그 다음 월요일인 4월 10일을 지정한 것과 다르지 않게 됩니다.

이 DAG은 4월 10일을 data_interval_start로 잡고 4월 17일을 data_interval_end로 잡는 DAG이 최초로 실행되고, 실행일은 4월 17일입니다.

<br/>

보셔서 아시겠지만, `0 9 * * MON`으로 스케줄을 지정한 DAG의 start_date를 수요일로 지정하면 그 범위가 헷갈립니다. 예시에서는 1주일을 기준으로 보여드렸지만, 시간마다 도는 DAG의 start_date에 분 단위까지 지정한다거나 하는 경우도 이렇게 없는 세계의 시간을 끌어다 쓰게 되는 DAG 구성입니다. 그렇기 때문에 schedule_interval과 start_date는 그 간격을 일치시켜 지정해주는 것이 좋습니다.

<br/>

## 내가 이 DAG을 언제부터 실행시키고 싶은가

정리하자면

- start_date 이전의 스케줄은 backfill을 하지 않는 이상 일어나지 않는다.
- schedule_interval과 start_date의 시간대는 일치시켜주는 것이 좋다.
- catchup=False라면 
  - start_date 기준으로 data_interval_start로 두는 DAG이 최초로 실행된다. data_interval_end는 `Airflow 기준의 내일`이 되고, 이 때 data_interval_start가 매번 다를수 있음에 주의한다.
- catchup=True라면
  - start_date를 기준으로 가장 먼저 도래할 `Airflow의 날짜`를 data_interval_start로 잡는 DAG이 가장 먼저 수행된다.
  - start_date가 과거라면 과거부터 현재까지 DAG이 모두 실행된 된다

<br/>

# 결론

2가지를 기억하면 좋을 것 같습니다. 물론 DAG을 구성하는 사용자마다 원하는 시간개념이 다르기 때문에 주관적인 의견입니다.

1. catchup=True를 쓰자

위에서도 언급했지만, catchup=False를 쓰면 DAG의 시간에서는 없어야 할 날이 data_interval_start로 잡히는 경우가 생깁니다. 그렇기 때문에 catchup은 True로 잡는 것이 혼란을 줄일 수 있습니다.

<br/>

2. 내가 DAG을 최초로 실행시키고자 하는 실제 날짜를 기준으로, `airflow의 어제`를 start_date로 등록하자.

4월 5일 수요일에 매주 월요일 오전 9시에 도는 DAG을 세팅한다고 해보겠습니다. 원하는 최초 개시일은 4월 10일 월요일입니다. 그렇다면 4월 10일 오전 9시를 기준으로 `airflow의 어제`는 언제일까요?

4월 3일 오전 9시입니다. 이 날짜를 start_date로 두고 schedule_interval을 `0 9 * * MON`로 두면

- data_interval_start : 4월 3일 오전 9시
- data_interval_end : 4월 10일 오전 9시

로 DAG이 세팅됩니다. 

```python
from datetime import datetime
from airflow.decorators import dag, task

@dag(
	schedule_interval="0 9 * * MON",
	start_date=datetime(2023, 4, 3, 9, 0),
	catchup=True,
	tags=['parksw2', 'test'])
def CATCHUP_TEST():
	@task
	def start():
		print("hello start!")

	@task
	def end():
		print("hello end!")

	start() >> end()


CATCHUP_TEST_DAG = CATCHUP_TEST()
```

task 내부에서 data_interval_start을 쓰든 data_interval_end를 쓰든 헷갈릴 일이 없는 DAG입니다.

<br/>

매일 실행되는 DAG도 다르지 않습니다. 4월 5일 수요일 오후 6시에 매일 오전 10시에 도는 DAG을 세팅하고 싶다고 해보겠습니다. 이 DAG의 최초 개시일은 4월 6일 10시가 되어야 하고, 4월 6일 10시의 `airflow의 어제`는 4월 5일 오전 10시입니다.

```python
from datetime import datetime
from airflow.decorators import dag, task

@dag(
	schedule_interval="0 10 * * *",
	start_date=datetime(2023, 4, 5, 10, 0),
	catchup=True,
	tags=['parksw2', 'test'])
def CATCHUP_TEST():
	@task
	def start():
		print("hello start!")

	@task
	def end():
		print("hello end!")

	start() >> end()


CATCHUP_TEST_DAG = CATCHUP_TEST()
```

이 DAG의 최초 개시는 4월 6일 오전 10시이고,

- data_interval_start : 4월 5일 오전 10시
- data_interval_end : 4월 6일 오전 10시

로 DAG이 세팅될 것입니다.

<br/>

airflow의 시간 개념에 대해 간당명료하게 설명하고 싶었는데, 쓰다보니 산문처럼 길어졌습니다. 그래도 airflow의 시간 개념을 헷갈려하시는 분들께 조금이나마 도움이 되었으면 좋겠습니다.

