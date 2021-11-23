---
title: "[AWS] MWAA 사용기"
date: 2021-09-27 00:00:00
categories:
- AWS
tags: [AWS]
---



8월 31일에 MWAA(Amazon Managed Workflows for Apache Airflow)가 서울 리전에도 출시되었습니다. 짝짝!!

https://aws.amazon.com/ko/about-aws/whats-new/2021/08/amazon-managed-workflows-apache-airflow-mwaa/

그동안 온프레미스 환경에서 airflow를 사용하며 이런저런 애로사항이 있었는데, 서울 리전에도 출시가 되어서 한번 사용해봤습니다. MWAA 자체가 나온지 1년도 채 되지 않았고 서울 리전 출시도 늦은 편이라 usecase도 많지 않고 저 또한 많은 부족함이 있으나 MWAA를 사용하시는 분들께 도움이 되면 좋겠습니다.



<br/>

# MWAA



## 장점

boto3와 같은 SDK, 다른 AWS 서비스(ex: lambda)로 리소스 관리를 할 수 있다는 점, 따로 webserver / scheduler 등 환경 설정 및 리소스 산정을 할 필요가 없다는 점이 MWAA의 장점이라 할 수 있습니다. 



## 단점

S3에 dag 소스파일과 custom 모듈, requirements.txt를 업로드해 사용하는 전형적인 AWS식 소스 관리 방법을 따르고 있습니다. GCP cloud composer는 Airflow를 프레임워크 그대로 개발해 배포까지 이어지는, 흔히 말하는 Drop-in replacement가 가능하도록 되어있는 것 같은데 S3에 파이썬 스크립트를 파일 단위로 업로드하며 관리하는 방식은 좀 아쉽네요. 소스 관리가 상당히 귀찮겠습니다.

버전도 2개밖에 없습니다. 1버전과 2버전에서 가장 안정적인 버전 1개씩만을 지원하는것 같네요.



## 결론

많이 쓰일것 같은 서비스는 아닐것 같습니다.

S3까지 이어지는 파이프라인은 codepipeline도 따로 구축해야 할텐데... GCP Cloud Composer는 이런 부분이 꽤 간편한걸로 알고 있는데 좀 아쉽습니다.

AWS에서 airflow를 쓰려면 아직은 직접 개발해서 EKS에다 배포해서 쓰는게 더 편할거 같습니다. 단, 환경 구축이나 리소스 산정을 빠르게 하고 싶다면 MWAA는 나쁘지 않은 대안이라 생각합니다.



<br/>

# 테스트

job을 하나 띄워보고 External module, requirements.txt까지 제대로 동작하는지 보겠습니다.

MWAA 첫 화면입니다. Create environment로 airflow를 띄울 수 있습니다. 

![image](https://user-images.githubusercontent.com/52685258/135724592-53831ce8-8184-44b1-9887-fef3f46c7e49.png)



기본 정보입니다. Airflow 버전과 dag 폴더, 플러그인 zip 파일, requirements.txt 파일이 있는 S3 경로를 지정합니다. S3 버킷은 `airflow-*` 형태의 이름을 가지고 있어야 하며, 경로에 파일이 없거나 규칙에 어긋나면 다음으로 넘어갈 수 없습니다.

2021년 10월 현재 선택할 수 있는 버전은  

- 1.10.12
- 2.0.2 

2가지입니다. 음... 처음부터 MWAA를 사용하는 분들에게는 문제가 되지 않을수도 있겠으나 저처럼 다른 환경에서 AWS 환경으로 airflow를 이관하려는 분들에게는 이 또한 치명적일 수 있겠습니다.

플러그인은 `$AIRFLOW_HOME/plugins`에 저장해 바로 import해서 사용할 수 있는 airflow에서 제공하는 기본 기능입니다. custom module을 여기에 등록해서 사용하게 되는데, 프레임워크 단위로 코드 관리를 하면 더 좋지 않을까? 하는 생각이 여기에서도 들긴 합니다. 공식 문서는 [여기](https://airflow.apache.org/docs/apache-airflow/stable/plugins.html) 

<br/>

![image](https://user-images.githubusercontent.com/52685258/135724688-74e9daec-d7e5-44ed-abab-b047f4830e59.png)

<br/>

다음으로는 네트워크, 리소스 등 상세 설정입니다. 

네트워크는 subnet 2개가 있는 VPC 하나를 필요로 합니다. UI를 띄우는 과정에서 webserver - environments를 연결하고 접근을 관리하기 위해 필요한 등록이라 보시면 되겠습니다. 그 외에도 airflow가 접근해야 할 리소스(redshift, s3, dynamodb...) 또한 VPC의 범위를 산정하는데 필요합니다.

MWAA 설정에서는 따로 언급하지 않는데, WEB UI에 기본적으로 EIP가 하나 붙습니다. 만약 등록할 수 있는 EIP 갯수가 최대치라면 Environments 설정을 끝까지 마치고 환경을 생성하더라도 Creating에서 몇시간 멈춰있다가 Fail이 떨어지는 사태가 벌어질 수 있습니다.

https://docs.aws.amazon.com/mwaa/latest/userguide/t-create-update-environment.html#t-stuck-failure

여기에서는 VPC에 문제가 있는지 확인해보라고 하지만, 정상적으로 2개의 subnet이 있는 VPC를 등록했는데도 Stuck in Creating을 만난다면 생성할 수 있는 EIP 갯수를 넘은게 아닌지 한번 체크해보시기 바랍니다. 

그 다음으로는 리소스를 어떻게 쓸 것인지, 로그는 어떻게 남길 것인지 등에 대한 사항들을 입력합니다. 이 부분은 사용자의 환경마다 상이하기 때문에 본인의 환경에 맞는 세팅을 해주시면 되겠습니다. Step 3는 최종 확인 절차이기 때문에 실질적으로는 이게 세팅의 전부입니다.

![image](https://user-images.githubusercontent.com/52685258/135725012-7e515c9f-aa14-4443-9448-fc2f67c9f306.png)

<br/>

완료하면 airflow environment가 생성됩니다.

![image](https://user-images.githubusercontent.com/52685258/142954339-8c7e0be4-3d40-42d7-9933-03f772e95dd7.png)

UI에도 바로 접속할 수 있습니다.

![image](https://user-images.githubusercontent.com/52685258/142954502-2944f537-d798-4bf1-b158-e704efc9910c.png)



폴더 구조는 아래 github 참고했습니다.

https://github.com/czam01/mwaa



