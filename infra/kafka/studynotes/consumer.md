---
description: 카프카 컨슈머
---

# Consumer



카프카에서 데이터를 읽는 애플리케이션은 토픽을 구독하고 구독한 토픽들로부터 메시지를 받기 위해 KafkaConsumer를 사용한다. 카프카에서 데이터를 읽는 것은 다른 메시지 전달 시스템에서 데이터를 읽는 것과는 조금 다르다.





## 컨슈머와 컨슈머 그룹

> 단일 컨슈머가 토픽을 구독하고 메시지를 받기 시작한 뒤 받은 메시지를 받아 검사하고 결과를 쓴다고 생각해보자. 어떤 문제가 있을까? 만약 프로듀서가 애플리케이션이 검사할 수 있는 속도보다 더 빠른 속도로 토픽에 메시지를 쓰기 시작한다면, 새로 추가되는 메시지의 속도를 따라잡지 못하고 결국에는 메시지 처리가 뒤로 밀릴 것이다.
>
> 하나의 토픽이 다수의 파티션으로 나뉘어 있을 경우, 컨슈머 그룹 내의 컨슈머들은 각기 다른 파티션을 할당받아 병렬로 메시지를 처리한다. 이를 통해 소비 처리량을 수평적으로 확장할 수 있다.





## Springboot 컨슈머 설정

```
# 3. 컨슈머(Consumer) 설정
consumer:
  # 컨슈머 그룹 ID (필수)
  group-id: my-group
  # 브로커로부터 받은 바이트를 객체로 변환하는 역직렬화(Deserializer) 클래스 지정
  key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
  value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
  # (옵션) 처음 연결 시 읽을 오프셋 위치 (earliest: 가장 오래된 메시지부터)
  auto-offset-reset: earliest
```







## 파티션 할당 전략

_리밸런스에는 컨슈머 그룹이 사용하는 파티션 할당 전략에 따라 2가지가 있다._

#### 1. 조급한 리밸런스(eager rebalance)

> 컨슈머 그룹에 변경 사항(새 컨슈머 추가, 기존 컨슈머 이탈 등)이 발생했을 때, 그룹 내 모든 컨슈머가 자신이 담당하던 모든 파티션을 즉시 포기하고, 메시지 처리를 완전히 중단하는 'Stop-the-World' 방식으로 동작한다.
>
> 이 방식의 가장 큰 단점은 리밸런스가 발생하는 동안 그룹 전체의 데이터 처리가 멈추기 때문에, 일시적인 서비스 중단이 발생한다는 점이다.

#### 2. 협력적 리밸런스(cooperative rebalance)

> 컨슈머 그룹에 변경 사항이 발생해도, 모든 컨슈머가 작업을 멈추는 대신 재할당이 필요한 파티션만 일부 컨슈머가 반납하고, 컨슈머 그룹 리더가 반납된 파티션들을 새로 할당한다. 나머지 컨슈머들은 중단 없이 메시지 처리를 계속하는 방식이다.
>
> 이 방식의 가장 큰 장점은 리밸런스 중에도 컨슈머 그룹 전체가 멈추지 않아, 서비스 중단 시간을 최소화하고 안정성을 크게 향상시킨다는 점이다.





## 스프링부트에서 컨슈머 설정하기

```yaml
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: earliest
```

* application.yml 파일을 위와 같이 설정해준다.





```java
@Service
public class KafkaConsumerService {

    private static final String TOPIC_NAME = "test-topic";
    private static final String GROUP_ID = "my-group";

    @KafkaListener(topics = TOPIC_NAME, groupId = GROUP_ID)
    public void listen(String message) {
        System.out.println("Received message: " + message);
    }
}
```

* GROUP\_ID 는 KafkaConsumer 인스턴스가 속하는 컨슈머 그룹을 지정하는 속성이다.





### &#x20;:pencil2:순수 자바로 구현하는 컨슈머

```java
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

// 3. 토픽 구독 (subscribe 호출)
consumer.subscribe(Collections.singletonList("my-topic"));

// 여러 토픽 구독도 가능
// consumer.subscribe(Arrays.asList("topic-a", "topic-b"));

// 정규식을 사용한 구독 (예: 'test-'로 시작하는 모든 토픽)
// consumer.subscribe(Pattern.compile("test-.*"));

System.out.println("Consumer started. Subscribed to 'my-topic'.");

// 4. 무한 루프를 돌며 메시지를 지속적으로 가져오기 (Polling)
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("Received message: key = %s, value = %s, partition = %d, offset = %d%n",
                record.key(), record.value(), record.partition(), record.offset());
    }
}
```

> 순수 Java Kafka 클라이언트를 사용하는 환경의 경우는 위 코드처럼 subscribe 함수를 사용하여 토픽을 구독할 수 있고, poll 함수를 실행하여 할당된 파티션으로부터 메시지를 가져온다. 하지만, Spring Boot 환경에서는 `@KafkaListner` 어토테이션을 사용하여 subscribe()과 poll() 실행을 자동으로 처리해준다







## 스레드 안전성

하나의 스레드에는 반드시 하나의 컨슈머만 사용한다'\*\*는 원칙을 지켜야 한다.

**1. 하나의 컨슈머를 여러 스레드가 공유하면 안 된다.**

하나의 컨슈머 인스턴스를 여러 스레드가 공유하면 내부 상태가 망가진다. 메시지 처리 순서를 기록하는 오프셋(offset) 정보가 꼬이고, 브로커와의 통신(heartbeat)에 문제가 생긴다. 결과적으로 메시지가 중복 처리되거나 유실될 수 있으며, 컨슈머 그룹에서 이탈되는 심각한 오류가 발생한다.

**2. 하나의 스레드에 여러 컨슈머를 생성하는 것은 무의미하다.**

하나의 스레드는 코드를 순차적으로 실행하므로 병렬 처리가 불가능하다. 한 스레드에 여러 컨슈머를 생성해도 성능상 이점이 전혀 없고 코드만 복잡해진다.

**3. 올바른 확장 모델은 '1스레드 1컨슈머'이다.**

메시지 처리량을 높이는 올바른 방법은 스레드를 추가하는 것이다. 그리고 새로 생성한 스레드 각각에 새로운 컨슈머 인스턴스를 할당한다. 이렇게 구성하면 각 컨슈머가 독립적으로 안전하게 동작하며, 카프카 브로커가 파티션을 자동으로 분배하여 진정한 병렬 처리가 완성된다.









## 오프셋과 커밋

&#x20;_카프카에서는 파티션에서의 현재 위치를 업데이트 하는 작업을 오프셋 커밋 이라고 부른다._



**1. 오프셋(Offset)이란?**

오프셋은 토픽의 각 파티션 안에서 메시지가 저장된 순서 혹은 위치를 나타내는 번호표이다. 각 메시지는 파티션 내에서 고유한 번호표(오프셋)를 가지며, 이 번호는 0부터 시작하여 1씩 순차적으로 증가한다. 컨슈머는 이 오프셋을 기준으로 어디까지 메시지를 읽었는지 추적한다.

**2. 커밋(Commit)이란?**

커밋은 컨슈머가 '어디까지 메시지를 안전하게 처리했는지' 그 위치(오프셋)를 카프카 브로커에 저장하고 보고하는 행위이다. 컨슈머는 특정 오프셋을 커밋함으로써, "이 번호표까지의 메시지는 모두 처리 완료했다"고 공식적으로 알린다.



_컨슈머는 메시지를 가져와 처리한 뒤, 처리한 메시지의 오프셋을 커밋한다. 만약 컨슈머에 장애가 발생하여 재시작되더라도, 브로커에 커밋된 마지막 오프셋을 확인하고 그 다음 위치부터 메시지를 가져와 처리를 이어간다._

_이 과정을 통해 메시지의 유실이나 중복 처리를 방지하는 안정적인 시스템이 완성된다._



























