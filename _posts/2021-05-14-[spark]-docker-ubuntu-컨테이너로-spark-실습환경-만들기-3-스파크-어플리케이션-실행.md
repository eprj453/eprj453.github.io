---
title: "[spark] Docker Ubuntu 컨테이너로 Spark 실습환경 만들기 3. spark application 실행"
date: 2021-05-14 00:00:00
categories:
- spark
tags: [spark, docker]
---

worker와 master도 연결했으니 스파크 어플리케이션을 실행해보겠습니다.

<br/>

# spark shell

다음과 같이 실행합니다.

```shell
${SPARK_HOME}/bin/spark-shell --master {MASTER_URL}
```

마스터 url은 master web ui에서 확인할 수 있습니다. 포트를 따로 설정하지 않을 경우 7077 포트로 바운딩됩니다.

마스터 url이 잘못되거나 정상적인 환경이 아니더라도 shell이 오류없이 실행될 수 있습니다. web ui에서 `Running Applications`를 확인하는 것이 가장 정확합니다.

![image](https://user-images.githubusercontent.com/52685258/118147538-a2646480-b44a-11eb-8aba-1b586bc5d874.png)

어플리케이션이 잘 실행되고 있다면 shell에 코드를 입력해보겠습니다. 1장에서 확인했지만, hdfs에 저장되어 있는 파일을 사용하기 위해 namenode와 datanode가 잘 떠있는지 설정했던 hadoop web ui로 접속해서 확인해주시기 바랍니다.

```scala
scala> val inputPath = "hdfs://localhost:9000/sample/README.md"
inputPath: String = hdfs://localhost:9000/sample/README.md

scala> val outputPath = "hdfs://localhost:9000/sample/output"
outputPath: String = hdfs://localhost:9000/sample/output...

scala> sc.textFile(inputPath) flatMap{ line => line.split(" ") map (word => (word, 1L)) } reduceByKey(_ + _) saveAsTextFile (outputPath)
scala> sc.textFile(inputPath) flatMap{ line => line.split(" ") map (word => (word, 1L)) } reduceByKey(_ + _) saveAsTextFile (outputPath)
2021-05-14 01:07:23,757 INFO namenode.FSEditLog: Number of transactions: 13 Total time for transactions(ms): 38 Number of transactions batched in Syncs: 17 Number of syncs: 7 SyncTimes(ms): 100
2021-05-14 01:07:28,982 INFO hdfs.StateChange: BLOCK* allocate blk_1073741826_1002, replicas=127.0.0.1:9866 for /sample/output/_temporary/0/_temporary/attempt_202105140107231690350717401463493_0004_m_000001_0/part-00001
2021-05-14 01:07:28,983 INFO hdfs.StateChange: BLOCK* allocate blk_1073741827_1003, replicas=127.0.0.1:9866 for /sample/output/_temporary/0/_temporary/attempt_202105140107231690350717401463493_0004_m_000000_0/part-00000
2021-05-14 01:07:29,066 INFO datanode.DataNode: Receiving BP-172521654-172.17.0.2-1620407592592:blk_1073741826_1002 src: /127.0.0.1:37840 dest: /127.0.0.1:9866
...
...
...
```



spark job이 실행되는 듯한 메세지가 뜨고 맨 마지막에 completeFile들이 쫘라락 잘 떨어진다면 잘 실행된 것입니다. shell에서 나간 뒤 hdfs에 파일이 제대로 생성되었는지 확인해보겠습니다.

```shell
:quit

root@master# hdfs dfs -cat hdfs://localhost:9000/sample/output/p* | more
(package,1)
(this,1)
(integration,1)
...
...
...
```



<br/>

# spark-submit

코드를 입력하고 바로 결과를 확인할 수 있다는 점에서 테스트용으로는 shell은 장점을 가지고 있습니다. 그러나 보통 shell로는 처리하기 부담스러운 규모의 프로그램이 대부분입니다. 

spark-submit스파크에서 제공해주는 실행 스크립트입니다. 이를 이용해 여러 라이브러리를 참조하는 소스코드를 직접 실행시키거나, 배포 파일을 만든 뒤 스파크 클러스터에 배포해 사용할 수 있습니다. 개발 언어나 클러스터 매니저에 종속적이지 않기 때문에 일관된 방법으로 어플리케이션을 실행할 수 있습니다.

java는 스크립트 단위로 코드를 실행시킬 수 없기 때문에 패키지 전체를 jar 파일로 만들어 실행시킵니다.

```shell
./${SPARK_HOME}/bin/spark-submit --class com.sample.spark.xxxx \
																--master spark://{MASTER_HOST}:{PORT} \
																{JAR 파일 경로} \
																hdfs://{NAMENODE 경로}/sample/README.md \
																hdfs://{NAMENODE 경로}/sample/output/
```



python은 스크립트 단위로 코드를 실행할 수 있기 때문에 배포 파일을 만들지 않고 어플리케이션을 실행할 수 있습니다. 코드는 (여기)[https://github.com/wikibook/spark/blob/master/Python/ch3/wordcount.py]. 아까 shell에서 실행했던 스칼라 코드와 100% 동일한 동작을 하는 코드입니다.

실행은 클래스, jar 파일 경로 없이 특정 라이브러리 내부의 py 파일을 지정해주면 됩니다. py 파일은 -v로 로컬 폴더와 마운트되어있다면 로컬에서 바로 작성하면 되고, 마운트가 되어있지 않다면 도커 컨테이너 내부에서 직접 작성하면 됩니다.

```shell
/spark-submit --master spark://{MASTER_HOST}:{PORT} \
> /spark2nd/Python/ch3/wordcount.py \
> hdfs://localhost:9000/sample/README.md \
> hdfs://localhost:9000/sample/output_submit /
```



py 파일을 제출하면 무시무시한 실행 로그들이 쫘라락 뜨는 것을 볼 수 있습니다. 이 역시 마지막 부분에 

```shell
2021-05-14 01:39:09,982 INFO hdfs.StateChange: DIR* completeFile: /sample/output_submit/_temporary/0/_temporary/attempt_202105140139022178383439754362730_0008_m_000001_0/part-00001 is closed by DFSClient_attempt_20210514013901_0000_m_000000_0_-1297471761_33
```

이와 같은 compleFile이 잘 떨어진다면 잘 작동한 것입니다. spark submit은 shell과 달리 어플리케이션 작동이 끝나면 자동으로 종료됩니다. 

docker 환경에서 master / worker 컨테이너를 연결하고 spark shell, spark submit까지 실행시켜봤습니다.

<br/>

# 결론

사실 이렇게 도커 이미지를 직접 만들 필요는 없습니다. 로컬 모드로도 테스트는 충분히 가능하고, 도커 이미지를 쓰고 싶다면 구글에 docker hadoop spark으로 검색해보면 이미 많은 분들이 이미지를 만들어놓은 것이 있으니 그걸 쓰는게 더 현명한 방법입니다.

그것보다 더 좋은 방법은 일정 금액을 지불하고 AWS EMR같은 클라우드 클러스터 서비스를 이용하는 것입니다. 금액이 좀 들긴 하지만, 이런 수고로움 없이 양질의 클러스터 환경을 경험할 수 있습니다.

그러나 저처럼 굳이 직접 해봐야 직성이 플리는 분들이 있을거라 생각합니다. 그런 분들에게 이 포스팅이 조금이나마 도움이 되길 바랍니다.

