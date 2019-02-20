# 로그 모니터링 구축


## 구성도
![](https://lh3.googleusercontent.com/mykpIaIYoitDmhVa2F9b6ojrVqjrF6J7iY52zc70_CHfLxRqWbljKxZCJsVXr7gyutIAzIUWDxyw)


### Elastic Search
* 분산형 RESTful 검색 및 분석 엔진
* 정형 데이터, 비정형 데이터, 위치 정보, 메트릭 등 다양한 유형의 데이터를 사용자가 원하는 방식으로 검색하고 결합할 수 있도록 지원


#### **JDK 설치**

```
$wget https://download.oracle.com/otn-pub/java/jdk/8u191-b12/2787e4a523244c269598db4e85c51e0c/jdk-8u191-linux-x64.tar.gz?AuthParam=1543373289_82c7ae5128ad66b8ccfe28ddd0464270
$tar xzf jdk-8u191-linux-x64.tar.gz\?AuthParam\=1543373289_82c7ae5128ad66b8ccfe28ddd0464270
$mv jdk1.8.0_191 /usr/local
```
```

$alternatives --install /usr/bin/java java /usr/local/jdk1.8.0_191/bin/java 2
$alternatives --config java

$alternatives --install /usr/bin/jar jar /usr/local/jdk1.8.0_191/bin/jar 2
$alternatives --install /usr/bin/javac javac /usr/local/jdk1.8.0_191/bin/javac 2
$alternatives --set jar /usr/local/jdk1.8.0_191/bin/jar
$alternatives --set javac /usr/local/jdk1.8.0_191/bin/javac

```
* JDK 설치 확인
```
$java -version

java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)

```
* 환경변수 설정
```
$vi /etc/profile
```
```
export JAVA_HOME=/usr/local/jdk1.8.0_144
export JRE_HOME=/usr/local/jdk1.8.0_144/jre
export PATH=$PATH:/usr/local/jdk1.8.0_144/bin:/usr/local/jdk1.8.0_121/jre/bin
```

#### **Elastic Search 설치 및 실행**



```
$rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
```
$wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.1.rpm

```
```
$vim /etc/yum.repos.d/elasticsearch.repo

[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

$yum install elasticsearch-6.5.1.rpm
```
* 설정 및 실행
```
$vim /etc/elasticsearch/elasticsearch.yml

network.host: 'IP_ADDR'
http.port: 9200
```
```
$systemctl start elasticsearch
```
* 작동상태 확인

```
$curl -IGET http://IP_ADDR:9200
```
```

{
  "name" : "sF5Prn1",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "AYQxzvKPRXKX2xGovAgv3w",
  "version" : {
    "number" : "6.5.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "8c58350",
    "build_date" : "2018-11-16T02:22:42.182257Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}

```
### Logstash
* 데이터를 처리하는 파이프라인
#### **Filebeat vs Logstash as a Log Shipper**

* Logstash는 JVM 기반으로 동작, Ruby도 같이 구동 → 과다한 메모리 소비량

* 적은 메모리 소모만으로 Logstash 기능을 대신할 수 있는 대안으로 Filebeat가 개발됨

* 주로 Filebeat가 로그 데이터를 수집, 정제하고, Logstash가 Elastic Search로 전달하는 역할 수행

* 불필요한 비트를 걸러내고, 필요 비트를 추출하는 과정은 Logstash로만 수행 가능

* Input, Filter, Output에 필요한 다양한 플러그인을 Logstash에서 제공

#### **구성**
![](https://shortstories.gitbooks.io/studybook/content/pipeline.png)
* Input
    * file - 파일시스템에 저장된 파일로부터 읽어옴
    * syslog - 보통 514번 포트로부터 들어오는 system log 메세지들을 받아 RFC3164 형식으로 파싱함.
    * redis - redis lists와 redis channels을 사용해서 redis server로부터 읽어올 수 있음.
    * 이외에도 beats, http, jdbc 등 다수의 Input이 있음
* Filter - Log 데이터를 Customize
    * grok - 로그 데이터를 파싱할 때 사용
    * mutate - rename, remove, replace 등 일반적인 편집을 수행
    * drop - 완전히 삭제
    * clone - 복제본을 생성. 필드를 추가하거나 제거
    * geoip - IP를 활용한 위치 파악에 사용
* Output
    * elasticsearch, file, graphite 등

#### **Logstash 설치 및 실행**
```
$rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
```
$vim /etc/yum.repos.d/logstash.repo

[logstash-6.x]
name=Elastic repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
```
$yum install logstash
```
* SSL Configuration
```
$vim /etc/pki/tls/openssl.cnf

v3_ca Section에 IP 추가
```
* SSL 인증서 발급
```
$openssl req -config /etc/pki/tls/openssl.cnf -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
```

* Logstash as a Shipper Configuration
```
$vim /etc/logstash-6.5.1/logstash.conf
input{
    redis{
    host => "REDIS_IP"
    port => "6379"
    codec => "json"
    data_type => "list"
    key => "logstash"
    }
}
output{
    elasticsearch{
        hosts => "ELASTICSEARCH_IP:9200"
    }
}

```
* 실행
```
$./etc/logstash-6.5.1/bin/logstash -f logstash.conf
```
### Kibana
수집된 데이터를 시각화하여 분석할 수 있는 Tool
#### **Kibana vs Grafana**

* 기능 상의 차이점
    * Kibana - Elastic Search 기반, Log Message 분석 지원
    * Grafana - Full-Text 데이터 쿼리 미지원

* Data Source 상의 차이점
    * Kibana - Elastic Search 기반에서만 동작. 다른 Data Source 기반 동작 불가
    * Grafana - InfluxDB, MySQL 등 여러 Data Source 기반 동작 가능

* Alert 기능
    * Kibana - 미지원. ELK Stack, X-Pack 등으로 구현
    * Grafana - Built in 지원. 
#### **Kibana 설치 및 실행**

```
$rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
```
$vim /etc/yum.repos.d/kibana.repo

[kibana-6.x]
name=Kibana repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
```
$yum install kibana
$systemctl start kibana
```

### 모니터링 대상에 Logstash 설치 및 연동

#### Metricbeat를 활용한 Resource Monitoring

* 설치
```
$curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-5.5.3-x86_64.rpm
$rpm -vi metricbeat-5.5.3-x86_64.rpm
```
* 설정

```
$vim /etc/metricbeat/metricbeat.yml

output.elasticsearch:

  hosts: ["ELASTIC_SEARCH_IP:9200"]

```
* 실행
```
$service metricbeat start
```
* Kibana 상에서 확인
![](https://lh3.googleusercontent.com/_JiCcwL2JPp-3NV-3nfxqWjOb_2N50_rtU30is3bzyzGWf_5IY-fQi5Gw7P5z3YSWb1JWh2Q55V1)
#### Apache

* Access Log Configuration
```
$vim /etc/httpd/conf/httpd.conf

<IfModule log_config_module>
LogFormat "{ \"time\":\"%{%Y-%m-%d}tT%{%T}t.%{msec_frac}tZ\", \"process\":\"%D\", \"filename\":\"%f\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\", \"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\" }," combined
LogFormat "\"%h\" \"%l\" \"%u\" \"%t\" \"%r\" %>s %b" common
LogFormat "{ \"time\":\"%t\", \"clientip\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\", \"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\" }" json_format
CustomLog "logs/jsonaccess_log" combined
CustomLog "logs/jsonacess_log" json_format
</IfModule>

service httpd restart
```
* Logstash as a Collector Configuration
```
$vim /etc/logstash-6.5.1/config/logstash.conf

input {
file {
    path => "/etc/httpd/logs/jsonaccess_log"
    type => apache
    codec => json
    }
}
filter {
    geoip {source => "remoteIP"}
}
output {
    redis {
    host => "REDIS_IP"
    port => 6379
    data_type => "list"
    key => "logstash"
    }
}

```
```
$./etc/logstash-6.5.1/bin/logstash -f config/logstash.conf
```

#### Tomcat
*  모듈 설치 및 연동

```
$wget http://apache.tt.co.kr/logging/log4j/1.2.17/log4j-1.2.17.tar.gz
$tar -xvf log4j-1.2.17.tar.gz

$cp apache-log4j-1.2.17/log4j-1.2.17.jar /etc/tomcat7-1/lib


#Installing Dependencies

$wget "http://central.maven.org/maven2/net/logstash/log4j/jsonevent-layout/1.7/jsonevent-layout-1.7.jar"
$wget "http://repo1.maven.org/maven2/commons-lang/commons-lang/2.4/commons-lang-2.4.jar"
$wget "http://repo1.maven.org/maven2/net/minidev/json-smart/1.1.1/json-smart-1.1.1.jar"
```
* Access Log

```
$vim /etc/logstash-6.5.1/config/logstash-tomcat.conf

input {
    file {
        path => "/etc/tomcat7-1/logs/localhost_access_log*"
        type => tomcat
        codec => json
    }
}

filter {
    geoip {source => "remoteIP"}
}

output {
    redis {
        host => "REDIS_IP"
        port => 6379
        data_type => "list"
        key => "logstash"
    }
}

```
### Kafka vs Redis
* 메시지 저장 장소의 차이
    * Redis - 메모리 / Kafka - 파일 시스템
    * 속도 측면에서 Redis가 우수하지만, 대용량의 데이터를 장기간 보관 불가
* 횡적 구성 / 종적 구성
    * Kafka - Broker가 클러스터로 구성
    * Redis - Master / Slave 구조

| Kafka | Redis |
| --- | --- |
| Disk 저장 | Memory 저장 |
| Data Partitioning 지원 | Data Partitioning 미지원 |
| 데이터 장기간 보관 가능 | 데이터 장기간 보관 불가 |
| 횡적 구성(Cluster) | 종적 구성(Master - Slave) |

### Kafka

* 오픈소스 기반의 분산 메시징 시스템. 대용량 실시간 로그처리에 특화
* 발행(Publish) - 구독(Subscribe) 모델 기반
    * 비동기 메시징 패러다임, 불특정된 수신자
    * 발행된 메시지는 정해진 범주에 따라, 범주에 대한 구독을 신청한 구독자 모두에 전달됨
    * 구독자는 발행자에 대한 지식 없이 원하는 메시지만 수신
#### **특징 및 장단점**
![](https://t1.daumcdn.net/cfile/tistory/253BF244550914E21A)
* Producer
    * 특정 Topic의 메시지를 생성, Broker에 전달
* Consumer
    * 쌓아둔 Topic 별 메시지를 각자 가져가서 처리
* Broker
    * Topic을 기준으로 메시지를 관리
* 확장성과 고가용성을 위해 Broker들이 클러스터로 구성
    * Broker의 분산 처리는 Zookeeper가 담당
![](https://t1.daumcdn.net/cfile/tistory/270D49435509151E2A)

* 메시지를 파일 시스템에 저장 - 데이터의 영속성 보장(일정 기간이 도래하면 데이터 삭제)
* Consumer가 Broker로부터 직접 메시지를 가지고가는 Pull 방식

* Topic의 Partitioning
![](https://t1.daumcdn.net/cfile/tistory/213CA13D5509180234)
    
   * Topic은 Partition 단위로 쪼개져 클러스터의 각 서버에 분산 저장됨
   * 장애 발생 시 Partition 단위로 Failover 수행

### Kafka 설치 및 설정 - 실패 사례
![](https://lh3.googleusercontent.com/cpQ21fDeJdDhsclzAIxWq_EPDiaCUeLoItO6QGJioYxCFBu8BYKXvGByhvN00_yT3JCrBpMELCpN)

* 3개의 인스턴스를 클러스터로 구성

#### **JDK 설치**

```
$wget https://download.oracle.com/otn-pub/java/jdk/8u191-b12/2787e4a523244c269598db4e85c51e0c/jdk-8u191-linux-x64.tar.gz?AuthParam=1543373289_82c7ae5128ad66b8ccfe28ddd0464270
$tar xzf jdk-8u191-linux-x64.tar.gz\?AuthParam\=1543373289_82c7ae5128ad66b8ccfe28ddd0464270
$mv jdk1.8.0_191 /usr/local
```
```

$alternatives --install /usr/bin/java java /usr/local/jdk1.8.0_191/bin/java 2
$alternatives --config java

$alternatives --install /usr/bin/jar jar /usr/local/jdk1.8.0_191/bin/jar 2
$alternatives --install /usr/bin/javac javac /usr/local/jdk1.8.0_191/bin/javac 2
$alternatives --set jar /usr/local/jdk1.8.0_191/bin/jar
$alternatives --set javac /usr/local/jdk1.8.0_191/bin/javac

```
* JDK 설치 확인
```
$java -version

java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)

```
* 환경변수 설정
```
$vi /etc/profile
```
```
export JAVA_HOME=/usr/local/jdk1.8.0_144
export JRE_HOME=/usr/local/jdk1.8.0_144/jre
export PATH=$PATH:/usr/local/jdk1.8.0_144/bin:/usr/local/jdk1.8.0_121/jre/bin
```

#### Zookeeper 설치 및 실행

* 각 Cluster hostname 등록

```
$ vi /etc/hosts

192.168.0.96 kafka.novalocal
192.168.0.97 kafka-2.novalocal
192.168.0.98 kafka-3.novalocal

```
* 다운로드 및 압축풀기
```
 $wget http://apache.tt.co.kr/zookeeper/zookeeper-3.4.12/zookeeper-3.4.12.tar.gz
 $tar xzf zookeeper-3.4.12.tar.gz
```

```
$mkdir /zookeeper-3.4.12/data
$echo 1 > data/myid     # #2, #3 Cluster는 2, 3으로 설정
```
```
$vim /zookeeper-3.4.12/conf/zoo.cfg

dataDir=                      #myid 위치

server.1=#1_HOSTNAME:2888:3888
server.2=#2_HOSTNAME:2888:3888
server.3=#3_HOSTNAME:2888:3888
```

* Zookeeper Service 등록

```
[Unit]
Description=zookeeper-server
After=network.target

[Service]
Type=forking
User=root
Group=root
SyslogIdentifier=zookeeper-server
WorkingDirectory=/etc/kafka_2.12-2.0.1/zookeeper-3.4.12
Restart=always
RestartSec=0s
ExecStart=/etc/kafka_2.12-2.0.1/zookeeper-3.4.12/bin/zkServer.sh start
ExecStop=/etc/kafka_2.12-2.0.1/zookeeper-3.4.12/bin/zkServer.sh stop

[Install]
WantedBy=multi-user.target

```
```
$systemctl start zookeeper-server.service
```
#### Kafka 설치 및 Zookeeper와 연결

```
$wget http://www-us.apache.org/dist/kafka/2.0.1/kafka_2.12-2.0.1.tgz
$tar xvf kafka_2.12.-2.0.1.tgz


```
```
$vim /etc/kafka_2.12-2.0.1/config/server.properties


broker.id=1                             #설정한 Server ID

log.dirs=                                  #분산 data dir


zookeeper.connect=kafka.novalocal:2181,kafka-2.novalocal:2181,kafka-3.novalocal3:2181/zookeeperex    #zookeeper 연결 설정
```
```
$vim /etc/systemd/system/kafka-server.service

[Unit]
Description=kafka-server
After=network.target

[Service]
Type=simple
User=root
Group=root
SyslogIdentifier=kafka-server
WorkingDirectory=/etc/kafka_2.12-2.0.1
Restart=always
RestartSec=0s
ExecStart=/etc/kafka_2.12-2.0.1/bin/kafka-server-start.sh /etc/kafka_2.12-2.0.1/config/server.properties
ExecStop=/etc/kafka_2.12-2.0.1/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target

```
```
$ systemctl start kafka-server.service
```
### Redis 설치 및 설정 - 대안

#### 대안 설정 이유
* 클러스터 별 Zookeeper, Kafka 연동 상의 오류
* 클러스터 설정 오류로 Kafka에 Log Data 저장 실패

#### Redis 설치 및 설정
* 설치
```
$yum -y install redis
```
* 설정
```
$vim /etc/redis.conf

protected-mode no
# bind 127.0.0.1
```
```
$service redis start
```

### Kibana에서 Log 값 확인

#### Resource

![](https://lh3.googleusercontent.com/cJsDyjJQFkUA5jS-W3rikl8trohQ9KWlpeMCx2WkknwH98Bz9FZR95RAtjxrK0v_qrfg1VhZ8kUi)
#### Apache
* Access Log

![](https://lh3.googleusercontent.com/4mS5TxlRYjsfGUPg-Lztr5PR0aYmESuMfFwUmzwTtYG_ttmvKoFviEt39iM-Cg8IJD1AGFNWI2oc)

![](https://lh3.googleusercontent.com/r-Ng8Bo1LRAN-rucm2QpkwSXYzuUmhvVFFjIzdLediNlhk7tZgkACvRS_WYhqK65S787D2T8_-_I)


#### Tomcat

* Access Log

![](https://lh3.googleusercontent.com/QhbQSFYSnu7oZXhJlFjdweY-YtXOetZeby7NNyVFPQpNMcvLSGCJE46FAqqcu_WTT-uCEvN95LZx)
## 보완점
* Error Log
* Tomcat JSON Parsing
* Resource Monitoring
    * Resource Log 전송 루트 변경 (Elastic Search → Logstash)
    * Dashboard Customizing
* Alert 기능
