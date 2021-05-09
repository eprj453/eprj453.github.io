---
title: "[spark] Docker Ubuntu 컨테이너로 Spark 실습환경 만들기 1. 컨테이너 준비"
date: 2021-05-08 00:00:00
categories:
- spark
tags: [spark, docker]

---



최근 스파크를 공부하고 있습니다. 실무에서 aws glue를 사용하면서 직접 스파크 코드를 작성하기도 하지만, 단순히 문법이나 메서드보다는 spark의 클러스터 환경을 직접 구축해보고 경험해보는 게 더 값지고 재밌을거라 생각했습니다.

위키북스의 [빅데이터 분석을 위한 스파크 2 프로그래밍](https://wikibook.co.kr/spark2nd/) 책을 따라하며 실습중인데, 만족스럽습니다. 클러스터 / 마스터 / 워커 등등 글로는 이해하기 어려울 수 있는 부분들을 최대한 직접 코드로 짜보며 따라해볼 수 있게 만들어 놓은것 같아 저같은 신입들에게 좋을 것 같습니다.

스파크 개념서들은 RDD / dataframe / dataset의 개념과 사용할 수 있는 메서드 등등 데이터 자체의 활용을 전면으로 내세우는 경우가 많은데 비교적 앞쪽인 3장에 클러스터 환경에 대한 내용이 있어 클러스터 매니저 / spark submit 등을 먼저 경험할 수 있게 배치된 것도 좋습니다.

이 책에서 저자는 같은 환경 서버 3대를 가지고 master 1대, slave 2대로 스파크 실습 환경을 구축했습니다. 서버 3대를 가용하기 부담스러운 경우에는 VM을 띄우거나 도커 컨테이너를 띄워서 여러 대의 서버를 돌릴 수 있는데 그 중 도커 컨테이너를 가지고 master 1대, worker 1대를 띄우는 과정을 기록합니다.

<br/>

# 1. 요구사항

- master / slave용으로 각각 하나의 컨테이너
- 각 컨테이너에서는 하둡(정확히는 HDFS)을 사용할 수 있어야 하고 하둡 namenode / datanode는 master 서버에서 가동

책에서 나온 환경과 100% 일치하지는 않지만 실습에는 큰 무리가 없었습니다.

<br/>

# 2. 이미지 준비

먼저 ubuntu 이미지를 받은 뒤 컨테이너를 띄우고 bash로 접속합니다.

```bash
docker pull ubuntu
docker run -itd --name spark-base ubuntu
docker exec -it spark-base bash
```

컨테이너에 접속했다면 필요한 패키지 및 라이브러리를 설치합니다.

```bash
apt-get update
apt-get install vim wget unzip ssh openjdk-8-jdk python3-pip
pip3 install pyspark
```

이제 실습에 필요한 환경을 설정한 뒤 컨테이너를 이미지로 commit하겠습니다. 기본 준비사항은 책 190P부터 시작하는데, 크게

- 네트워크(ssh)
- JAVA
- HADOOP

3가지를 요구하고 있습니다. 먼저 위에서 openjdk로 자바를 설치했으니 환경변수를 등록해줍니다.

```bash
vi ~/.bashrc

export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```



## 하둡 바이너리 다운로드

하둡은 바이너리 파일을 다운로드받아 환경변수를 설정하는 것으로 충분합니다. path / 하둡 버전은 본인에 맞게 설정해주세요.

```bash
# 폴더 생성 및 하둡 바이너리 파일 다운로드 / 압축 해제
mkdir /opt/hadoop && cd /opt/hadoop
wget <https://mirror.navercorp.com/apache/hadoop/common/hadoop-3.2.2/hadoop-3.2.2.tar.gz>
tar -xzf hadoop-3.2.2.tar.gz
rm hadoop-3.2.2.tar.gz

# 환경변수 등록
vi ~/.bashrc
export HADOOP_HOME=/opt/hadoop/hadoop-3.2.2
export PATH=${HADOOP_HOME}/bin:$PATH

source ~/.bashrc
```

그 외에 기본적인 hadoop 설정(core-site.xml, hdfs-site.xml....)은 이 [미디엄 포스트](https://alibaba-cloud.medium.com/how-to-install-hadoop-cluster-on-ubuntu-16-04-bd9f52e5447c)를 참고했습니다. HDFS를 사용할 때 namenode에 대한 설정이 되어있는것이 편하기 때문에 설정하는 것을 권장합니다.



## 스파크 바이너리 다운로드

하둡과 동일하게 스파크도 바이너리 파일을 설치합니다.

```bash
# 폴더 생성 및 하둡 바이너리 파일 다운로드 / 압축 해제
mkdir /opt/spark && cd /opt/spark
wget <https://mirror.navercorp.com/apache/spark/spark-3.1.1/spark-3.1.1-bin-hadoop2.7.tgz>
tar -xzf spark-3.1.1-bin-hadoop2.7.tgz
rm spark-3.1.1-bin-hadoop2.7.tgz

# 환경변수 등록
vi ~/.bashrc
export SPARK_HOME=/opt/spark/spark-3.1.1-bin-hadoop2.7.tgz
export PATH=${SPARK_HOME}/bin:$PATH
export PYSPARK_PYTHON=/usr/bin/python3

source ~/.bashrc
```



## 이미지 생성

환경설정이 완료된 컨테이너를 이미지로 만듭니다. docker commit에 대한 공식문서는 [여기](https://docs.docker.com/engine/reference/commandline/commit/)

```bash
docker commit spark-base hadoop-spark

docker images
REPOSITORY                                                           TAG                          IMAGE ID       CREATED        SIZE
psw/hadoop-spark                                                     latest                       a258b26665df   2 hours ago    2.74GB
```

<br/>

# 3. 마스터 컨테이너 띄우기

먼저 마스터로 사용할 컨테이너를 띄웁니다.

```bash
docker run -itd --name spark-master -p 9870:9870 -p 8080:8080 hadoop-spark
```

포트의 용도는

- 9870 : hadoop namenode webserver
- 8080: spark master webserver

입니다.

포트가 잘 바인딩되었는지 확인하고, 컨테이너에 접속합니다.

```bash
docker exec -it spark-master bash
```

책 191P의 예제 파일 HDFS 업로드를 실행합니다. namenode host와 port는 위에서 설정한 하둡 파일 중 `core-site.xml`의 `fs.defaultFS`의 value를 따라갑니다.

```bash
# hdfs dfs -mkdir hdfs://{namenode_host:port}/sample
hdfs dfs -mkdir hdfs://localhost:9000/sample

cd ${SPARK_HOME}
hdfs dfs -put ./README.md hdfs://localhost:9000/sample/

hdfs dfs -ls hdfs://localhost:9000/sample
Found 1 items
-rw-r--r--   1 root supergroup       4488 2021-05-08 22:21 hdfs://localhost:9000/sample/README.md
```

여기까지 잘 처리됐다면, 클러스터 구성에 필요한 기본준비가 완료된 것입니다. 다음 장에서는 master 컨테이너와 worker 컨테이너를 연결시키고 spark-submit으로 스파크 어플리케이션을 실행해보겠습니다.