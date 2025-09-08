---
description: 카프카 어드민 클라이언트
---

# AdminClient

카프카의 AdminClient는 카프카 클러스터를 코드(애플리케이션)를 통해 관리하고 모니터링할 수 있도록 도와주는 API다. Kafka 0.11.0.0 버전부터 도입되었으며, 이전에는 셸 스크립트(`kafka-topics.sh` 등)로 수행하던 관리 작업을 프로그래밍 방식으로 처리할 수 있게 해준다.

토픽 목록 조회, 생성, 삭제, 클러스터 상세 정보 확인, ACL 관리, 설정을 확인하거나 변경하는 등의 관리 작업을 자동화하거나 자체 관리 도구를 만드는 데 사용된다.





### :bulb:AdminClient 의 비동기적 처리와 최종적 일관성

> Kafka의 AdminClient는 대부분의 작업을 비동기(Asynchronous) 방식으로 처리한다. 즉, 토픽을 만들거나 삭제하라는 명령을 내렸을 때, 그 작업이 끝날 때까지 기다리지 않고 즉시 다음 코드를 실행한다.
>
> AdminClient의 메서드들(예: `createTopics()`, `deleteTopics()`)은 호출하면 즉시 `XxxResult` 객체(예: `CreateTopicsResult`)를 반환한다. 그리고 이 객체 안에는 `KafkaFuture` 라는 특별한 객체가 들어있다.
>
> 이 `KafkaFuture`를 통해 비동기 작업의 결과를 확인하거나, 각각의 토픽 상태를 하나씩 확인하거나, 작업이 완료되었을 때 특정 동작을 하도록 만들 수 있다.
>
>
>
> &#x20;AdminClient의 비동기 요청을 받아서 클러스터 전체에 일관성을 맞추는 작업은 카프카 내부의 컨트롤러 브로커가 알아서 처리한다. 이 시간 동안 어떤 브로커는 새 토픽을 알고, 어떤 브로커는 아직 모르는 '일시적 불일치 상태'가 존재할 수 있다.
>
> 하지만 결국에는 모든 브로커가 컨트롤러로부터 변경 사항을 전달받아 동일한 메타데이터 상태를 갖게 된다. 이것을 최종적 일관성이라고 한다.





## 옵션

AdminClient의 각 메서드는 메서드별로 특정한 Options 객체를 인수로 받는다.

AdminClient의 `Options` 객체는 API 메서드를 호출할 때, 기본적인 파라미터 외에 세부적인 동작 방식을 제어하기 위해 사용하는 설정 객체다. 예를들어, `createTopics()` 메서드는 `CreateTopicsOptions` 객체를, `deleteTopics()`는 `DeleteTopicsOptions` 객체를 추가 파라미터로 받는다.

* 타임아웃 설정 (`timeoutMs`): 클라이언트가 브로커로부터 응답을 받을 때까지 기다릴 최대 시간을 밀리초(ms) 단위로 지정한다. 이 시간을 초과하면 타임아웃 예외가 발생한다.
* Dry Run (`validateOnly`): 가장 유용한 기능 중 하나다. `true`로 설정하면, 실제 명령을 실행하지 않고 요청이 성공할 수 있는지 유효성 검사만 수행한다. 예를 들어, 중요한 토픽을 삭제하기 전에 명령이 유효한지 미리 확인하는 등의 작업에 매우 유용하다.
* 재시도 정책 설정: 특정 조건에서 요청을 재시도할지 여부 등을 설정할 수 있다.









## SpringBoot 에서 AdminClient 사용법

```yaml
  kafka:
    bootstrap-servers: localhost:9093
    admin:
      properties:
        request.timeout.ms: 5000 # 요청 타임아웃 5초
```

* application.yml 파일에 위와같이 추가해준다.&#x20;



#### TOPIC 생성

```java
@Service
@Slf4j
@RequiredArgsConstructor
class KafkaAdminServiceImpl implements KafkaAdminService {
    private final KafkaAdmin kafkaAdmin;

    @Override
    public boolean createTopic(String topicName, int partitions, int replicationFactor) {
        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) { 
            NewTopic newTopic = new NewTopic(topicName, partitions, (short) replicationFactor);

            // 토픽 생성 옵션 (타임아웃 5초)
            CreateTopicsOptions options = new CreateTopicsOptions().timeoutMs(5000);
            // 토픽 생성 요청 및 결과 확인
            adminClient.createTopics(Collections.singleton(newTopic), options).all().get();
            log.info("토픽 '{}'이 성공적으로 생성되었습니다.", topicName);
            return true;
        } catch (ExecutionException | InterruptedException e) {
            log.error("토픽 '{}' 생성 중 오류가 발생했습니다: {}", topicName, e.getMessage());
            return false;
        }
    }
}
```

* `application.yml`을 기반으로 자동 생성한 `KafkaAdmin` 빈을 주입받는다. 이 빈을 통해 `AdminClient` 인스턴스를 생성하고 관리할 수 있다.
* `AdminClient`는 사용 후 반드시 `close()`를 호출하여 리소스를 해제해야 한다. `try-with-resources` 구문을 사용하면 자동으로 `close()`가 호출되어 편리하다.
* `NewTopic` 객체로 토픽 정보를 정의하고 `createTopics()`를 호출하여 토픽을 생성한다.





#### TOPIC 정보 조회

```java
@Override
public void describeTopic(String topicName) {
    try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {
        DescribeTopicsResult result = adminClient.describeTopics(Collections.singleton(topicName));
        Map<String, TopicDescription> topicInfo = result.allTopicNames().get();

        log.info("토픽 정보 [{}]: {}", topicName, topicInfo);
    } catch (ExecutionException | InterruptedException e) {
        log.error("토픽 '{}' 정보 조회 중 오류가 발생했습니다: {}", topicName, e.getMessage());
    }
}
```

* `DescribeTopicsResult`  객체 안에는 키(토픽 이름), 토픽에 대한 상세 정보를 담은 Future 객체를 밸류로 하는 맵이 들어 있다.
* `describeTopics()`를 호출하고, 비동기 결과(`KafkaFuture`)를 `.get()`으로 기다려 `TopicDescription` 정보를 받아온다. (.get()은 블로킹 방식)
* `TopicDescription` 객체는 다음과 같은 상세 정보를 담고 있다.

> `name()`: 토픽의 이름.
>
> `topicId()`: 클러스터 내에서 토픽을 식별하는 고유 ID.
>
> `isInternal()`: 카프카 내부에서 사용하는 토픽(예: `__consumer_offsets`)인지 여부.
>
> `partitions()`: 가장 중요한 정보로, `TopicPartitionInfo` 객체들의 리스트를 반환한다.

* `whenComplete()` : 블로킹 방식인 .get() 과 달리 논블로킹 방식이다. 람다식을 인자로 받아 비동기 작업을 할 수 있고, 그 결과를 처리하기 위한 콜백 함수를 등록할 수 있다.





#### &#x20;TOPIC 목록 조회

```java
@Override
public Set<String> getTopicList(String topicName) {
    try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {
        // listTopics()는 ListTopicsResult를 반환한다.
        ListTopicsResult topics = adminClient.listTopics();
        // .names()를 통해 토픽 이름 Set을 담은 KafkaFuture를 얻는다.
        KafkaFuture<Set<String>> names = topics.names();
        // .get()으로 결과를 기다려 실제 Set<String>을 가져온다.
        Set<String> topicNames = names.get();

        log.info("🔍 조회된 토픽 수: {}", topicNames.size());
        return topicNames;
    } catch (ExecutionException | InterruptedException e) {
        if (e instanceof InterruptedException) {
            Thread.currentThread().interrupt();
        }
        log.error("토픽 목록 조회 중 오류가 발생했습니다: {}", e.getMessage());
        // 실패 시에는 null 대신 비어있는 컬렉션을 반환하는 것이 더 안전하다.
        return Collections.emptySet();
    }
}
```





#### 비동기 방식 TOPIC 정보 조회

```java
@Override
public void describeTopicAsync(String topicName) {
    try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {
        DescribeTopicsResult result = adminClient.describeTopics(Collections.singleton(topicName));
        KafkaFuture<Map<String, TopicDescription>> future = result.allTopicNames();

        // whenComplete 콜백 등록
        future.whenComplete((topicInfoMap, throwable) -> {
            if (throwable == null) {
                TopicDescription description = topicInfoMap.get(topicName);
                log.info("(Callback) 토픽 정보 [{}]: {}",  topicName, description);
            } else {
                log.error("(Callback) 토픽 '{}' 정보 조회 중 오류 발생: {}",  topicName, throwable.getMessage());
            }
        });
    }
}
```

* `whenComplete()` : 블로킹 방식인 .get() 과 달리 논블로킹 방식이다. 람다식을 인자로 받아 비동기 작업을 할 수 있고, 그 결과를 처리하기 위한 콜백 함수를 등록할 수 있다.









### 설정관리 (ConfigResource)

카프카 설정 관리는 `AdminClient`와 `ConfigResource` 객체를 활용하여 토픽이나 브로커 같은 대상의 설정을 코드로 직접 조회하고 바꾸는 것이다. 이를 사용하면 수동 작업 없이 설정 변경을 자동화할 수 있다.



`ConfigResource`를 활용한 작업은 크게 두 가지다.

* 설정 조회 (`describeConfigs`): `ConfigResource`로 대상을 지정해서 현재 설정값이 뭔지 전부 받아보는 작업이다. `retention.ms`(메시지 보관 기간)나 `cleanup.policy`(정리 정책) 같은 값들을 확인할 수 있다.
* 설정 변경 (`alterConfigs`): `ConfigResource`로 대상을 지정하고, 바꾸고 싶은 설정값을 담아 보내는 작업이다. 이걸로 서비스 운영 중에 코드를 통해 특정 토픽의 설정을 동적으로 바꿀 수 있다.





















