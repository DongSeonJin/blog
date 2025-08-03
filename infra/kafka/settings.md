# Settings

카프카와 주키퍼는 모든 openJDK 기반 자바 구현체 위에서 원활히 작동한다.&#x20;

카프카 최신 버전은 자바8과 11을 모두 지원한다.



{% embed url="https://www.oracle.com/kr/java/technologies/downloads/#jdk21-mac" %}

오라클 홈페이지에서 jdk 설치를 먼저 해주자.



```
brew install zookeeper

brew services start zookeeper

zkCli ls /
```

zookeeper 를 설치하고 kCli ls / 입력하여 정상적으로 실행 중인지 확인한다.\


```
brew install telnet

telnet localhost 2181

srvr
```

telnet 접속 후 srvr 입력시 주키퍼 버전, 지연시간, 시션 수, 노드 상태 등 다양한 정보를 볼 수 있다. :arrow\_down:

```
> telnet localhost 2181
Trying ::1...
Connected to localhost.
Escape character is '^]'.
srvr
Zookeeper version: 3.9.3-${mvngit.commit.id}, built on 2024-10-24 22:41 UTC
Latency min/avg/max: 0/5.5/15
Received: 9
Sent: 8
Connections: 1
Outstanding: 0
Zxid: 0x6
Mode: standalone
Node count: 5
Connection closed by foreign host.
```





## 카프카 설치

```
brew install kafka
```

윈도우 가이드 자료에는 명령어 끝에 .sh가 붙어있다.\
예) kafka-server-start.sh

하지만 homebrew 로 설치한 mac 환경에서는 저  명령어를 입력하면 command not found 오류가 나타난다.

```
ls -l /opt/homebrew/bin | grep kafka
```

위 명령어로 homebrew/bin 경로의 카프카 실행파일 들 확인해보면

```
lrwxr-xr-x@ 1 jindongseon  admin    57  8  3 02:53 kafka-streams-application-reset -> ../Cellar/kafka/4.0.0/bin/kafka-streams-application-reset
lrwxr-xr-x@ 1 jindongseon  admin    38  8  3 02:53 kafka-topics -> ../Cellar/kafka/4.0.0/bin/kafka-topics
lrwxr-xr-x@ 1 jindongseon  admin    44  8  3 02:53 kafka-transactions -> ../Cellar/kafka/4.0.0/bin/kafka-transactions
lrwxr-xr-x@ 1 jindongseon  admin    51  8  3 02:53 kafka-verifiable-consumer -> ../Cellar/kafka/4.0.0/bin/kafka-verifiable-consumer
lrwxr-xr-x@ 1 jindongseon  admin    51  8  3 02:53 kafka-verifiable-producer -> ../Cellar/kafka/4.0.0/bin/kafka-verifiable-producer
```

이런식으로 .sh 가 생략되어 있는것을 확인할 수 있다.

```
#카프카 서버 실행
kafka-server-start /opt/homebrew/etc/kafka/server.properties

#토픽 생성 
kafka-topics --create --replication-factor 1 --partitions 1 --topic test --bootstrap-server localhost:9092
```







## 토픽에 메시지 쓰기

```
kafka-console-producer --bootstrap-server localhost:9092 --topic test
Test Message 1
Test message 2
```

test 토픽에 메시지를 쓴다. 프로듀서 멈추기는 ^C

```
/kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
Test Message 1
Test message 2
^C
Processed a total of 2 messages
```

test 토픽에서 메시지 읽기.

방금 test 토픽에 쓴 Test Message 1, Test Message 2 가 출력된다면 잘 된거다.



