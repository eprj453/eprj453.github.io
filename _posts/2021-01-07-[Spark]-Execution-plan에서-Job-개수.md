---
title: "[Spark] Execution Plan: Job"
date: 2021-01-07 00:00:00
categories:
- spark
tags: [spark]
---

데이터에 관련된 업무를 하다보니 자연스럽게 Spark에 대해서 공부하고 있습니다. 아직까지는 내부 동작원리와 같은 복잡한 내부구조는 잘 모르고, 결과과 현상에 대해서만 얕게 체험해보는 중입니다. 

Execution plan을 보다가 잘 이해가 되지 않아 개인적으로 공부하며 이에 대해 기록합니다.

공부기록이므로 틀린 내용이 있을 수 있습니다.

<br/>



# Job

Spark 애플리케이션에서 RDD 생성 ~ Action 적용까지를 하나의 Job이라는 단위로 처리합니다.  Job은 다시 Stage / Task 단위로 나뉘게 됩니다.

Action이 없으면 Job으로 만들어지지 않기 때문에, Job 개수의 기준은 코드에서 **Action이 얼마나 있는가**입니다.

<br/>

이런 코드가 있다고 합시다.

```python
spark = SparkSession.builder \
        .config(conf=conf) \
        .getOrCreate()

df = spark.read.csv('sample.csv')
```

csv 파일을 읽어서 dataFrame으로 읽어왔습니다. 이 때 하나의 Job이 생성됩니다. 모든 Job은 적어도 1개의 Stage를 갖고 모든 Stage는 적어도 1개의 Task를 가지므로, 특별한 일이 없다면 이 Job은 Job : Stage : Task = 1:1:1이 될 것입니다.

<br/>

만약 df를 작성하는 코드가 이런 코드라면 어떨까요?

```python
df = spark
	.option("header", "true")
	.read.csv('sample.csv')
```

option은 그 자체로 Object를 바꾸게 됩니다. Transformation에 해당하는 각종 작업들(where, select....)들이 기존 Object는 그대로 놔두고 결과로 새로운 DataFrame을 반환하는 것과는 다른 작업입니다.

그래서 DataFrame에 Object를 바꾸는 option이 추가된다면 Job 또한 추가로 생성됩니다.

![image](https://user-images.githubusercontent.com/52685258/103785633-c8ce5980-507e-11eb-9275-c1b326934214.png)

option으로 인해 생겨난 Job의 DAG인데, DeserializeToObject가 Object를 조작하는 부분이 아닐까 생각합니다. Spark UI의 DAG을 보면 코드의 작동과정을 유츄해볼 수 있습니다.

<br/>Age, Gender, Country, state 컬럼을 가진 간단한 csv 파일을 읽어온 뒤, Transformation을 추가해 Job의 갯수를 확인하겠습니다.

```python
def load_df(spark, file_path):
    return spark.read \
        .option("header", "true") \
        .csv(file_path)
        
def transformation_df(data_frame):
    return data_frame.filter("Age < 40") \
        .select("Age", "Gender", "Country", "state")
    
spark = SparkSession.builder \
        .config(conf=conf) \
        .getOrCreate()   
        
df = load_survey_df(spark, 'sample.csv')
selected_df = transformation_df(df)
```

DataFrame에 select와 filter Transformation이 추가되었습니다. Job의 개수는 어떨까요?

![image](https://user-images.githubusercontent.com/52685258/103788895-b6eeb580-5082-11eb-8bcc-22a9ca8dcc75.png)

DataFrame에 Transformation이 적용된다고 해서 Job이 늘어나지는 않습니다. 추가로 Job을 생성하려면 역시 Action이 필요합니다.

<br/>

collect를 추가해보겠습니다. collect는 Transformation의 결과를 list로 반환해줍니다. 

```python
df = load_survey_df(spark, 'sample.csv')
selected_df = transformation_df(df)
result_list = selected_df.collect()
```

![image](https://user-images.githubusercontent.com/52685258/103787169-9aea1480-5080-11eb-9dfd-a6fea6d88acf.png)

Job이 하나 추가되어 총 3개의 Job이 만들어졌습니다. 

<br/>

Description에는

- csv 파일을 읽어오며 생긴 Job 0, 1
- spark application에서 일어난 collect에 의해 생긴 Job 2

총 3개의 Job이 생겨난 근거를 간략하게 나와 있습니다.

<br/>

Job의 갯수는 이러한 과정에 의해 결정됩니다.