# Settings

카프카와 주키퍼는 모든 openJDK 기반 자바 구현체 위에서 원활히 작동한다.&#x20;

카프카 최신 버전은 자바8과 11을 모두 지원한다.

책은 주키퍼 기반으로 되어있지만, 카프카4.0 부터 주키퍼는 지원을 중단했기 때문에 Kraft로 대체 한다.



{% embed url="https://www.oracle.com/kr/java/technologies/downloads/#jdk21-mac" %}

오라클 홈페이지에서 jdk 설치를 먼저 해주자.



```
cd /opt/homebrew/etc/kafka
```

&#x20;위 경로로 이동하여 server.properties 파일을 수정해주자.

```
controller.quorum.voters=1@localhost:9093
```

&#x20;파일 안에 추가

```
kafka-storage random-uuid
```

Kraft 모드는 클러스터를 식별할 고유 ID가 필요하다. 위 명령어로 ID를 생성하고 복사해놓자.

```
kafka-storage format -t <YOUR_CLUSTER_ID> -c <CONFIG_FILE_PATH>/server.properties
```

설정이 완료되면, 생성한 클러스터 ID를 사용하여 카프카 데이터 디렉토리를 Kraft용으로 포맷해야 한다. 이 과정은 클러스터 생성 시 단 한 번만 수행된다.





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

방금 test 토픽에 쓴 Test Message 1, Test Message 2 가 출력 될 것이다.







## 하드웨어 적 고려사항

* CPU

CPU Bound Application과 I/O Bound Application중 카프카는 데이터 입출력 작업에 치중되어 있기때문에 I/O Bound Application에 가깝다. 그렇기때문에 CPU 성능이 많이 요구되지는 않는다.



