---
title: "[spark] spark sql"
date: 2021-05-19 00:00:00
categories:
  - spark
tags: [spark]
---

처음에 spark sql은 이름만 보고 스파크에서도 sql을 사용할 수 있구나! 정도로 생각했지만 spark sql은 이보다 좀 더 큰 범위의 기능을 제공하고 있습니다. spark sql은 단순히 sql을 사용하는 것은 물론, sql과 비슷한 개념으로 dataframe / dataset에서 데이터를 처리할 수 있도록 여러 메서드도 제공합니다.

spark sql이 생겨난 배경과 그 기능을 살펴보겠습니다.



# RDD

스파크에서 가장 먼저 사용한 데이터 모델은 RDD입니다. RDD는 block 단위로 데이터를 데이터를 읽어들이고, 이는 스파크의 파티션 단위가 됩니다. 스파크 자체가 하둡의 파일 입출력 API에 의존성을 갖고 있기 때문에 RDD 또한 데이터를 읽고 쓰는데 맵리듀스처럼 쪼개기, 합치기가 진행됩니다.

RDD에 대해 자세히 다루지는 않겠지만, 

- RDD는 스파크의 근간이 되는 데이터 모델이다.
- RDD는 메모리를 기반으로 분산처리를 한다.
- RDD는 많은 메서드를 제공해 개발자가 편하게 연산을 할 수 있다.

위와 같은 사실은 RDD를 사용해보았다면 알 수 있습니다.

그렇다고 해서 RDD가 단점이 없는 것은 아닙니다. 그 중 하나가 데이터의 스키마, 즉 컬럼을 만들 수 없다는 것입니다. 데이터의 타입이나 Nullable의 여부 등 데이터의 정보를 표현할 마땅한 방법이 없다는 것은 꽤나 불편했기 때문에, 이를 보완하기 위해 Spark SQL이 등장하게 됩니다.

<br/>

# Spark SQL

위에서도 언급했지만 spark sql은 sql을 이용해 연산을 할 수 있을뿐 아니라 dataframe / dataset 환경 또한 제공합니다. 그렇기 때문에 Spark SQL은 하나의 개념이라고 보는 것이 더 맞다고 생각합니다.

dataframe과 dataset은 완전히 다른 것이 아닙니다. 처음에는 언어에 관계없이 dataframe을 가지고 연산을 했으나, 이후에 dataset이 등장하고 스파크 2.0부터 dataframe이 dataset 클래스로 통합됨에 따라

- 스칼라는 dataframe과 dataset을 동시에 사용

  - ```scala
    class Dataset[T] // Dataset in scala
    type DataFrame = Dataset[Row] // DataFrame in scala
    ```

  - Dataset의 인자가 Row 타입인 것이 DataFrame

  - 그 외(ex DataSet[Int]..)는 DataSet

- 자바는 DataSet을 사용

- R, Python은 DataFrame을 사용

언어에 따라 위와 같은 사용환경을 갖게 되었습니다. 저는 pyspark을 주로 사용하기 때문에 Spark SQL에서 제공하는 데이터 모델은 DataFrame이라고 하겠지만 환경에 따라 이는 Dataset이 될수도, DataFrame이 될 수도 있습니다.

<br/>

# DataFrame

DataFrame은 데이터셋의 구성요소가 Row인 DataSet입니다. 그렇기 때문에 Python에서도 DataFrame은 Row로 이루어져 있어야 합니다. 

DataFrame은

- 특정 데이터 파일(txt, csv, parquet...) 
- 기존에 가지고 있는 Sequential 데이터(list, tuple...)
- RDD

로 만들 수 있습니다.

```python
from pyspark.sql import Row

data = [
    Row(1, 'jack'),
    Row(2, 'sally')
]

columns = ['number', 'name']

dataframe = spark.createDataFrame(data, columns)
```



DataFrame과 RDD의 가장 큰 차이점은 스키마를 제공하는 것입니다. 스키마는 다음과 같이 확인할 수 있습니다.

```python
df.printSchema()

root
 |-- number: long (nullable = true)
 |-- name: string (nullable = true)
```

<br/>

# SparkSession

RDD를 SparkContext에서 만들었던것처럼, DataFrame은 SparkSession을 가지고 만듭니다. pyspark shell을 사용한다면 spark이라는 변수에 이미 SparkSession 객체가 만들어져 있습니다. 그렇지 않다면 builder 메서드를 이용해 SparkSession 인스턴스를 생성할 수 있습니다.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder().master('local[1]').appName('learning_spark').getOrCreate()
```

처음에는 왜 pyspark에 sql이라는 이름으로 라이브러리가 만들어졌는지 이해가 가지 않았는데, spark sql의 개념을 알고 나니 왜 그런지 바로 알 수 있습니다.

sparksession이 사용되기 이전(spark 2.0 이전)에 작성된 스파크 어플리케이션과 호환을 위해 SparkSession에서 SparkContext를 만들수도 있습니다.

```python
spark = SparkSession.builder \
    .config(conf=conf) \
    .appName("Learning_Spark") \
    .getOrCreate()

spark_context = spark.sparkContext
```

