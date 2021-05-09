---
title: "[Spark] Execution Plan: Stage, Task"
date: 2021-01-08 00:00:00
categories:
- spark
tags: [spark]
---



Stage가 나눠지는 기준은 **Job 안에 repartition이 얼마나 들어있는가**, Task갯수는 **repartition으로 인해 몇 개의 partition으로 나눠졌는가**입니다. Executer가 각 Task를 처리함으로써 RDD가 분산처리됩니다.

코드를 보겠습니다.

```python
df = load_survey_df(spark, sys.argv[1])
partitioned_df = df.repartition(3)
```

repartition() 메서드는 dataframe의 파티션을 임의의 개수로 나눕니다. 파티션이 나누어지면, 다음 Stage가 생성되고 그 stage의 갯수는 파티션의 개수가 될 것입니다.

<br/>

repartition에 의해 stage가 2개가 되었고, 2번째 Task의 개수는 repartition 갯수에 따라 3개인 것을 볼 수 있습니다.

![image](https://user-images.githubusercontent.com/52685258/103870303-dd5a3280-510e-11eb-9d40-a9707d35b6e6.png)

<br/>

repartition을 하나 더 추가해보겠습니다.

```python
df = load_survey_df(spark, sys.argv[1])
partitioned_df = df.repartition(3)
result_df = partition_df.filter("Age < 40") \
        .select("Age", "Gender", "Country", "state") \
        .groupBy("Country") \
        .count()
        
result.show()
```

groupby 메서드를 사용하면, shuffle에 의해 repartition이 발생합니다. 몇 개로 나눠지는지에 대한 내부 구조까지는 알 수 없고, 값을 설정해 갯수를 설정할수 있습니다.

```python
spark_conf = SparkConf().set("spark.sql.shuffle.partitions", "5")
```

Spark session을 만드는 config에 직접 설정하거나 conf 파일을 따로 만든 뒤

 `spark.sql.shuffle.partitions = 개수` 

이렇게 설정해줘도 됩니다. 

<br/>

앞서 repartition은 3개였고, shuffle partition의 갯수는 5개로 설정해보겠습니다.

![image](https://user-images.githubusercontent.com/52685258/103881386-b7885a00-511d-11eb-8967-cc7e6798e9aa.png)

![image](https://user-images.githubusercontent.com/52685258/103881423-c242ef00-511d-11eb-83b7-96763f02c660.png)

stage는 3개, Task는 stage별로 1개, 3개, 5개 총 9개입니다.

위 코드가 실행되면서 DataFrame은 이러한 전체 과정을 겪게 됩니다.![image](https://user-images.githubusercontent.com/52685258/103874914-1a292800-5115-11eb-8ac0-e2cd3f8f5b0e.png)

