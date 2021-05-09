---
layout: post
title: "[AWS] glue에서 내 프로젝트 import하기"
date: 2021-04-02 00:00:00
categories:
- AWS
tags: [AWS]
---

glue job을 사용하다보니 불편함을 느낀 점은 2가지입니다.

- 테스트 환경이 너무 별로다.
- 스크립트만 실행이 가능하다.

로컬에 glue 라이브러리를 설치할 수 있지만 s3에서 갯수가 많은 파일 버킷 단위로 읽어서 spark 연산을 돌린다거나 하면 웬만한 로컬에서는 아쉬운 속도를 보입니다.

또한 glue job이 실행하는 것은 스크립트 하나입니다. aws batch처럼 도커 이미지를 띄워서 빌드하는 그런 개념도 아니기 때문에 프로젝트 단위로 ETL 소스를 개발하거나, 필요한 기능을 클래스 / 메서드 단위로 정리하기에도 참 애매합니다.

glue에서 좀 더 다양하게 내 프로젝트 기능을 사용할 수 있도록 테스트했던 내용을 기록합니다.



# Repository

https://github.com/BillMills/python-package-example

간단한 프로젝트입니다. myPackage라는 라이브러리가 있고 이를 빌드하기 위한 setup.py, 패키지가 저장될 dist 폴더 등이 있습니다.

python package 빌드에 대해서는 자세하게 다루지 않지만, aws glue에서 사용하기 위해서는 egg / whl 형태로 업로드해야 합니다. tar 형식으로 빌드했을때는 glue에서 인식하지 못했습니다.

setup.py로 프로젝트를 빌드하되, egg 파일로 압축하겠습니다.

```shell
python setup.py bdist_egg
```

![image1](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/ad1414eb-5875-4149-afce-c8253c496750/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20210401%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20210401T153058Z&X-Amz-Expires=86400&X-Amz-Signature=f5ae15d1b36153bcfcd492abdd8f1a5563daf9c16797d3ce84ec1ee84c5df1b0&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)



해당 egg 파일을 job -> Python library path에 입력합니다.

![image](https://user-images.githubusercontent.com/52685258/113318245-3b339a80-934b-11eb-966a-3f02bbe58ac7.png)



![image](https://user-images.githubusercontent.com/52685258/113318453-733add80-934b-11eb-920a-01dcf1d31e98.png)

glue script에서 이미 배포된 라이브러리 뿐만 아니라 custom library 사용이 가능합니다. 해당 라이브러리 패키징 → s3 업로드 과정은 codebuild를 이용해 자동화할 수 있습니다.

