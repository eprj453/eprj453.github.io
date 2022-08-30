---
title: "[Spark] repartition, coalesce 속도 비교"
date: 2022-08-30 00:00:00
categories:
- spark
tags: [spark, data-enginnering]
---

저는 업무에서 AWS를 사용해 Spark Job을 돌리고 있습니다. 마지막에 파티션을 병합해야 결과 파일이 하나로 모이기 때문에, 파티션 1개로 병합하는 작업을 거칩니다. 이 때 사용할 수 있는 메서드가 파티션 갯수를 조절하는 repartition, coalesce 입니다.

간단하게 말하면 파티션을 병합하는 과정에서 repartition은 셔플을 실행하고, coalesce는 셔플을 실행하지 않습니다. 이를 비교해놓은 포스팅이나 stackoverflow 글은 상당히 많기 때문에 차이점을 깊게 언급하지는 않겠습니다.

파티션을 줄이는 과정에서는 coalesce, 파티션을 늘려야 하는 경우는 repartition을 쓴다고 알고 있었고 틀렸다고도 생각하지는 않습니다. 

그렇지만 Spark Document에서도 예외를 두고 있는 경우가 있습니다.

> However, if you’re doing a drastic coalesce, e.g. to numPartitions = 1, this may result in your computation taking place on fewer nodes than you like (e.g. one node in the case of numPartitions = 1). To avoid this, you can call repartition(). This will add a shuffle step, but means the current upstream partitions will be executed in parallel (per whatever the current partitioning is).

이번에 마주했던 문제상황도 numPartitions를 1로, 즉 파티션을 하나로 만드는 과정에서 발생했습니다. 처리하는 데이터의 양이 많아지면서 Job 실행시간이 실패 기준으로 정했던 시간을 넘어가 버린 것입니다.

이 문제는 coalesce 메서드를 repartition 메서드로 변경하고 해결되었으나, shuffle을 하지 않음에도 더 많은 시간이 걸린다는 것이 이해가 잘 가지 않아 리니지와 실행 계획들을 살펴봤습니다.

그 내용을 기록합니다

<br/>

# 문제 상황

spark에서 group by나 join과 같은 wide transformation이 일어나는 경우, 파티션 값을 따로 설정하지 않으면 파티션 갯수가 200개로 늘어납니다. 이대로 RDD나 Dataframe을 바로 파일로 떨구면 파티션의 갯수와 동일하게 여러 파일로 나눠서 저장됩니다.

이를 막기 위해 파티션을 1로 만들어야 했고, coalesce 메서드를 사용했습니다. repartition으로도 가능하지만 repartition은 전체 파티션을 대상으로 셔플이 들어가기 때문에 셔플이 없는 coalesce이 더 나을거라 생각했습니다.

그런데 실제로는 repartition이 훨씬 빠른 속도를 보였습니다.

<img width="1147" alt="image" src="https://user-images.githubusercontent.com/52685258/180640116-015570ae-bf7a-4363-a619-84d516a61222.png">

이를 이해하기 위해서는 spark에서 partition이 무엇인지, shuffle이 정확히 어떻게 동작하는지 자세히 볼 필요가 있습니다.

<br/>

# spark partition

RDD / dataframe / dataset을 이루는 최소 단위입니다. 즉 RDD는 n개의 파티션으로 이루어져 있습니다. 각 파티션은 서로 다른 노드에서 분산 처리됩니다.

파티션은 모든 Executor Node의 core수만큼 분산됩니다. 즉 Executor Node가 5개고 각 Node의 Core가 2개라면, 기본적으로 RDD는 10개의 파티션으로 쪼개집니다.

파티션 하나는 하나의 core에서 처리합니다. 만약 파티션이 200개인 dataframe이 있다 하더라도 Executor 총 core가 10개라면, 병렬성은 10을 넘을 수 없습니다.

또한 Spark Job의 연산의 최소 단위를 Task라고 표현하는데, 이 Task 또한 하나의 파티션을 처리합니다. 즉 1 Core = 1 Partition = 1 Task라고 볼 수 있습니다.

파티션 갯수를 많이 잡으면 병렬성이 늘어나고 Task 1개당 필요한 메모리가 줄어듭니다. 예를 들어 100mb짜리 dataframe을 처리한다고 해보겠습니다.

모든 Executor Node의 총합이 10 core / 메모리 32gb라고 했을때,

- 10개 파티션으로 분해 시 => 1 파티션은 10mb이고, 사용 리소스는 1 core / 3.2gb

- 5개 파티션으로 분해 시 => 1 파티션은 20mb이고, 사용 리소스는 1 core / 6.4gb



물론 Node 마다 리소스가 흩어져 있고 파티션이 리소스를 사용하는 방식이 실제로 이럴리는 없습니다. 그렇지만

- 파티션의 갯수가 많을수록 core가 더 필요하다
- 파티션의 크기가 클수록 메모리가 더 필요하다

라는 것을 개념적으로 이해할 수 있습니다. 또한

- 총 core 갯수보다 파티션 갯수를 작게 잡는 것은 일반적으로 좋지 않다.

라는 것도 생각해볼 수 있습니다.

<br/>

# repartition, coalesce



## 스크립트 코드

```python
df = spark.read.format("json").load(target_paths).select(F.explode("e")).select(F.col("col.*"))


timestamp_to_datetime_udf = udf(lambda x: timestamp_to_datetime(x), DateType())

df = df.select(get_nested_column_in_struct_type(df.schema))\
    .withColumn('s3_upload_time', F.lit(datetime.now()).cast(TimestampType()))\
    .withColumnRenamed('timestamp', 'event_time').withColumn('event_time', timestamp_to_datetime_udf(F.col('event_time')).cast(TimestampType()))\
    .withColumnRenamed('event_properties.1차 카테고리명', 'event_properties.1차_카테고리명') \
    ...
    ...
    ...
    .withColumn('sequence_number', F.col('sequence_number').cast(StringType())) \
    .withColumn('api_level', F.col('api_level').cast(DoubleType()))


impression_df = df.select('*').where(F.col("`event_properties.액션_타입`") == '노출')


product_df = impression_df.select([F.col(product_column) for product_column in product_columns]).where(F.col("`event_properties.상품_번호`").isNotNull()).repartition(1)

banner_df = impression_df.select([F.col(banner_column) for banner_column in banner_columns]).where(F.col("`event_properties.배너`").isNotNull()).repartition(1)

banner_df.write.parquet(f"s3://...")
product_df.write.parquet(f"s3://...")

```

불필요한 부분은 생략하고, 

- spark.read로 json => dataframe으로 읽은 뒤 필요한 부분만 1차 select (action 발생)
- 컬럼 형변환 및 이름 변경
- 특정 컬럼 값이 `노출` 인것만 select
- where 조건에 따라 각각 df를 생성하고 파티션을 하나로 병합 (각각 action 발생)
- write.parquet로 s3에 df write (각각 action 발생)



동일한 로직에서 병합 메서드에 따른 시간 차이는 다음과 같습니다.

![image-20220818005019425](/Users/psw/Library/Application Support/typora-user-images/image-20220818005019425.png)

<img width="1092" alt="image" src="https://user-images.githubusercontent.com/52685258/184083927-121ee684-72ce-4417-8623-d6ca58c1a529.png">



<br/>

## repartition

먼저 repartition으로 파티션을 병합한 경우입니다. UI를 통해 Job이 몇 개가, 어떻게 실행되었는지 살펴보겠습니다.

![image](https://user-images.githubusercontent.com/52685258/184756631-3de5b106-00f3-473f-a52a-8dc2f9ddd8f7.png)



Job 흐름에 따라

`0, 1, 2 / 3, 4, 5, 6`

총 3단계로 나눠보겠습니다.

<br/>

### 0, 1, 2 -> 데이터 스캔 및 DF 생성 단계

json을 읽는 단계에서 3개의 job이 생성된 것은 조금 의아합니다. 총 956개의 json 파일을 스캔하는 작업인데, 원래 실제 df를 생성하는 read 단계와 path, format 등을 지정하는 option 단계는 각각 따로 job을 만들어 2개의 job이 만들어질거라 생각했는데 총 3개의 job이 만들어져 실행되었습니다.

로그를 봐서도 0번과 1번 Job, 그리고 Job이 갖고 있는 Stage의 로그를 보더라도 동작이 동일한데, 왜 2개의 job이 만들어졌는지는 잘 모르겠습니다.

뒤에 붙은 select 단계는 transformation 단계이기 때문에 Job을 생성하지 않습니다.

<br/>

### 3, 4, 5, 6 -> DF write 단계

df를 parquet로 쓰는 단계입니다. df가 2개이기 때문에 action이 2개 job도 2개일거라 생각했는데 하나의 df write에 2개의 job이 생성되어 총 4개가 생겼습니다. 데이터를 쓰는 단계에서 coalesce와 시간 차이가 나는만큼 이 단계를 봐야할것 같습니다.

여기서 3, 4 / 5, 6은 각각 다른 df를 write하는 작업이기 때문에 내부 동작은 3, 4와 5, 6이 동일할 것입니다. 시간이 더 오래 걸리고 크기가 큰 df를 write한 5, 6 과정만 보겠습니다.

<br/>

### repartition and write

<img width="2975" alt="image" src="https://user-images.githubusercontent.com/52685258/187478835-ee3cd8b7-a41d-4a06-ba36-76907e8eebad.png">

<img width="2977" alt="image" src="https://user-images.githubusercontent.com/52685258/187479013-872df805-d494-4c57-ba31-74020d6740e1.png">



순서대로 Job 5, Job 6의 spark ui입니다. 처음에는 shuffle write와 shuffle read가 실행되고, shuffle query 실행 이후에 exchange라는 과정이 추가로 발생하면서 Job 6번이 생긴 것을 볼 수 있습니다. Exchange가 실행된 Stage 8번을 보면, 1 partition으로 이루어진 ShuffledRowRDD가 생성된 것을 볼 수 있습니다.

executor 간에 파티션을 재정렬하기 위해 모든 executor가 컴퓨팅 리소스를 사용했고, 하나의 ShuffledRowRDD로 다시 병합되어 S3 write가 이루어졌음을 알 수 있습니다.

<br/>

## coalesce

coalesce로 파티션을 병합한 경우입니다.

![image-20220831004524816](/Users/psw/Library/Application Support/typora-user-images/image-20220831004524816.png)



repartition에 비해 단계가 줄었습니다. load까지는 repartitionrhk 동일하지만 ShuffledRowRDD를 생성하지 않기 때문에 exchange를 위한 Job이 df 2개 분량만큼 줄어들어 job이 7 -> 5개로 줄어든 것을 볼 수 있습니다.

Job 3, Job 4가 df의 크기만 다를 뿐 동일한 작업을 했을 것이기 때문에 시간이 더 오래 걸린 Job 4만 보면 될 것 같습니다.

<br/>

### coalesce and write

![image](https://user-images.githubusercontent.com/52685258/187482953-6be9afdc-7478-4882-a8c4-243d795097d8.png)

Shuffle 과정이 없는건 그렇다 치고, Executor 인스턴스로 떠있던 10개의 인스턴스 중에서 8번만 남기고 모두 작업에서 제외됐습니다. repartition에서는 Shuffle이 발생해 executor 간의 데이터 이동이 일어나긴 했지만 각 Executor가 균등하게 컴퓨팅 리소스를 사용했던 반면, coalsece는 shuffle로 인해 발생하는 리소스(네트워크 비용, 컴퓨팅 리소스)는 들지 않을지 몰라도 하나의 Executor가 모든 spark 연산을 처리한 것을 볼 수 있습니다.

<br/>

## Aggregated Metrics 비교

### repartition

<img width="2917" alt="image" src="https://user-images.githubusercontent.com/52685258/187484221-458fe7f8-e39f-4712-8aaa-51792a648cdf.png">

<br/>

### coalesce

![image](https://user-images.githubusercontent.com/52685258/187484280-bc7f01c6-be5a-472e-89ae-32e0edfcb4ac.png)



repartition에서 10개의 executor만 쓰인줄 알았는데 그게 아니라 11개였습니다. 사전에 Spark sql을 잘 작성해 데이터 크기를 최대한 줄이고 적절한 파티션 숫자를 사용하는게 당연히 연산의 효율에서 더 중요하겠지만, 이 정도의 executor 차이라면 repartition과 coalesce의 극명한 속도가 이해가 됩니다.

다음 포스팅에서는 왜 coalesce에서는 극단적으로 executor가 제한되는지, 그렇다면 coalesce는 어떻게 써야 적절할지 테스트해보고 작성해보도록 하겠습니다 :)









