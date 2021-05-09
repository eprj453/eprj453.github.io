---
title: "[AWS] Lake Formation Blueprint 설정"
date: 2020-12-23 00:00:00
cateroeis:
- AWS
tags: [AWS, data-engineering]
---

# Lake Formation

Lake Formation은 Glue를 기반으로 하는 Trigger, Crawler, Job의 집합인 Worflow를 손쉽게 만들고, 관리할 수 있게 해주는 AWS 서비스입니다. WorkFlow를 직접 관리할수도 있지만, 여러 Job을 오가며 생길 수 있는 보안 및 권한 문제를 한번에 관리할 수 있고 WorkFlow 전체를 한 번의 Blueprint 사용으로 만들 수 있다는 점에서 꽤 매력적인 서비스입니다.

Blueprint 사용으로 WorkFlow를 만들어보고 실행시켜 RDS ==> S3(parquet) 저장까지 간단한 ETL을 진행해보고, 만났던 Error 케이스들을 살펴보겠습니다.

**개인 사용기이기 때문에 환경이 다른 부분이 많고 모든 에러가 같은 원인에 의해 발생하는 것은 아닙니다. 개개인의 상황과 다르거나 내용이 틀린 부분이 많기 때문에 참고사항으로만 봐주세요.**

<br/><br/>

## 사전 세팅

위에서 언급했던 Lake Formation의 장점 중 하나가 보안 / 권한입니다. WorkFlow 안에서 간편하다는 말은 반대로 말하면 그만큼 사전에 설정할 사항이 많고 꼼꼼해야 한다는 것입니다.

![image](https://user-images.githubusercontent.com/52685258/102898614-1eb2d700-44ad-11eb-9ba9-e96c67636743.png)

<br/>

### Data lake locations

만들고자 하는 data lake의 저장 공간(S3)을 입력합니다. S3 Path와 Role을 입력합니다. Role의 경우 기본적으로      **AWSServiceRoleForLakeFormationDataAccess**이 선택되어 있으나 blueprint에서 사용하고자 하는 Custom Role을 등록하는 것을 권장합니다. EMR을 사용하는 경우 위와 같은 Service-linked Role을 사용할 수 없기도 하고, Custom Role로 한꺼번에 필요한 Policy를 가져다 쓰는게 더 편하기 때문입니다.

![image](https://user-images.githubusercontent.com/52685258/102900098-312e1000-44af-11eb-8440-9a16eb816a80.png)

<br/>

### Data permissions

database에 대한 권한을 설정합니다. RDS가 아닌 ETL Job에서 사용될 data catalog -> database에 대해 등록하는 것입니다.

User와 Role을 등록할 수 있고, User는 기본적으로 Super를 포함한 모든 권한이 등록되어 있습니다. 등록되어 있지 않을 경우 등록하면 됩니다.

Role은 직접 등록해야 하는데, 필수적으로는

- **Create Table**
- **Describe**

권한을 포함해주시고, 나머지는 선택사항입니다.

![image](https://user-images.githubusercontent.com/52685258/102899949-02179e80-44af-11eb-996d-34e58f276521.png)

<br/>

### Data locations

Data lake locations에서 등록한 S3에 대한 User / Role 권한을 지정합니다. 여기서 권한을 지정함으로써 workflow가 data Destination에 데이터를 쓸 수 있게 됩니다.

공식문서 Tutorial에서는 WorkFlow에서도 **LakeFormationWorkflowRole**을 사용하라고 적혀있으나 저는 Custom Role을 사용하기 때문에 Data locations에도 Custom Role과 IAM User를 등록합니다.

![image](https://user-images.githubusercontent.com/52685258/102900731-10b28580-44b0-11eb-93c2-0c96f21819da.png)



이제부터는 Lake Formation Console에서 설정하는 사항이 아닙니다.

<br/>

### Glue Connection

Lake Formation 안에 있는 사항은 아니지만 Target으로 하는 RDS에 Glue Connection을 필수로 설정해놓아야 합니다. 이를 설명하기 위해서는 VPC에 대한 전반적인 사항도 함께 알아야하지만 이에 관해서는 간단하게 언급만 하고 넘어가겠습니다.

- RDS와 Subnet이 같은 VPC를 공유하도록 합니다.
- VPC는 S3 Endpoint(Gateway Endpoint)를 갖고 있어야 하고, 그럼에도 Glue Connection이 제대로 이루어지지 않을 경우 Public Subnet과 Private Subnet을 생성한 뒤 Private Subnet에는 NAT Gateway를 연결합니다.

<br/>

### IAM Role

Trust relationship

- lakeformation.amazonaws.com
- glue.amazonaws.com
- export.rds.amazonaws.com

AWS Maneged Policy

- AWSLakeFormationDataAdmin
- AdministratorAccess

Custom Policy 참고사항

- [Underlying Data Access Control](https://docs.aws.amazon.com/lake-formation/latest/dg/access-control-underlying-data.html#underlying-data-access-control)

- [Creating a Database](https://docs.aws.amazon.com/lake-formation/latest/dg/creating-database.html)

<br/>

그 외에도 [Security and Access Control to Metadata and Data in Lake Formation](https://docs.aws.amazon.com/lake-formation/latest/dg/security-data-access.html), Data Permission의 경우에는 [Upgrading AWS Glue Data Permissions to the AWS Lake Formation Model](https://docs.aws.amazon.com/lake-formation/latest/dg/upgrade-glue-lake-formation.html)을 참고했습니다.

<br/><br/>

## Blueprint로 workflow 만들기

Lake Formation console -> Register and ingest -> Blueprints -> Use a blueprint

<br/>

### Blueprint type

workflow에서 data load를 어떤 식으로 할 것인지 지정합니다. 저는 새로 생성되는 데이터만 load하기 위해 Incremental database를 선택합니다. 

> Incremental을 직역하면 증분이라는 말로 사용하던데, 개인적으로 증분이라는 말이 잘 와닿지는 않습니다.

<br/>

### Import source

- Database connection : workflow에서 사용할 Glue Connection을 선택합니다. 이 Glue connection에 VPC를 포함한 RDS와의 연결정보가 모두 담겨있습니다.
- Source data path : RDS 기준으로 어떤 데이터베이스 or 스키마 / 테이블을 대상으로 할 것인지 지정합니다. 하위 모든 사항을 선택하는 와일드카드도 %를 이용하면 가능합니다.
  - schema1에 있는 모든 Table : **schema/%**
  - schema2에 있는 TB_1: **schema2/TB_2**

<br/>

### **Incremental data**

blueprint type에서 Incremental database를 선택했을 경우에만 선택하는 메뉴입니다. 새로운 데이터를 판단할 기준이 될 column을 지정합니다. 예를 들어, id 10번까지의 data를 이전에 load 했다면 다음 workflow 실행에서는 11번부터 data만 load합니다.

Timestamp이나, autoIncrement가 걸려있는 것을 bookmark key로 지정하는 것이 바람직하겠습니다. 

<br/>

### Import target

- Target database : Data catalog에 database를 선택합니다.
- Target storage lcoation : data lake Storage(S3)의 경로를 입력합니다. S3라면 모두 입력 가능하지만, data lake location에 등록된 S3를 입력하는 것을 권장합니다.
- Data format : 원하는 data format을 선택합니다(parquet or csv)

<br/>

### **Import frequency**

workflow 스케줄을 선택합니다.

<br/>

### Import options

- workflow name : workflow 이름
- Maximum capacity : 최대 DPU 갯수
- Concurrency : Job 병렬처리, 최대 동시 실행 수를 설정합니다.

<br/><br/>

다음 사항에서는 직접 workflow를 작동시켜 결과를 확인해보고, 에러를 공유한다.



