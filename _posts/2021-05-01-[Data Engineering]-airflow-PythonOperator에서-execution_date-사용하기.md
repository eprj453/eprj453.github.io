---
title: "[airflow] Airflow - PythonOperator에서 execution_date 사용하기"
date: 2021-05-01 00:00:00
categories:
- airflow
tags: [data-engineering, airflow]
---



airflow에서 DAG instance는 execution_date를 기준으로 생성됩니다. 주기적으로 발생하는 ETL 스케줄을 편리하게 task / dag 단위로 관리할 수 있다는게 airflow의 큰 장점입니다.

현업에서 airflow로 ETL을 작성하면서 맨 처음 사용했던 Operator는 BashOperator와 PythonOperator였습니다. 그 중에서 PythonOperator에서 execution_date를 어떻게 사용했고 개선했는지 정리합니다.



# 1. keyword argument

PythonOperator는 `python_callable` Argument에 메서드를 콜백함수로 주어 실행할 수 있습니다.

```python
{% raw %}
def test_func(arg1, arg2):
  print(arg1)
  print(arg2)

def test_task(name):
    compare_size_task = PythonOperator(
        task_id=f"{name}_task",
        python_callable=test_func,
        op_kwargs={
            "arg1":"hello",
         		"arg2": "airflow!"
            "execution_date": "{{ execution_date.in_timezone('Asia/Seoul').strftime('%Y-%m-%d') }}"
        },
        queue="s21",
        dag=dag,
    )
{% endraw %}
```



여기서 provide_context 옵션을 True로 주게 되면, 해당 메서드에서 task instance의 attribute들을 keyword argument로 바로 받아서 사용할 수 있습니다(default : False). 키워드 인자로 넘어간 변수를 바로 dictionary 형태로 참조할 수 있습니다.

```python
def test_func(arg1, arg2, **context):
    print(arg1)
    print(arg2)
 	  print(context['execution_date'])


test_task = PythonOperator(
      task_id=f"{name}_task",
      python_callable=test_func,
      op_kwargs={
        "arg1":"hello",
        "arg2": "airflow!"
      },
      provide_context=True,
      queue="s21",
      dag=dag,
    )
```

execution_date는 이 외에도 task 인스턴스가 생성되었을 때의 정보를 가지고 있습니다. pythonOperator가 아닌 다른 operator에서도 사용할 수 있습니다. 대표적으로 xcom에 변수를 넘길때도 이 keyword argument를 이용해 메세지를 주고 받습니다.

위에서 보았듯, `python_callable` 에서 인자로 넘겨준 메서드 내부에서 context를 가지고 execution_date을 가져올 수 있습니다. 이 execution_date는 가져올 당시 pendulum 클래스 타입이기 때문에 원하는 형태로(datetime으로 변환 / KST로 변환...) 변환해 사용하는 것을 권장합니다.

그러나 이 방법을 권장하고 싶지는 않습니다. 부득이한 경우 keyword argument를 메서드에 넘겨 사용해야겠지만, 이렇게 모든 task의 정보를 keyword argument 하나로 넘겨주게 되면 해당 operator가 함수에서 어떤 것을 필요로 하는지 알 수 없게 됩니다. 즉, 위에서 `test_task`만 봐서는 `test_func`에서 어떤 정보를 필요로 하는지 알 수 없습니다. 그러기 위해서는 `test_func`을 타고 들어가서 어떤 값을 뽑아서 사용하는지 확인해야 합니다.

이렇게 명확하지 않는 부분이 생겨나는 것은 바람직하다고 생각하지 않습니다. 프로그램 구조가 아주 간단할 때는 큰 문제가 되지 않을지 몰라도, 이런 사항 하나가 나중에 리팩토링에 큰 번거로움을 가져올 수 있기 때문입니다.

<br>

# 2. macro - ds

```python
{% raw %}
def test_func(arg1, arg2, execution_date):
    print(arg1)
    print(arg2)
 	  print(execution_date) # YYYY-MM-DD


test_task = PythonOperator(
      task_id=f"{name}_task",
      python_callable=test_func,
      op_kwargs={
        "arg1":"hello",
        "arg2": "airflow!",
        "execution_date": "{{ds}}"
      },
      provide_context=True,
      queue="s21",
      dag=dag,
    )
{% endraw %}
```

airflow의 매크로 기능을 가지고 여러 변수들을 바로 참조할 수 있습니다. 주로 jinja template을 이용해 바로 shell 파일을 실행할 때 사용하지만 위처럼 PythonOperator에서 함수에 인자로 넘겨줄 때도 사용할 수 있습니다. 위에서 사용한 `{% raw %}"{{ ds }}"{% endraw %}` 는 실행 시간을 str 타입 / YYYY-MM-DD 형태로 가져옵니다. 이 외에도 어제 / 내일 / YYYYMMDD 형태 등등 여러가지를 사용할 수 있습니다.  어떤 변수들을 매크로 형태로 참조할 수 있는지는 airflow 공식문서의 macros reference를 참고하시기 바랍니다.
https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html

1번에서 context로 메서드에서 바로 execution_date를 참조하는 것보다는 Operator / 메서드가 어떤 것을 주고받고 있는지가 더 명확해졌습니다.

<br>

# 3. macro - execution_date

ds의 가장 큰 문제는 가져올때부터 str이기 때문에 시간 단위의 연산이 어렵다는 점입니다. 

- 2020-12-31이라는 str을 넘겨 받았을 때 단순하게 year / month / day 단위로 나눈 뒤 day에 1을 더해 2020-12-32가 된다면 오류!
- 년 / 월 / 일까지는 받았는데.. 시간 / 분 단위는?

그래서 저는 execution_date를 사용하는 것을 권장합니다.

```python
{% raw %}
def test_func(arg1, arg2, execution_date):
    print(arg1)
    print(arg2)
 	  print(execution_date) # e.g : 2018-01-01T00:00:00+00:00


test_task = PythonOperator(
      task_id=f"{name}_task",
      python_callable=test_func,
      op_kwargs={
        "arg1":"hello",
        "arg2": "airflow!",
        "execution_date": "{{execution_date}}"
      },
      provide_context=True,
      queue="s21",
      dag=dag,
    )
{% endraw %}
```

메서드에서 execution_date를 받아올 때는 이미 str으로 인식하기 때문에 "2018-01-01T00:00:00+00:00"을 datetime으로 변환해 시간 연산을 하는 것은 매우 번거롭습니다.어제 / 오늘 등 기본적인 건 airflow macro에서 제공하기 때문에 그대로 사용하셔도 좋고, KST 변환 등 airflow에서 기본적으로 제공하지 않는 연산은 이렇게 사용할 수 있습니다.



```python
{% raw %}
def test_func(arg1, arg2, execution_date):
    print(arg1)
    print(arg2)
 	  print(execution_date) # e.g : 2018-01-02


test_task = PythonOperator(
      task_id=f"{name}_task",
      python_callable=test_func,
      op_kwargs={
        "arg1":"hello",
        "arg2": "airflow!",
        "execution_date": "{{execution_date.in_timezone('Asia/Seoul').strftime('%Y-%m-%d')}}"
      },
      provide_context=True,
      queue="s21",
      dag=dag,
    )
{% endraw %}
```

execution_date 변수는 macro 안에서는 pendulum 클래스 타입이기 때문에 ([공식 문서 참조](https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html)) 시간 연산이 용이합니다. 이렇게 사용하면 `python_callable` 메서드에 원하는 형태 / 원하는 시간의 execution_date를 바로 넘겨줄 수 있고 어떤 형태로 넘기고자 하는지도 명확하게 확인이 가능합니다.



airflow의 ETL 스케줄에서 논리적 실행시간인 execution_date를 이해하는 것은 매우 중요합니다. 메인 로직 뿐만 아니라 재처리나 장애 대응 또한 exectution_date를 기준으로 동작하기 때문에 이를 잘 활용해야겠습니다.