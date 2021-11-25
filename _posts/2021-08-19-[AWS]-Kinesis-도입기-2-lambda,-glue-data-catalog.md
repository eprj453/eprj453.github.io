---
title: "[AWS] Kinesis 도입기 2. Lambda, Glue"
date: 2021-08-19 00:00:00
categories:
- AWS
tags: [AWS]

---



firehose는 data stream의 consumer로 등록 가능한 aws 서비스입니다. broker인 data stream에 있는 데이터를 특정 목적지(s3, redshift, http...)로 보내주는 역할을 하는데, 중간에 끼워넣을 수 있는 transform 단계로 lambda와 glue catalog table이 있습니다. 

<br/>

# Lambda

Lambda는 AWS에서 제공하는 코드 실행 서비스입니다. 코드를 lambda에 업로드하면 스케줄이나 트리거에 의해 발동해 코드를 실행합니다. 

Firehose에 들어온 데이터(bytecode)를 인자로 받아 lambda function를 실행할 수 있습니다. pandas를 이용해 dataframe 연산을 실행하거나 특정 데이터를 삭제하는 등의 연산이 가능합니다.

Lambda가 고정적으로 받는 인자인 event 내부에 딕셔너리 형태로 데이터가 저장되어 있습니다. firehose의 옵션에서 lambda를 지정할 수 있기 때문에 사용은 어렵지 않습니다. 



그러나 firehose에서 lambda를 실행할 때는 quota에 주의해야 합니다. 동시 실행 가능한 lambda의 갯수는 shard 당 5개입니다. 즉 4개의 shard를 갖는 data stream을 firehose가 polling할 때, 동시에 실행 가능한 lambda 갯수는 20개가 됩니다.

여기서 lambda가 연산을 마치고 내려가는 시간보다 data stream의 buffer가 차는 시간이 더 빨라지게 되면, lambda는 에러를 발생시킨 채 그대로 내려가게 됩니다. 이로 인해 유실되거나 제대로 변환되지 않은 데이터에 대해서는 kinesis에서 따로 처리할수는 없습니다. error data path를 따로 지정해 실패한 데이터에 대한 후처리가 필요합니다. 

firehose에 lambda를 사용할 때는 buffer를 길게 두어 실행시간에 여유를 두거나 shard를 늘려 처리량을 분산시키는 것이 바람직합니다.

<br/>

# Glue data catalog

aws에서 사용하는 데이터에 대한 메타데이터를 담고 있는 테이블입니다. 컬럼명, 데이터 타입, 원본 데이터 포맷, 저장소 위치 등을 담고 있습니다.

aws에서 hive 연산이나 presto 연산 등 메타 테이블이 필요한 경우 주로 glue data catalog에 생성된 메타 테이블을 참조합니다. 서버리스 쿼리 서비스인 athena는 presto 엔진을 사용하고 있기 때문에 사용하고 있고, EMR에서 hive 연산을 하더라도 glue data catalog를 참조하도록 하고 있습니다(물론 따로 hive metastore를 만들어서 참조해도 무방하고, EMR 이전 버전에서는 glue data catalog를 지원하지 않았던 적도 있습니다).



kinesis에서 주요 역할은 2가지입니다.

- data catalog에 정의되어 있는 컬럼명, 데이터 타입과 firehose로 들어온 데이터 간의 정합성
- 최종 목적지로 보내고자 하는 파일의 포맷 지정

메타 스토어의 정보로 위 2가지를 지정한 뒤 최종 목적지로 보내게 됩니다.

<br/>

# 결론

결론적으로 firehose에서 lambda는 사용하지 않았습니다. shard 갯수에 따른 실패 확률도 크고, lambda 때문에 필요 이상으로 shard를 늘리는건 배보다 배꼽이 더 크다고 생각했습니다.

Glue data catalog는 1차로 데이터 정합성을 걸러내는 역할을 하고 있습니다. 단, 특정 컬럼을 삭제하는 작업이 필요해 firehose가 s3로 떨어트리는 파일을 트리거로 lambda를 발동시켜 최종 가공되는 데이터를 따로 적재해 사용하는 방식을 채택했습니다.

저는 kinesis를 먼저 접하고 kafka의 구조를 나중에 공부하기 시작한 경우입니다. 쓰고 보니 형태가 상당히 유사합니다. 개발자가 직접 개발해 배포하지 않고 서비스만으로 kafka를 쓸 수 있게 만든게 kinesis라는 생각이 들 정도입니다. 

이렇게 AWS는 개발자가 프레임워크나 시스템을 몰라도 서비스가 가능하도록 많은 서비스를 제공합니다. 만약 이런 실시간 로그 수집 아키텍처를 제가 kafka 공부를 시작해 개발하고 배포했다면 엄청나게 오랜 시간이 걸렸을겁니다.

그런데 쓰다보니 AWS에서 제공해주는 기능 외에는 할 수 없는 사람이 되는듯한 느낌입니다. 실제로도 그렇습니다. kafka의 broker와 유사한 역할을 하는 data stream의 shard는 1초에 1mb 이상의 데이터는 받을 수 없습니다. AWS에서 지정한 그 이상을 쓸 수 없습니다. kafka에서도 한 번에 엄청난 크기의 데이터를 partition에 밀어넣지는 않겠지만, 하지 않는 것과 할 수 없는 것은 차이가 큽니다. 

프레임워크, GC, AWS 등등 프로그래밍은 점점 개발자가 내부를 몰라도 되도록 개발을 하기 편한 구조로 진화합니다. 그렇지만 그렇게 탄생한 편리함의 산물을 잘 쓰기 위해서는 그 원리와 구조를 알아야겠다는 생각이 듭니다.