---
title: "[AWS] Kinesis 도입기 1. Data Stream과 Firehose"
date: 2021-08-03 00:00:00
categories:
- AWS
tags: [AWS]

---



최근 실시간 로그 수집을 위해 kinesis를 도입했습니다. 앞단에서는 k8s 환경으로 배포된 fastapi 어플리케이션이 로그를 받고, 여기서 여러 군데로 로그를 전송하는데 그 중 하나가 kinesis입니다. 안정적으로 kinesis를 가용하기 위해 공부하고 테스트했던 사항들을 기록합니다.



<br/>

# Data Stream

kinesis에서 데이터 수집을 할 수 있는 가장 간단한 방법입니다. Producer와 Consumer를 지정할 수 있는 메세지 큐와 비슷한 형태를 띄고 있는 듯 하지만, 실시간 데이터 처리가 가능한 데이터 저장소라고 보는 게 더 맞을 것 같습니다.

Data stream으로 로그를 수집하면 뒤에서 설명할 firehose 뿐만 아니라 Analytics와 같은 다른 kinesis 서비스, Kinesis 라이브러리로 사용자가 직접 개발한 어플리케이션에서 로그를 편리하게 당겨갈 수 있다는 장점이 있습니다.

데이터 저장소답게 data stream에 전송된 데이터는 retension period 기간만큼 저장됩니다. retension period는 기본 1일입니다. 추가로 요금을 지불하면 최대 7일까지 데이터를 보관할 수 있고 polling을 하면 언제든 가져갈 수 있습니다.

<br/>

# Shard

Data stream은 최소 1개, 최대 200개의 shard로 구성됩니다. shard의 갯수가 늘어날수록 소화할 수 있는 데이터의 양은 커지고, 그만큼 비용도 증가합니다. 

1 shard의 스펙은 다음과 같습니다.

- ingest : 1mb / 1000 record
- Consume : 2mb / 5TPS

shard의 비용은 시간당 0.015달러이기 입니다. 24시간에 0.36달러면 그리 비싼 금액은 아니지만 지역별로 요금 차이가 있고 사용 환경마다 최적의 shard 갯수는 다르기 때문에 AWS에서 제시하고 있는 quota를 확인하시기 바랍니다.

https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html



안타깝게도 shard는 자체 오토 스케일링을 지원하지 않습니다. 이를 위해서는 추가 API 개발이 필요합니다. 자세한 사항은 이 포스트에 잘 정리되어 있습니다.

https://medium.com/slalom-data-analytics/amazon-kinesis-data-streams-auto-scaling-the-number-of-shards-105dc967bed5

<br/>

# Producing

![image](https://user-images.githubusercontent.com/52685258/127911376-713b48cf-6296-4af7-98d4-1525f8ea99f9.png)

n개의 producer에서 log를 수신하면, 특정 hash function을 거쳐 고유 record id를 부여받습니다. 고유 record id를 부여받은 record들은 그림과 같이 shard에 차곡차곡 쌓이고, consumer는 이 shard를 기준으로 데이터를 가져갑니다. 데이터를 polling할 때 offset도 이 record id를 기준으로 합니다.

데이터를 push할때 hask key와 sequence number를 수동으로 지정하는 것도 가능하지만, 저는 이 기능을 사용하지 않았습니다. Sequence number와 hash key 모두 kinesis에서 자동으로 지정해주는 데에 큰 불편함을 느끼지 못했습니다.

shard의 데이터가 polling될때는 records, 즉 record의 array 형태입니다.

<br/>

# Consuming

Consuming은 크게 2가지 방법이 있습니다.

- get record CLI나 SDK를 사용해 직접 polling한다.

- consumer를 지정해 data stream에서 바로 데이터를 흘려보낸다.

  

직접 polling할때는 data stream 전체가 아닌 shard를 기준으로 합니다. 각 shard마다 들어있는 데이터가 다르기 때문에 어떤 shard에서 데이터를 polling하는지가 중요합니다. get_record 명령 또한 shardIterator를 필수 지정해야 합니다. 

![image](https://user-images.githubusercontent.com/52685258/127912652-26a66ec2-903a-409d-89bd-6250ac58a19e.png)

그렇다면 실시간으로 데이터를 polling할때 모든 shard에 poller가 붙어 안정적으로 데이터를 처리할 수 있느냐가 중요합니다. 모든 shard에 1:1로 붙지 않더라도 빠른 속도로 스위칭이 가능해야 특정 shard에 쏠림이 없이 안정적으로 데이터를 처리할 수 있습니다.

저는 이 polling을 위해 lambda function 등을 띄우는 것도 여의치 않아서 firehose를 data stream에 consumer로 두는 방법을 선택했습니다. 

뒤에서 설명이 되겠지만 firehose에도 바로 데이터를 push할 수 있습니다. 그럼에도 data stream -> firehose의 workflow를 만든 것은 추후 analytics 등이 붙을 확률이 높기도 하고, 현재 데이터를 수신하는 곳 외에도 데이터를 보내야 할 곳이 많아지면 1 stream에 n개의 consumer를 꽂는 구조를 만들어 놓는게 편하기 때문입니다.



Firehose 등으로 consuming을 할 때도 각 shard의 상황을 잘 고려해야 합니다. 제한된 shard 리소스를 초과하지 않도록 설계하는 것이 중요합니다. WriteThroughputExceeded나 Rate Exceeded 에러가 발생하는 경우 consumer에서 너무 많은 GetRecords 요청을 보내고 있는건 아닌지 의심해봐야 합니다.

> More than one Kinesis Data Firehose delivery stream can read from the same Kinesis stream. Other Kinesis applications (consumers) can also read from the same stream. Each call from any Kinesis Data Firehose delivery stream or other consumer application counts against the overall throttling limit for the shard. To avoid getting throttled, plan your applications carefully. For more information about Kinesis Data Streams limits, see [Amazon Kinesis Streams Limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html).

<br/>

# firehose

firehose는 실시간 전송 및 적재를 위한 서비스입니다. data stream이 produce와 consume을 담당했다면 firehose는 데이터를 전송하거나 데이터 변환 과정을 거치고자 할 때 사용합니다.

firehose에도 데이터를 바로 push할 수 있습니다. API도 제공되고 있고 그 자체만으로도 완전 관리형 서비스이기 때문에 굳이 data stream이 필요하지 않다면 firehose 단일 인스턴스만으로도 실시간 데이터 처리가 가능합니다.



# Data stream -> firehose

data stream의 consumer로 등록된 firehose는 1초에 1번 data stream에 있는 각 shard에 GetRecords를 요청합니다.

>  Kinesis Data Firehose starts reading data from the `LATEST` position of your Kinesis stream. For more information about Kinesis Data Streams positions, see [GetShardIterator](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetShardIterator.html). Kinesis Data Firehose calls the Kinesis Data Streams [GetRecords](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetRecords.html) operation once per second for each shard.



이렇게 초마다 각 shard에서 모이는 데이터들은 firehose에서 지정한 buffer가 다 찰 때까지 대기했다가, 지정한 경로로 전송됩니다. 지정해야 하는 buffer는 다음과 같습니다.

- 60초 ~ 360초
- 64mb ~ 128mb

buffer를 60초, 100mb로 지정했다면 100mb가 다 차지 않더라도 60초가 지나면 자동으로 지정 장소로 전송됩니다. 또한 60초가 지나지 않았더라도 100mb가 다 차면 지정 장소로 전송됩니다.



다음 포스팅에서는 firehose에서 data transform에 사용한 lambda, convert format에 사용했던 glue data catalog에 대해 알아보겠습니다.