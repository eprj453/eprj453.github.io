---
title: "[AWS] Glue Crawler"
Date: 2020-12-09 00:00:00
caterogies:
- AWS
tags: [AWS, data-engineering]
---

Glue의 Crawler를 만들고 실행한 뒤, 메타 데이터 테이블이 만들어지는지 확인하고 여러 특성을 살펴본다.

<br/>

# Data Source

크롤러가 대상으로 하고 메타 테이블을 생성할 Data Source가 필요하다. RDS, S3, DynamoDB 등 AWS의 데이터 저장소 뿐만 아니라 JDBC를 지원하는 DB, file 등등 거의 모든 형태의 데이터 저장소가 가능하다.

S3에 csv 파일 2개를 업로드 한 후 실습한다.

![image](https://user-images.githubusercontent.com/52685258/101626080-c10b9d00-3a5f-11eb-8d98-1df0399adeb6.png)

id_name : id(int), name(varchar)

id_score : id(int), id(score)

그런데 여기서 주의할 점은, **스키마가 서로 다른 파일은 각각 다른 폴더에 들어있어야 한다는 점**이다. 메타 데이터 테이블은 Glue 뿐만 아니라 다른 AWS 서비스에서도 사용되는데, **만약 다른 스키마의 파일이 같은 폴더에 있다면 이 때 문제가 된다**. Zero record returned 때문에 한참을 헤맸다.

위 업로드 결과에도 폴더 단위로 파일을 구분해놓은 것을 볼 수 있다.

<br/><br/>

# Crawler 생성

<br/>

## 1. Crawler info

![image](https://user-images.githubusercontent.com/52685258/101626707-b6053c80-3a60-11eb-9d30-b153959720f5.png)

간단하게는 이름만 설정하고 넘어가도 된다. option 설정 사항도 펼쳐봤을 때 특별한건 없지만, Classifier는 때에 따라서 꼭 필요할 수 있다. 

csv 파일일 때 특히 그런데, 일반적으로 쉼표를 구분자로 사용하긴 하지만 구분자가 통일되지 않은 경우가 있을 수 있기 때문에 이럴 때에는 Custom classifier를 이용해 구분자를 통일하는 것이 좋겠다.

<br/>

## 2. Crawler source type 

![image](https://user-images.githubusercontent.com/52685258/101627269-72f79900-3a61-11eb-9535-be5f7fc537db.png)

**Crawler source type**

Data store(데이터 저장소)를 바라볼지, 기존에 생성한 Data Catalog의 메타 데이터 테이블을 바라볼 지 선택한다.

**Repeat crawls of S3 data stores**

S3의 경우 적용이 가능하다. 만약 지정한 S3 버킷에 새로운 스키마를 가진 파일이 생겨있을 경우, 마지막으로 Crawler가 작동했던 시점을 기준으로 새롭게 생성된 Schema에만 Crawler가 돌게 할 수 있다.

<br/>

## 3. Data store

![image](https://user-images.githubusercontent.com/52685258/101627822-44c68900-3a62-11eb-9e12-68eeaa15bb7d.png)

**Choose a data store, Include path**

data store를 추가한다. 메타 테이블을 생성할 데이터 저장소를 1개 혹은 여러 개 추가할 수 있다(Next를 누르면 더 추가할 지 물어보는 옵션이 나온다).

data store의 종류에 따라 선택하는 option이 조금씩 달라진다. 그렇지만 connection이나 테이블 위치 등을 묻는 옵션들이기 때문에 어렵지 않게 선택할 수 있다.

<br/>

## 4. IAM Role

![image](https://user-images.githubusercontent.com/52685258/101628397-19906980-3a63-11eb-959c-9a948aa2469c.png)

IAM 계정의 경우, 여러 사람이 통합된 AWS 안에서 여러 서비스를 이용하기 때문에 그에 맞는 Role과 Policy가 필요하다(하나의 Role이 여러 개의 Policy를 포함하고 있다고 생각하면 된다).

현재는 Glue를 사용하고 있으니 이에 맞는 Role이 있다면 선택하고, 없다면 추가로 생성할수도 있다.

<br/>

## 5. Schedule

![image](https://user-images.githubusercontent.com/52685258/101628728-94f21b00-3a63-11eb-89eb-142f88e80cb9.png)

Crawler의 작동 스케줄을 설정할 수 있다. 시간마다, 일마다, 주마다 등 기본 옵션이 있고 직접 커스텀하거나 원할때만 작동하도록 고를 수도 있다. 보통 Run on demand를 설정한 뒤 필요한 경우에만 lambda 등으로 작동시키지 않을까 싶다.

<br/>

## 6. output

![image](https://user-images.githubusercontent.com/52685258/101628957-f1553a80-3a63-11eb-9e0b-8004f609ee7e.png)

메타 테이블(Cralwer 결과)을 저장할 곳을 지정한다. Glue 초기화면 -> Data catalog에서 DataBase를 만들 수 있다. 아니면 Add database로 바로 만들수도 있다.

output의 option은 유용하게 쓸 일이 있을것 같아 짚고 넘어가려고 한다.

<br/>

**Create a single schema for each S3 path**

이 옵션을 선택하면, 스키마 간의 유사성을 고려해 메타 테이블을 통합시켜준다. 

```
# table A
id1 : int
id2 : int

# table B
id3 : int
id4 : int
```

위와 같은 스키마를 가진 테이블 2개가 만들어진다고 할 때, 스키마의 유사성이 크다고 판단된다면 메타 테이블을 굳이 따로 만들지 않고 하나의 테이블을 만든 뒤, partition 등을 통해 구분하게 된다.

<br/>

**When the crawler detects schema changes in the data store, how should AWS Glue handle table updates in the data catalog?**

크롤러가 같은 테이블이지만 스키마 변경을 감지했을 때, 

- 메타 테이블 전체를 다시 정의할 지 
- 추가된 컬럼만 정의할 지
- 아무 것도 하지 않을지

**How should AWS Glue handle deleted objects in the data store?**

data store에서 file 등이 삭제되었을 때 glue에서는 어떻게 처리할지 고른다.

마지막으로 모든 사항을 확인한 뒤 크롤러를 만들 수 있다.

<br/>

<br/>

# Crawler 작동

crawler를 실행시켜보자. 파일의 용량 / 갯수에 따라 실행시간은 다르겠지만 생각보다 시간이 좀 걸리는 편이다.

어차피 kb 단위의 작은 파일로 실습을 하고 있어 크게 상관없지만, glue는 서비스 요금이 꽤 비싼 편에 속하기 때문에 혹시 모를 요금폭탄에 주의하자. 

![image](https://user-images.githubusercontent.com/52685258/101630093-be13ab00-3a65-11eb-9538-8c1d1de9286a.png)

<br/>

실행이 완료된 뒤 output에서 생성했던 Database를 보면, 이렇게 메타 데이터 테이블들이 만들어진 것을 볼 수 있다. 메타 데이터 테이블은 이렇게 생겼다.

![image](https://user-images.githubusercontent.com/52685258/101630390-25c9f600-3a66-11eb-9e65-b9caf79dfd84.png)

스키마 정보와 테이블의 기본적인 통계자료를 가지고 있다.

다음에는 glue job을 생성한 뒤 table을 기반으로 작동시켜 원하는대로 데이터를 조작할 수 있는지 확인해볼 것이다.