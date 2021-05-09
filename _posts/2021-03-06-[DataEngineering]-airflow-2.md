---
title: "[airflow] airflow 사용기 2. docker/airflow 설정 파일"
date: 2021-03-06 00:00:00
categories:
- airflow
tags: [data-engineering, airflow]
---





docker / airflow의 설정 파일을 크게 3가지로 나눴습니다.

**docker-compose.yml / Dockerfile / airflow.cfg**



# docker-compose.yml

docker 컨테이너를 띄우는 명령어는 다음과 같습니다.

```bash
$ docker run ubuntu:20.04
```

여기에 필요에 따라 -it(키보드 입력을 하기 위한 옵션), —rm(프로세스가 종료되면 자동으로 컨테이너 삭제) 등의 옵션을 추가해 컨테이너를 띄워 사용합니다.

그런데 컨테이너가 여러개일 경우 이 명령어를 하나하나 치는 건 비효율적입니다. 그래서 여러 개의 컨테이너를 하나의 파일에서 빌드 / 관리하기 위해 docker-compose.yml 파일을 사용합니다. 형태는 다음과 같습니다.

```yaml
version: '2'
services:
  db:
    image: mysql:5.7
    volumes:
      - ./mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: wordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
  wordpress:
    image: wordpress:latest
    volumes:
      - ./wp:/var/www/html
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_PASSWORD: wordpress
```

services 밑으로 여러 개의 서비스들이 있습니다. 예시 문서에는 db, wordpress 2개의 컨테이너가 있습니다. 이전 포스트에서 소개했던 [puckel/docker-airflow](https://github.com/puckel/docker-airflow)의 compose.yml 파일에도 airflow에 필요한 서비스들이 적혀있는 걸 볼 수 있습니다.

간략히 이야기하면, compose 파일은 컨테이너를 위한 설정파일입니다.

## .env

airflow에서 aws에 접근해야 하기 때문에 airflow 컨테이너는 aws access key, aws access secret key를 알고 있거나, 프로세스 실행 시에 읽어올 수 있어야 합니다. .env에 환경 변수로 세팅을 해줬는데, 컨테이너가 뜨는 시점에 환경변수를 추가하기 위해 docker-compose.yml에 세팅해줬습니다.

## 컨테이너가 여러개인데 Dockerfile에 ENV를 추가해 환경변수를 설정해주면 컨테이너에 하나씩 추가할 필요가 없지 않나?

맞습니다. Dockerfile에서도 환경변수를 추가할 수 있고, 그러면 컨테이너를 띄우는 시점에 환경변수를 추가하지 않아도 됩니다. 이미지를 빌드할 때 이미 추가될테니까요. 방법은 [여기](https://help.cloud66.com/maestro/how-to-guides/build-and-config/env-vars-in-dockerfile.html)

그런데 aws key는 외부에 노출되면 안되기 때문에 .env 파일에 따로 보관했습니다. 이 .env는 일반적으로 gitignore 설정을 해두어 github / codecommit / bitbucket 등 공용 repository에 올라가지 않도록 합니다.

Dockerfile에서 .env 파일에 있는 환경변수를 바로 읽는 방법을 찾지 못했습니다. compose 파일에서는 .env 파일을 읽어올 수 있기 때문에 번거롭지만 보안을 위해 이 방법을 택했습니다.

환경변수가 필요한 컨테이너마다 `env_file`에 .env 파일을 통으로 등록하거나 `environment`에 개별적으로 변수를 등록할 수 있습니다. 자세한 방법은 [여기](https://docs.docker.com/compose/environment-variables/)

```yaml
env_file:
  - .env

environment: 
		AWS_ACCESS_KEY="${AWS_ACCESS_KEY}"
```



# Dockerfile

compose 파일이 컨테이너를 위한 설정파일이라면, Dockerfile은 이미지를 위한 설정파일입니다. 이미지를 가지고 컨테이너를 띄운다라고 생각하면 어느정도 이해가 쉽게 됩니다.

이미지 빌드를 위한 파일인만큼 Dockerfile은 대부분 명령어의 형태로 이루어져 있습니다. 이미지를 빌드하면 Dockerfile에 정의되어 있는 명령어가 차례대로 실행되는 것을 볼 수 있습니다. compose에서 이야기했던 것처럼 컨테이너에 관계없이 공통으로 사용해야 하는 설정이나 필요한 라이브러리의 설치 등은 Dockerfile에 세팅해놓으면 좋습니다.



# airflow.cfg

파일 이름에서도 알 수 있지만 여기는 airflow 설정을 위한 파일입니다. dags_folder / log_folder 등 path, airflow가 사용하는 db, executor, timezone 등의 설정을 이 파일에서 할 수 있습니다.