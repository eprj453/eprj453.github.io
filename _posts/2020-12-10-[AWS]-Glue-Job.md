---
title: "[AWS] Glue Job"
Date: 2020-12-10 00:00:00
categories:
- AWS
tags: [AWS, data-engineering]
---

Crawler로 생성된 테이블을 기반으로 데이터를 추출, 가공, 적재하는 Job을 만들고 실행시켜보자.

![image](https://user-images.githubusercontent.com/52685258/101777716-39dd2880-3b36-11eb-9612-e9e5ef569a0b.png)

Job은 왼쪽 메뉴 ETL 항목에서 찾을 수 있다.

<br/>

# 1. Job properties

이름, IAM role, 실행 환경 등을 설정한다. 이해하기 어렵거나 매우 중요한 옵션은 없지만, 몇 가지 보고 넘어가자.

<br/>

![image-20201210222829524](C:\Users\zetca\AppData\Roaming\Typora\typora-user-images\image-20201210222829524.png)

Glue는 Spark을 기반으로 한다. Cluster 안에 driver node와 worker node가 1:N 관계로 묶여있는 구조이다. **Worker type**과 **Number of workers** 옵션은 이 구조를 조절하는 옵션이기 때문에 필요에 따라 조절하는 것이 좋다.

이 외에도 **Job timeout, Max concurrency** 옵션 등을 상황에 맞게 잘 설정해주면 효율적인 Job이 구성될 것이다.

<br/>

# 2. Data source

사전에 Crawler가 생성했던 메타 데이터 테이블 중 어떤 것을 바라볼지 선택한다. Crawler가 생성한 테이블이 1개라면 그걸 선택하고, 2개 이상이라면 어떤 것을 선택해도 무방하다.

<br/>

# 3. Transform type

![image](https://user-images.githubusercontent.com/52685258/101778934-e966ca80-3b37-11eb-8d64-75f3cdcfa8b1.png)

옵션이 2개가 있는데, 2번째 옵션인 머신러닝은 Glue 2.0에서는 지원하지 않는다. 첫번째 옵션이 스키마를 변경하고 새로운 데이터셋을 만드는 것이기 때문에 ETL의 목적에 더 부합하기도 하니 첫번째를 선택하면 된다.

<br/>

# 4. Data target

Data source에서 고른 테이블을 어떻게 변환시키고자 하는지, 결과가 될 테이블을 선택한다. 이 다음 Schema 메뉴를 보면 Data target을 어떻게 골라야할지 잘 알 수 있다.

<br/>

# 5. Schema

 ![image](https://user-images.githubusercontent.com/52685258/102008850-bfe7b200-3d76-11eb-81ed-bb9a6a43b037.png)

왼쪽에는 Data Source에서 골랐던 테이블의 컬럼, 오른쪽에는 Data target에서 골랐던 테이블의 컬럼이 있다. Transform 과정에서 컬럼 매핑을 어떻게 할 것인지 지정할 수 있으며, 컬럼은 추가, 삭제, 순서 변경이 가능해 원하는 output의 형태를 만들 수 있다. job을 만든 이후에 스크립트에서 직접 수정도 가능하다.

여기서는 id끼리만 매핑한 뒤 스크립트를 살펴보도록 하겠다.



# 스크립트 확인 / 수정

![image-20201213191626508](C:\Users\zetca\AppData\Roaming\Typora\typora-user-images\image-20201213191626508.png)

스크립트 수정화면인데, 스크립트가 딱 봐도 눈에 들어오지는 않는다. 우측 상단 메뉴에서 Transform을 이용하면 컬럼 추가, 삭제, 매핑 등 여러 가공과정을 추가할 수 있지만, spark 문법을 이용해 직접 스크립트를 작성하는걸 권장한다. 좀 더 직관적으로 스크립트를 작성할 수 있다.

 

# spark 스크립트

```python
from pyspark.context import SparkContext
import pyspark.sql.functions as f

# Import glue modules
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.dynamicframe import DynamicFrame
from awsglue.job import Job

# Glue Job ETL Script

# Initialize contexts and session
spark_context = SparkContext.getOrCreate()
glue_context = GlueContext(spark_context)
session = glue_context.spark_session

# parameter
glue_db = "test-parksw2" # catalog db 이름
id_name = "id_name" # table 이름
id_score = "id_score" # table
s3_write_path = "s3://s3-dev2-parksw2" # output을 저장한 s3 bucket

# 1. glue dynamic frame
name_dynamic_frame = glue_context.create_dynamic_frame.from_catalog(database=glue_db, table_name=id_name)
score_dynamic_frame = glue_context.create_dynamic_frame.from_catalog(database=glue_db, table_name=id_score)

# 2. spark data frame
name_df = name_dynamic_frame.toDF()
score_df = score_dynamic_frame.toDF()

joined_frame = name_df.join(score_df, name_df.id == score_df.id).filter(score_df.score > 80)

joined_frame_agg = joined_frame.repartition(1)

# 3. glue dynamic frame
dynamic_frame_write = DynamicFrame.fromDF(joined_frame_agg, glue_context, "dynamic_frame_write")

glue_context.write_dynamic_frame.from_options(
    frame=dynamic_frame_write,
    connection_type="s3",
    connection_options= {
        "path": s3_write_path,
    },
    format="csv"
)
```

위 스크립트에서는 id와 score를 묶은 뒤, score가 80점 이상인 row만 뽑아내도록 스크립트를 작성했다.



## data frame의 변환 과정

1. 처음 데이터를 가져올 때는 glue dynamic frame으로 가져온다(**# 1. glue dynamic frame**)
2.  spark data frame으로 변환한 뒤 데이터 가공, 연산을 한다 (**# 2. spark data frame**)
3. 데이터를 저장할 때는 다시 glue dynamic frame으로 변환한다(**# 3. glue dynamic frame**)



## repartition

glue는 apache spark을 기반으로 데이터를 처리한다. 분산 처리의 특성 상 여러 cluster들이 내부에서 작업을 조금씩 나누어 처리하게 되는데, 이렇게 작업한 결과는 여러 군데에 흩어져있기 때문에 한 군데로 모아주는 과정이 필요하다. **repartition(1)**을 이용해 하나의 frame으로 다시 묶어주었다.



## pyspark sql function

위에서 사용한 함수 뿐만 아니라 spark sql function에서 제공하는 모든 함수를 사용할 수 있기 때문에 필요에 따라 count, groupby 등등 함수를 사용해 작업을 하면 된다.



run job이 정상적으로 실행된다면, output으로 지정한 s3 버킷에 결과 파일이 저장되어 있을 것이다.

![image](https://user-images.githubusercontent.com/52685258/102009446-d2fc8100-3d7a-11eb-8c1f-ce93581749bd.png)

output 형식을 csv로 지정하긴 했는데, 저장될때는 그냥 파일로 저장된다. 다운로드 받을 때 다른 이름으로 저장 -> 파일 형식을 csv로 지정해줘야 한다.



![image](https://user-images.githubusercontent.com/52685258/102009502-1f47c100-3d7b-11eb-963e-0818a47d01cf.png)

파일을 확인해보면, 점수가 80점 이상인 column만 join이 되어 결과가 나온 것을 볼 수 있다.

다음에는 Data catalog와 Job을 가지고 다른 AWS 서비스에서 어떻게 이용하는지 확인할 것이다.