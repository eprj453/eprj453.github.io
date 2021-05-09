---
title: "[AWS] Glue Use Case"
Date: 2020-12-13 00:00:00
categories:
- AWS
tags: [AWS, data-engineering]
---

다른 AWS 서비스와 Glue를 함께 사용해보자.



# lambda

https://aws.amazon.com/ko/lambda/

AWS Lambda는 서버 배포 필요없이 코드를 실행시킬 수 있는 서비스다. lambda에 boto3(AWS API)로 코드를 작성하면 이미 생성되어 있는 Crawler를 대상으로 해 Glue Job을 실행시킬 수 있다.

```python
import boto3

client = boto3.client('glue')

glue_job_name = '{glue job 이름}'

def lambda_handler(event, context):
    response = client = start_job_run(JobName=glue_job_name)
```

https://aws.amazon.com/ko/premiumsupport/knowledge-center/start-glue-job-crawler-completes-lambda/

자세한 예제는 document를 참고하자.

<br/>

# Athena

https://docs.aws.amazon.com/ko_kr/athena/index.html

athena는 여러 저장소의 데이터를 SQL로 읽을 수 있는 서비스이다. 

![image](https://user-images.githubusercontent.com/52685258/102012816-ce8e9300-3d8f-11eb-947f-a6b103d3ecbf.png)

Data Catalog에 table이 생성되어 있으면 빠른 속도로 조회가 가능하다.

이전에도 언급했지만 서로 다른 스키마 형식의 파일의 테이블을 같은 Crawler에서 만들게 되면 Athena에서 데이터를 읽어올 수 없으니 주의하자.

https://docs.aws.amazon.com/ko_kr/athena/latest/ug/glue-best-practices.html#schema-classifier

이 곳에 사례가 좀 더 자세하게 나와있으니 참고하자.

<br/>

이것 말고도 EMR, Redshift 등 다른 AWS 서비스와 Glue도 연동하여 쓸 수 있으니 상황에 맞게 잘 사용하면 되겠다.

