---
title: "[AWS] Lake Formation Error Case"
date: 2020-12-23 00:00:00
cateroeis:
- AWS
tags: [AWS, data-engineering]
---

blueprint로 workflow를 만들고 실행하면서 만나봤던 에러 케이스들과 해결했던 방안에 대해 적어봅니다.

<br/><br/>

# Resource Setup Error: Exception in thread "main" com.amazon.ws.emr.hadoop.fs.shaded.com.amazonaws.SdkClientException: Unable to execute HTTP request: Connect to xxxxxxx failed: connect timed out

VPC에 Gateway endpoint가 없어서 발생했습니다. VPC에 Public subnet, Private subnet 이외에도 Gateway endpoint를 생성해보시기 바랍니다.

<br/>

# AccessDeniedException: An error occurred (AccessDeniedException) when calling the CreateTable operation: Insufficient Lake Formation permission(s) xxxxxxx

1. https://docs.aws.amazon.com/lake-formation/latest/dg/lf-permissions-reference.html 공식문서에서 **DATA_LOCATION_ACCESS, CREATE_TABLE** 권한을 IAM에 줍니다.

2. Lake Formation console -> **Data lake locations, Data locations**에 blueprint에서 사용할 role, user가 s3 경로로 잘 등록되어 있는지 확인합니다. 혹 그래도 해결되지 않는다면 Lake Formation console -> Admin and database creators, Data permission에도 IAM User에게 모든 권한을 부여해보시기 바랍니다.

<br/>

# Could not find table xxxx

Blueprint 생성 시 Import source에서 경로를 잘 지정했는지 확인합니다. data catalog database가 아닌 JDBC / RDS / mongoDB 등 data source의 경로를 입력해야 합니다.

<br/>

# Error crawling database xxxx: SQLException: SQLState: 42000 Error Code: 1049 Message: Unknown database

위와 동일합니다.

