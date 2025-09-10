---
description: 카프카 컨슈머
---

# Consumer



카프카에서 데이터를 읽는 애플리케이션은 토픽을 구독하고 구독한 토픽들로부터 메시지를 받기 위해 KafkaConsumer를 사용한다. 카프카에서 데이터를 읽는 것은 다른 메시지 전달 시스템에서 데이터를 읽는 것과는 조금 다르다.





## 컨슈머와 컨슈머 그룹

> 단일 컨슈머가 토픽을 구독하고 메시지를 받기 시작한 뒤 받은 메시지를 받아 검사하고 결과를 쓴다고 생각해보자. 어떤 문제가 있을까? 만약 프로듀서가 애플리케이션이 검사할 수 있는 속도보다 더 빠른 속도로 토픽에 메시지를 쓰기 시작한다면, 새로 추가되는 메시지의 속도를 따라잡지 못하고 결국에는 메시지 처리가 뒤로 밀릴 것이다.
>
> 하나의 토픽이 다수의 파티션으로 나뉘어 있을 경우, 컨슈머 그룹 내의 컨슈머들은 각기 다른 파티션을 할당받아 병렬로 메시지를 처리한다. 이를 통해 소비 처리량을 수평적으로 확장할 수 있다.

&#x20;컨슈머의 갯수가 파티션 갯수를 넘을 필요는 없다.





## Spring Boot 컨슈머 설정

```yaml
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
>
> (deprecated 예정)

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

> 순수 Java Kafka 클라이언트를 사용하는 환경의 경우는 위 코드처럼 subscribe 함수를 사용하여 토픽을 구독할 수 있고, poll 함수를 실행하여 할당된 파티션으로부터 메시지를 가져온다. 하지만, Spring Boot 환경에서는 `@KafkaListner` 어노테이션을 사용하여 subscribe()과 poll() 실행을 자동으로 처리해준다.







## 스레드 안전성

'하나의 스레드에는 반드시 하나의 컨슈머만 사용한다'는 원칙을 지켜야 한다.

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







### :heavy\_check\_mark:컨슈머의 커밋 방식

#### &#x20;1. 동기적 커밋 (Synchronous Commit)

동기적 커밋은 컨슈머가 `commitSync()`를 호출하면, 카프카 브로커로부터 커밋 성공 응답을 받을 때까지 기다리는(block) 방식이다.

* 특징: 확실하게 커밋이 완료된 것을 보장하기 때문에 안정성이 매우 높다. 만약 커밋에 실패하면 성공할 때까지 재시도할 수 있다.
* 단점: 응답을 기다리는 시간 동안 메시지 처리가 중단되므로 처리량이 감소한다.

***

#### 2. 비동기적 커밋 (Asynchronous Commit)

비동기적 커밋은 컨슈머가 `commitAsync()`를 호출한 뒤, 브로커의 응답을 기다리지 않고 즉시 다음 메시지 처리를 시작하는 방식이다.

* 특징: 기다리는 시간이 없어 처리량이 매우 높다.
* 단점: 커밋 요청이 실패하더라도 컨슈머는 이를 즉시 알지 못한다. 실패 시 재시도를 하더라도, 더 나중에 보낸 커밋 요청이 먼저 성공하면 순서가 꼬여 메시지가 중복 처리될 위험이 있다.

***

#### 3. 동기 + 비동기 혼합 커밋 (Hybrid Commit)

동기 커밋과 비동기 커밋을 함께 사용하는 것은 두 방식의 장점을 모두 취하는 가장 실용적인 방법이다.

* 평소에는 비동기 커밋: 메시지를 처리하는 동안에는 속도를 위해 `commitAsync()`를 사용하여 주기적으로 커밋한다. 이는 커밋 실패 시 약간의 중복 처리 위험을 감수하는 대신 높은 처리량을 유지하게 해준다.
* 마지막에는 동기 커밋: 컨슈머를 안전하게 종료하거나 리밸런싱이 발생하기 직전, 마지막으로 처리한 오프셋은 반드시 성공해야만 한다. 이때 `commitSync()`를 호출하여 다음 컨슈머가 메시지를 중복 처리하거나 유실하지 않도록 확실하게 보장한다.

***

#### 4. 특정 오프셋 커밋 (Manual Offset Commit)

`commitSync()`나 `commitAsync()` 메서드에 정확한 토픽, 파티션, 그리고 오프셋 정보를 담은 `Map`을 전달하여 커밋하는 방식이다.

일반적인 커밋은 컨슈머가 `poll()` 메서드로 가져온 메시지 배치의 마지막 오프셋을 기준으로 동작한다. 하지만 때로는 가져온 메시지를 여러 개로 나누어 처리하고, 각 묶음을 처리한 직후에 중간 오프셋을 커밋하고 싶을 때가 있다.

이때, 커밋하려는 '다음 메시지의 오프셋' 을 `OffsetAndMetadata` 객체로 만들어 정확히 지정한다. 이 방식은 매우 긴 배치 작업을 수행할 때 중간 저장지점을 만드는 것처럼 활용된다. 이를 통해 장애 발생 시 재처리해야 하는 메시지의 양을 최소화 할 수 있다.









## 리밸런스 리스너

_카프카  리밸런스 리스너(Rebalance Listener)는 컨슈머 그룹 내에서 파티션의 소유권이 변경되는 리밸런스 과정에 개입하여, 특정 동작을 수행하게 만드는 콜백 인터페이스이다._

&#xC774;_&#xB97C; 통해 컨슈머는 파티션을 넘겨주기 직전이나 새로 할당받은 직후에 안전한 마무리 작업이나 초기화 작업을 수행할 수 있다._

#### 리밸런스 리스너는 주로 두 가지 상황에 호출되는 메서드를 정의한다.

1. `onPartitionsRevoked(Collection<TopicPartition> partitions)`
   * 호출 시점: 리밸런스가 시작되기 직전, 즉 컨슈머가 기존에 할당받았던 파티션의 소유권을 잃기 바로 전에 호출된다.
   * 주요 용도: 지금까지 처리한 내용을 안전하게 저장하기 위해 마지막 오프셋을 동기적으로 커밋하는 데 사용된다. 이 작업을 하지 않으면, 해당 파티션을 새로 넘겨받은 컨슈머가 메시지를 중복 처리할 수 있다.
2. `onPartitionsAssigned(Collection<TopicPartition> partitions)`
   * 호출 시점: 리밸런스가 끝난 후, 컨슈머에게 새로운 파티션이 할당된 직후에 호출된다.
   * 주요 용도: 새로 할당받은 파티션의 오프셋 위치를 확인하거나, 처리 상태를 초기화하는 등 메시지 처리를 시작하기 전 필요한 준비 작업을 수행한다.
3. `onPartitionsLost(Collection<TopicPartition> partitions)`는 컨슈머가 브로커와의 통신 문제나 갑작스러운 종료 등으로 인해 파티션의 소유권을 비정상적으로 상실했을 때 호출되는 메서드이다.
   * `onPartitionsRevoked`와의 차이점: `onPartitionsRevoked`는 컨슈머 그룹이 정상적으로 리밸런스를 시작할 때, 즉 예측 가능한 상황에서 호출된다. 반면, `onPartitionsLost`는 컨슈머의 세션 타임아웃 초과 등 예측 불가능한 원인으로 파티션이 해제되었을 때 호출된다.
   * 주요 용도: 이 메서드가 호출되었다는 것은 마지막 오프셋을 커밋할 기회를 놓쳤을 가능성이 높다는 의미이다. 따라서 여기서는 최종 커밋을 시도하기보다는, 정리 작업(clean-up)이나 상태 초기화, 또는 비정상 유실 상황에 대한 로그 기록 및 모니터링 경고를 보내는 용도로 주로 사용된다.













## 폴링 루프를 벗어나는 방법

&#x20;순수 자바 환경에서는 무한 루프에서 폴링을 수행할 때 안전하게 종료하기 위해서는 `consumer.wakeup()` 을 호출하여 컨슈머를 안전하게 종료시켜야 한다.

`wakeup()` 메서드는 다른 스레드에서 `poll()` 메서드를 호출하여 대기(blocking) 중인 컨슈머를 즉시 깨울 때 사용한다.&#x20;



### ShutdownHook 예시 :arrow\_down:   &#x20;

```java
// 프로그램이 종료될 때 이 스레드가 실행된다.
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Starting exit...");
    // 컨슈머의 wakeup() 메서드를 호출하여 안전하게 종료시킨다.
    worker.shutdown();
    try {
        // 컨슈머 스레드가 완전히 종료될 때까지 기다린린다.
        consumerThread.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("Application has been shut down.");
}));
```



```java
@Override
public void run() {
    try {
        // 지속적으로 데이터를 폴링한다.
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Topic: %s, Partition: %d, Offset: %d, Key: %s, Value: %s%n",
                        record.topic(), record.partition(), record.offset(), record.key(), record.value());
            }
        }
    } catch (WakeupException e) {
        // wakeup()이 호출되면 poll() 메서드는 이 예외를 던진다.
        // 정상적인 종료 신호이므로 무시하고 루프를 빠져나간다.
        System.out.println("WakeupException! Consumer will be shut down.");
    } finally {
        // 컨슈머 리소스를 안전하게 닫는다.
        consumer.close();
        System.out.println("Consumer is now gracefully closed.");
    }
}
```

* 사용자가 Ctrl + C 를 누르거나, 프로그램이 종료되어 ShutdownHook이 동작한다.
* wakeup() 이 호출되고, consumer.poll()은 wakeup() 호출을 감지하는 즉시, `WakeupException`을 던지며 즉시 깨어난다.
* `finally` 블록에서 `consumer.close()`를 호출하여 리소스를 깔끔하게 정리하고 스레드를 종료한다.



> :bulb: Spring Boot 환경에서는 `@KafkaListener`를 사용하면 대부분의 경우 `wakeup()`을 직접 호출할 필요가 없다. 스프링이 애플리케이션 종료 시 자동으로 "우아한 종료(Graceful Shutdown)"를 처리해주기 때문이다.
>
> 물론, 관리자 API를 통해 특정 리스너를 동적으로 중지시키거나 재시작해야 하는 특별한 경우도 있다. 이럴 때는 `KafkaListenerEndpointRegistry`를 사용하여 리스너 컨테이너를 직접 제어할 수 있다.&#x20;

### Spring Boot에서 활용 예시 :arrow\_down:

```java
// 특정 리스너를 중지시키는 메서드
public void stopListener() {
    System.out.println("Stopping the Kafka listener...");
    // "my-specific-listener" ID를 가진 리스너 컨테이너를 중지시킵니다.
    registry.getListenerContainer("my-specific-listener").stop();
    System.out.println("Listener stopped.");
}
```

* `KafkaListenerEndpointRegistry`를 주입받아 `id`로 특정 리스너 컨테이너를 가져와 `stop()` 메서드를 호출한다. 이 `stop()` 메서드가 `wakeup()`을 호출하여 안전하게 리스너를 중지시킨다.



























