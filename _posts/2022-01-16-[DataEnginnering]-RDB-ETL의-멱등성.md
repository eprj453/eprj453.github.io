---
title: "[Data Enginnering] ETL의 데이터 소스가 RDB일 때, 멱등성은?"
date: 2022-01-16 00:00:00
categories:
- data
tags: [data-engineering, airflow]
---

대부분 파일로 저장하는 데이터들은 그 원본이 바뀌는 일은 많지 않습니다. 일자별로 파티션을 나누거나 적재된 시간을 기준으로 데이터를 ETL하면 원본이 변경되지 않는 한 동일한 결과를 도출할 것입니다.

그런데 RDB는 사정이 좀 다릅니다. 서비스에 사용되는 RDB는 사용자의 요청에 따라 수시로 Insert, Delete, Update가 이루어집니다. 흔히 생각하는 파일 시스템이나 RDD처럼 원본 데이터를 보존한다는 개념 자체가 성립하지 않습니다. 이 경우에는 언제 실행하느냐에 따라 ETL의 결과도 수시로 달라질 수 밖에 없습니다. 데이터 소스가 DB라면 12시에 돌아야 할 ETL Job이 실패해 1시에 재처리를 한다 해도, 12시에 실행한 것과는 그 결과가 다를 수 밖에 없는 것입니다.

이에 DB를 데이터 소스로 하는 ETL에서도 어떻게 하면 (최대한) 멱등성을 지킬 수 있을까?에 대해 생각해봤습니다.

<br/>

# 멱등성

멱등성(Idempotence)은 ***어떤 연산을 여러번 실행하더라도 그 결과가 동일한 성질***을 말합니다. 1+2는 몇 번 반복하더라도 그 결과가 3이고, 2022년 1월 15일 네이버에 로그인한 사용자 수를 계산하는 연산을 하루에 한 번씩 100년을 반복하더라도 그 결과는 동일해야 합니다. 이런 성질을 멱등성이라고 합니다.

연산의 횟수도 중요하지만, 시점에 조금 더 초점을 맞추고 싶습니다. ETL에서는 횟수만큼 시점도 중요하기 때문입니다. **여러 번 실행하더라도**보다 **언제 실행하더라도**에 좀 더 주목하는 것이겠죠? :)

<br/>

# ETL의 멱등성

연산이 대상으로 하는 날짜가 언제인가?를 기준으로 해, 해당 날짜의 Job은 오직 그 날짜만을 바라보도록 합니다. 

대부분의 배치 프로그램(ex, spring batch...)이나 ETL 스케줄러 프로그램은 크론탭으로 interval을 지정하거나, 파라미터로 원하는 날짜를 넘겨주는 식으로 배치성 ETL을 관리하지 않을까 생각합니다. 



그렇기 때문에, ETL이 도는 날짜를 다음과 같이 지정하는 것은 좋지 않습니다.

```python
from datetime import datetime

execution_date = datetime.now()
```

now() 메서드는 해당 메서드가 실행되는 그 순간이 언제인지를 반환합니다. 12시에 실행되면 12시, 1시에 실행되면 1시를 반환할 것입니다. 우리가 ETL의 멱등성을 지키기 위해서는 대상으로 하는 날짜는 다음과 같은 지조가 있어야 합니다.

> 이 ETL은 2022년 1월 16일 12시가 기준이야. 2022년 1월 16일 1시에 실행해도 2022년 2월 22일 2시에 실행해도 대상은 2022년 1월 16일 12시로 동일해!

airflow의 execution date에 대해서 이해하고 있다면 크게 어려운 개념은 아닐거라 생각합니다.



당연한 말이지만, ETL의 flow 자체가 멱등성을 잘 지킬 수 있도록 설계되어 있어야 합니다. 

<br/>

# DB의 멱등성



## 1. 주기적인 데이터 Dump

가장 쉽게 생각해볼 수 있습니다. 예를 들면 시간당 1번 정도 주기적으로 DB의 모든 데이터들을 어딘가에 저장해놓는 것입니다. 그건 파일일수도 있고, 같은 DB의 임시 테이블일수도 있습니다. 그럼 나중에 ETL을 다시 실행하더라도 괜찮습니다.

> 이 ETL은 2022년 1월 16일 12시가 기준이야. 그런데 실패해서 1시에 다시 실행시켰고, 이 때는 12시에 dump 떠놓은 데이터를 대상으로 할거기 때문에 12시에 수행한 ETL과 다르지 않아!



### 한계

첫 째는 리소스 낭비입니다. RDB에 저장하는 데이터는 보통 그 크기가 상당합니다. 이걸 snapshot 뜨는 것부터가 엄청난 시간과 리소스를 사용하는 일이고, 임시 데이터는 주기적으로 삭제도 해줘야 합니다. 여기에 들어가는 컴퓨팅 리소스와 금액은 ETL의 멱등성을 유지했을 때 드는 비용보다 훨씬 높을 확률이 큽니다.

두번째는 정확성의 한계입니다. dump를 뜨는 시점, dump가 완료된 시점이 언제냐에 따라 임시 저장 데이터의 결과도 달라질 수 밖에 없습니다. 또한 dump가 중간에 실패라도 하면 임시 데이터의 신뢰도는 낮아질 수 밖에 없습니다. 

<br/>



## 2. timestamp 컬럼

created_at, updated_at과 같은 시간대를 기록하는 컬럼을 두는 방법입니다. created_at과 updated_at이 같다면 그 동안 update 연산이 없었다는 것이기 때문에 여태까지 같은 데이터를 유지하고 있었다고 볼 수 있습니다. 



### 한계

변경된 데이터를 추적해 해당 시점에는 어땠는지에 대한 추적은 불가능합니다. update가 될 때마다 update되기 전의 데이터를 어딘가에 기록한다거나 해야 하는데 이는 1번 방법에 필적하는 비효율성을 가질 것 같습니다. 또한 만약 update가 아니라 delete라면 추적 자체가 불가능합니다.



<br/>

## 3. time travel

Google Bigquery에는 [해당 기능](https://cloud.google.com/bigquery/docs/time-travel)이 구현되어 있습니다. 최대 7일까지 변경 및 삭제된 데이터들을 쿼리하거나 복원할 수 있습니다. 이 뿐 아니라 유료 DW 서비스 (AWS, snowflake...)에서도 일정 기간 동안은 과거 테이터를 조회하거나 복구하는 기능을 제공하고 있습니다.

오픈 소스 중에서는 DeltaLake라는 툴이 time travel 기능을 제공한다고는 하는데, 흔히 생각하는 RDB는 아닌거 같습니다. (사실 Deltalake가 무엇인지 잘 모릅니다..)

https://databricks.com/blog/2019/02/04/introducing-delta-time-travel-for-large-scale-data-lakes.html



오라클에서는 Flaskback이라는 기능이 있어, AS OF 절로 특정 시점의 데이터를 확인할 수 있습니다. MariaDB에도 Flashback 기능이 있어 특정 시점의 데이터를 조회하는 것이 가능합니다. 단 select / insert / delete와 같은 DML만 조회가 가능하고 drop 같은 DDL의 이력까지는 지원하지 않습니다.

https://docs.oracle.com/cd/E11882_01/appdev.112/e41502/adfns_flashback.htm#ADFNS1008

https://mariadb.com/kb/en/flashback/



PostgreSQL에서는 6.2 버전까지는 Time Travel 기능을 제공했으나 그 윗 버전부터는 제공하지 않습니다. 모두가 예상할 수 있듯 퍼포먼스 저하와 저장공간 부족 등의 이유로 중지되었습니다.

> As of Postgres v6.2, *time travel is no longer supported*. There are several reasons for this: performance impact, storage size, and a pg_time file which grows toward infinite size in a short period of time.



Mysql은 비슷한 기능을 제공한 적이 없습니다. 오픈소스로 개발하려는 시도는 있었던 것 같으나 진전이 없거나 폐기된 프로젝트로 보입니다.

[https://mysql-time-machine.github.io/](https://mysql-time-machine.github.io/)



### 한계

유료 솔루션에서만 제공하는 것을 보면 알 수 있듯이, 일정 기간동안의 모든 기록을 가지고 있어야 하기 때문에 비용이 많이 들어갑니다.

<br/>

## 4. bin log 스캔

대부분 DBMS나 유료 DW 툴에서 제공하는 기능들도 결국은 일정 기간동안의 bin log를 전부 가지고 있기 때문에 가능합니다. 해당 날짜의 DB 데이터가 정말 필요한 경우라면 mysqlbinlog와 같은 툴로 바이너리 파일을 읽으면 DB가 떠있는 동안 일어났던 모든 트랜잭션이 기록으로 남아있는 것을 확인할 수 있습니다.

```sql
show binlog events;

mysql-bin.000001	111241	Xid	2132841112	111272	COMMIT /* xid=3965 */
mysql-bin.000001	111272	Gtid	2132841112	111337	SET @@SESSION.GTID_NEXT= '0444f713-6fd7-11ec-b33f-42010ab20002:392'
mysql-bin.000001	111337	Query	2132841112	111409	BEGIN
mysql-bin.000001	111409	Table_map	2132841112	111464	table_id: 109 (mysql.heartbeat)
mysql-bin.000001	111464	Update_rows	2132841112	111526	table_id: 109 flags: STMT_END_F
mysql-bin.000001	111526	Xid	2132841112	111557	COMMIT /* xid=3981 */
mysql-bin.000001	111557	Gtid	2132841112	111622	SET @@SESSION.GTID_NEXT= '0444f713-6fd7-11ec-b33f-42010ab20002:393'
mysql-bin.000001	111622	Query	2132841112	111694	BEGIN
mysql-bin.000001	111694	Table_map	2132841112	111749	table_id: 109 (mysql.heartbeat)
mysql-bin.000001	111749	Update_rows	2132841112	111811	table_id: 109 flags: STMT_END_F
mysql-bin.000001	111811	Xid	2132841112	111842	COMMIT /* xid=3993 */
mysql-bin.000001	111842	Gtid	2132841112	111907	SET @@SESSION.GTID_NEXT= '0444f713-6fd7-11ec-b33f-42010ab20002:394'
mysql-bin.000001	111907	Query	2132841112	111979	BEGIN
...
...
...
...
...
```



mysqlbinlog를 사용해 특정 시점의 DB를 백업하는 방법에 대한 포스팅을 공유합니다. 

https://myinfrabox.tistory.com/33



### 한계

현실적으로 모든 bin log를 뒤진다는것 자체가 너무 어렵습니다... bin log 또한 그 양이 엄청나기 때문에 필요할 때마다 해당 시점의 DB를 백업해서 사용한다는 것 자체가.. 타산이 맞는 발상인지 의심이 듭니다..

<br/>

# 결론



## 1. RDB의 본질은 무엇인가?

빅데이터라는 용어가 대중적으로 쓰이게 된 것도 어림잡아 10년은 넘은 것 같습니다. 그럼에도 RDB는 시장에서 그 입지가 확실하고 무결성, 보안, 안정성,  빠른 데이터 제공 속도 등의 장점을 가지고 여전히 서비스 용도의 DB로는 RDB가 쓰이고 있습니다.

서비스에서 사용자의 상태는 수시로 바뀌고 그 기록 또한 매우 방대합니다. 이런 RDB에서 파일 시스템과 같은 단단한 멱등성을 기대한다는 것이 RDB의 목적과 맞지 않겠다는 생각이 들었습니다. 물론 금전적 여유가 있다면 서비스에서 제공하는 flaskback 기능을 사용하면 되겠지만 그렇다면 이런 고민도 할 필요가 없겠죠...



## 2. 주기적인 백업

그럼에도 RDB 데이터는 중요합니다. 그렇기 때문에 주기적인 백업(dump)을 통해 최대한 ETL의 오차를 줄이는 것이 중요하다는 생각이 듭니다. 물론 데이터의 양이 어마어마하기 때문에 주기적인 삭제와 같은 데이터 관리 또한 중요합니다. `하루에 한 번 AWS RDS에서 S3로 스냅샷을 뜨고, 최대 1주일치만 보관한다` 와 같은 정책이 예시가 되겠습니다.

일주일 정도 전에 이런 생각이 들어서 여러 자료를 찾아본 것 같은데, 결론은 **`데이터 소스가 RDB라면 생각했던만큼 정확한 멱등성은 지키기 어렵겠다... `**입니다. 서두에 언급했던것처럼 RDB는 원본 데이터를 그대로 유지한다는 개념 자체가 없기 때문에, 남아있는 기록들을 역추척해 해당 시점을 복원하는 방식이 아니라면 현실적으로 이전 시점의 데이터를 정확히 바라보는 것이 어렵기 때문입니다.

