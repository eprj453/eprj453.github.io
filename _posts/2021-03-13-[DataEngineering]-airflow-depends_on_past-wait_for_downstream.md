---
title: "[airflow] airflow - depends_on_past / wait_for_downstream"
date: 2021-03-13 00:00:00
categories:
- airflow
tags: [data-engineering, airflow]
---

airflow task 설정을 하다가, 이전 task에 의존적으로 실행 계획을 만들 수 있는 옵션 2개를 찾았습니다. 이전 task에 상관없이 실행 가능한 모든 task를 실행시키는 경우에는 이 옵션들이 의미가 없겠지만, task의 성공여부에 따라 ETL 계획에서 조건을 주고 싶은 경우에는 이 두가지 옵션이 도움이 될 것입니다. 

이 두가지 옵션이 의존하는 이전 task의 범위가 어디까지인지 테스트한 사항들을 기록합니다.



# DAG 설정

```python
default_args = {
    "owner": "parksw2",
    'start_date': datetime(2021, 3, 1, 0, 0, 0),
    "email": ["airflow@airflow.com"],
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 0
}

default_dag = DAG(
    ...,
  	...,
  	...,
    max_active_runs=1
)
```



# Task 설정

```python
def sleep1(**context):
    time.sleep(10)


def sleep2(**context):
    time.sleep(5)
    execution_date = context['execution_date']
    if execution_date.month == 3 and execution_date.day == 3:
        raise FileNotFoundError


def sleep3(**context):
    time.sleep(1)


test_task1 = PythonOperator(
    task_id="test_sleep1",
    python_callable=sleep1,
    dag=default_dag,
    provide_context=True
)

test_task2 = PythonOperator(
    task_id="test_sleep2",
    python_callable=sleep2,
    dag=default_dag,
    provide_context=True
)

test_task3 = PythonOperator(
    task_id="test_sleep3",
    python_callable=sleep3,
    dag=default_dag,
    provide_context=True
)

test_task1 >> test_task2 >> test_task3
```



간단합니다. Test_task1 -> test_task2 -> test_task3 순으로 실행이 되고, 10초 / 5초 / 1초를 기다립니다.

3월 1일부터 현재 날짜까지의 task instance가 차례로 실행될 것이고, dag option에서 max_active_runs=1을 줬기 때문에 DAG instance는 하나씩 뜰 것입니다. 

단, 3월 3일의 test_task2 인스턴스는 조건에 따라 FileNotFoundError를 일으키고 종료됩니다.

다른 옵션을 주지 않았을 때, 실행결과는 다음과 같습니다.

![image](https://user-images.githubusercontent.com/52685258/111034841-c4e1fd80-845a-11eb-9315-aa7aeecdb90f.png)

3일의 test_sleep2 인스턴스는 에러가 발생했고, 그에 따라서 3일의 test_sleep3는 upstream_failed가 되었습니다. 그 외 다른 task 인스턴스는 영향을 받지 않았습니다.



# wait_for_downstream

wait_for_downstream 옵션을 주겠습니다.

```python
default_args = {
    "owner": "parksw2",
    'start_date': datetime(2021, 3, 1, 0, 0, 0),
    "email": ["airflow@airflow.com"],
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 0,
    "wait_for_downstream": True
}
```



실행 결과를 UI에서 결과를 보겠습니다.

![image](https://user-images.githubusercontent.com/52685258/111034103-6cf5c780-8457-11eb-950b-307e661a1f13.png)

3일에 test_sleep2 메서드가 에러를 발생시켰기 때문에(빨간 네모), 3일 DAG 인스턴스가 실패로 끝났습니다(빨간 동그라미). test_sleep3은 upstream_failed 상태가 되었네요(주황 네모).

여기서 4일째 인스턴스를 보겠습니다. 이전 날짜의 인스턴스가 실패로 끝났을 경우에는 DAG 인스턴스는 running 상태이지만 모든 task 인스턴스가 no_status 상태로 계속 대기합니다. 즉,  wait_for_downstream의 이전 task는 이전 날짜의 모든 task를 포함합니다.



# depends_on_past

이번에는 depends_on_past 옵션을 주겠습니다.

```python
default_args = {
    "owner": "parksw2",
    'start_date': datetime(2021, 3, 1, 0, 0, 0),
    "email": ["airflow@airflow.com"],
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 0,
    "depends_on_past": True
}
```

다른 조건은 이전과 동일합니다. 실행 결과를 UI에서 확인하겠습니다.

![image](https://user-images.githubusercontent.com/52685258/111034418-e8a44400-8458-11eb-8f79-8984d4230e8e.png)

Depends_on_past 옵션을 주었더니, 이전 task의 범위가 이전 날짜의 동일 task인것을 볼 수 있습니다. 4일자의 test_sleep1은 정상적으로 실행되었으니, 3일자의 test_sleep2의 실패에 영향을 받지 않았습니다. 4일자의 test_sleep2는 3일자의 test_sleep2의 실패에 영향을 받아 no_status 상태로 대기하고 있고, 이에 따라 그 이후의 모든 task 또한 대기하고 있음을 볼 수 있습니다.

정리하자면

- wait_for_downstream : 이전 날짜의 task 인스턴스 중 하나라도 실패한 경우에는 해당 DAG는 실행되지 않고 대기.
- Depends_on_past : 이전 날짜의 task 인스턴스 중에서 동일한 task 인스턴스가 실패한 경우 실행되지 않고 대기.



Wait_for_downstream과 depends_on_past 옵션에 대해 살펴봤습니다. 

이 두가지 옵션을 잘 활용한다면 인스턴스 실행 중지를 통해 잘못된 ETL을 방지하고 에러 추적 및 재처리에 도움을 받을 수 있습니다 :)

