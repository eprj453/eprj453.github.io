---
title: "[Spark] Window function in Pyspark"
date: 2021-01-29 00:00:00
categories:
- spark
tags: [spark]
---



window function은 행을 기준으로 하는 연산입니다. 이전 row, 다음 row에 접근할 수 있고 그 범위를 정해 그 안에서 여러 연산을 수행할 수 있습니다.

[wikipedia - SQL window function](https://en.wikipedia.org/wiki/SQL_window_function)

행을 돌면서 할 수 있는 것들로는

- 순위
- 분석
- 집계 및 누적 연산

정도를 예시로 들 수 있겠습니다.

sql window와 spark window의 개념이 다르지는 않습니다. 분석은 잘 몰라서 건너뛰고, 순위 매기는 것과 누적 연산을 pyspark에서 해보겠습니다.

사전 환경 및 sample data, dataframe는 다음과 같습니다.

```python
from pyspark.sql import *
from pyspark.sql import functions as f
from pyspark.sql import window

spark = SparkSession\\
    .builder\\
    .appName('sample')\\
    .master('local[3]')\\
    .getOrCreate()

sample_list = [
    ('watcha', 5, 7500),
    ('watcha', 2, 12000),
    ('watcha', 3, 12000),
    ('watcha', 12, 7500),
    ('Netflix', 4, 7000),
    ('Netflix', 10, 18000),
    ('Netflix', 2, 7000),
    ('Netflix', 5, 7000),
    ('Netflix', 6, 12000),
    ('Tving', 1, 11000),
    ('Tving', 10, 7000),
    ('Tving', 5, 11000),
    ('Wavve', 5, 7000),
    ('Wavve', 4, 10000),
    ('Wavve', 3, 10000),
    ('Wavve', 9, 7000),
]

columns = ['platform', 'month', 'price']

df = spark.createDataFrame(sample_list).toDF(*columns)
```

<br/>

# 순위 매기기

sql에서 rank, row_number의 역할과 동일합니다. platform 별로 파티션을 나누고, 총 이용금액을 기준으로 순위를 매겨보겠습니다.

```python
rank_function = f.rank()\\
        .over(Window.partitionBy('platform')
        .orderBy(f.expr('month * price')
        .desc()))

rank_df = df\\
    .withColumn('totalPrice', f.expr('month * price'))\\
    .withColumn(
        'rank', rank_function
    )
+--------+-----+-----+----------+----+
|platform|month|price|totalPrice|rank|
+--------+-----+-----+----------+----+
|   Tving|   10| 7000|     70000|   1|
|   Tving|    5|11000|     55000|   2|
|   Tving|    1|11000|     11000|   3|
|  watcha|   12| 7500|     90000|   1|
|  watcha|    5| 7500|     37500|   2|
|  watcha|    3|12000|     36000|   3|
|  watcha|    2|12000|     24000|   4|
|   Wavve|    9| 7000|     63000|   1|
|   Wavve|    4|10000|     40000|   2|
|   Wavve|    5| 7000|     35000|   3|
|   Wavve|    3|10000|     30000|   4|
| Netflix|   10|18000|    180000|   1|
| Netflix|    6|12000|     72000|   2|
| Netflix|    5| 7000|     35000|   3|
| Netflix|    4| 7000|     28000|   4|
| Netflix|    2| 7000|     14000|   5|
+--------+-----+-----+----------+----+
```

withColumn 메서드로 totalPrice, rank 컬럼을 새로 만들었습니다.

<br/>

rank 컬럼을 만드는 함수인 rank_function에는 rank() 메서드에서 필요로 하는 partitionBy, orderBy 등의 조건이 들어가 있습니다.

```sql
rank () over (partiton by platform, order by month * price desc)
```

위 sql expression과 동일하게 동작합니다.

<br/>

totalPrice는 month * price이고, totalPrice가 큰 순서대로 순위를 매겼습니다. platform을 기준으로 파티션이 분리된 채로 각 파티션 별로 연산이 수행된 결과를 볼 수 있습니다.

<br/>

# 누적 합계 연산

platform 별로 totalPrice 누적합을 구해보겠습니다.

```python
cumulative_sum_function = f.sum('totalPrice').over(
    Window.partitionBy('platform').orderBy((f.expr('totalPrice desc')))
)

cumulativeSum_df = df\\
    .withColumn('totalPrice', f.expr('month * price'))\\
    .withColumn('accumulate_sum', cumulative_sum_function)

cumulativeSum_df.show()
+--------+-----+-----+----------+--------------+
|platform|month|price|totalPrice|accumulate_sum|
+--------+-----+-----+----------+--------------+
|   Tving|    1|11000|     11000|         11000|
|   Tving|    5|11000|     55000|         66000|
|   Tving|   10| 7000|     70000|        136000|
|  watcha|    2|12000|     24000|         24000|
|  watcha|    3|12000|     36000|         60000|
|  watcha|    5| 7500|     37500|         97500|
|  watcha|   12| 7500|     90000|        187500|
|   Wavve|    3|10000|     30000|         30000|
|   Wavve|    5| 7000|     35000|         65000|
|   Wavve|    4|10000|     40000|        105000|
|   Wavve|    9| 7000|     63000|        168000|
| Netflix|    2| 7000|     14000|         14000|
| Netflix|    4| 7000|     28000|         42000|
| Netflix|    5| 7000|     35000|         77000|
| Netflix|    6|12000|     72000|        149000|
| Netflix|   10|18000|    180000|        329000|
+--------+-----+-----+----------+--------------+
```

rank와 방식이 크게 다르지는 않습니다. 동일하게 window function을 이용하고, spark function을 rank를 쓰느냐 sum을 쓰느냐의 차이만 있습니다.

<br/>

# rowsBetween

[rowsBetween 공식 문서](https://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.Window.rowsBetween)

rowsBetween은 window의 범위를 정하는 역할입니다. 윈도우에 포함될 행을 정하는 것이기도 합니다. 개인적으로 window function에서 제일 이해하는데 오래 걸렸던 부분입니다.

매개변수는 2개를 갖는데, 시작과 끝을 명시해 범위를 정합니다.

```python
rowsBetween(start, end)
```

start와 end에는 숫자가 들어가는데, 0을 기준으로 숫자가 줄어들면 이전 row, 숫자가 늘어나면 다음 row를 window 범위로 정합니다.

<br/>

즉,

```python
rowsBetween(-2, 2)
```

이렇게 범위를 지정하면 현재 row를 기준으로 전전 row, 다다음 row까지를 window의 범위로 지정합니다.

<br/>

window 범위에 따라 연산의 결과가 어떻게 달라지는지 누적 합계 연산에 적용시켜 확인하겠습니다. rowsBetween은 over 안에 조건에 추가해주면 됩니다.

```python
cumulative_sum_function = f.sum('totalPrice').over(
    Window.partitionBy('platform').orderBy((f.expr('totalPrice desc'))).rowsBetween(-2, 2)
)
+--------+-----+-----+----------+--------------+
|platform|month|price|totalPrice|accumulate_sum|
+--------+-----+-----+----------+--------------+
|   Tving|    1|11000|     11000|        136000|
|   Tving|    5|11000|     55000|        136000|
|   Tving|   10| 7000|     70000|        136000|
|  watcha|    2|12000|     24000|         97500|
|  watcha|    3|12000|     36000|        187500|
|  watcha|    5| 7500|     37500|        187500|
|  watcha|   12| 7500|     90000|        163500|
|   Wavve|    3|10000|     30000|        105000|
|   Wavve|    5| 7000|     35000|        168000|
|   Wavve|    4|10000|     40000|        168000|
|   Wavve|    9| 7000|     63000|        138000|
| Netflix|    2| 7000|     14000|         77000|
| Netflix|    4| 7000|     28000|        149000|
| Netflix|    5| 7000|     35000|        329000|
| Netflix|    6|12000|     72000|        315000|
| Netflix|   10|18000|    180000|        287000|
+--------+-----+-----+----------+--------------+
```

rowsBetween을 적용하고 나니, 뭔가 이상한 누적합 결과가 나왔습니다. 한 번에 보면 정신이 없으니까 Netflix만 따로 떼서 보겠습니다.

<br/>

```python
+--------+-----+-----+----------+--------------+
|platform|month|price|totalPrice|accumulate_sum|
+--------+-----+-----+----------+--------------+
| Netflix|    2| 7000|     14000|         77000|
| Netflix|    4| 7000|     28000|        149000|
| Netflix|    5| 7000|     35000|        329000|
| Netflix|    6|12000|     72000|        315000|
| Netflix|   10|18000|    180000|        287000|
+--------+-----+-----+----------+--------------+
```

첫 누적합 결과는 77000입니다. 왜일까요?

<br/>

아까 우리는 rowsBetween 메서드로 window 범위를 내 전전 row, 다다음 row까지로 정했습니다. partition을 나누어서 연산하고 있기 때문에 첫번째 row의 전전 row는 없습니다. 그렇다면 누적합이 실행되는 범위는 다음과 같습니다.

```python
+--------+-----+-----+----------+--------------+
|platform|month|price|totalPrice|accumulate_sum|  # 전전 row 없음
+--------+-----+-----+----------+--------------+  # 전 row 없음
| Netflix|    2| 7000|     14000|         77000|  # 현재 row!
| Netflix|    4| 7000|     28000|        149000|  # window 범위 안
| Netflix|    5| 7000|     35000|        329000|  # window 범위 안
| Netflix|    6|12000|     72000|        315000|  # window 범위 밖
| Netflix|   10|18000|    180000|        287000|  # window 범위 밖
+--------+-----+-----+----------+--------------+
```

window 범위 안의 모든 totalPrice의 합(14000 + 28000+ 35000)이 첫 번째 row의 누적합 결과가 됩니다.

<br/>

두번째 누적합 결과도 볼까요?

```python
+--------+-----+-----+----------+--------------+
|platform|month|price|totalPrice|accumulate_sum|  
+--------+-----+-----+----------+--------------+  # 전전 row 없음
| Netflix|    2| 7000|     14000|         77000|  # window 범위 안
| Netflix|    4| 7000|     28000|        149000|  # 현재 row!
| Netflix|    5| 7000|     35000|        329000|  # window 범위 안
| Netflix|    6|12000|     72000|        315000|  # window 범위 안
| Netflix|   10|18000|    180000|        287000|  # window 범위 밖
+--------+-----+-----+----------+--------------+
```

row가 한 칸 내려오면서, 전 row가 생겼습니다. 그렇다면 범위에 들어있는 모든 row의 totalPrice의 합(14000 + 28000 + 35000 + 72000)이 두 번째 row의 누적합 결과가 됩니다.