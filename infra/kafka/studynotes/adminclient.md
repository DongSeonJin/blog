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

* `describeTopics()`를 호출하고, 비동기 결과(`KafkaFuture`)를 `.get()`으로 기다려 `TopicDescription` 정보를 받아온다.



&#x20;TOPIC 목록 조회

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









