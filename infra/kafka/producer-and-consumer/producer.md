---
description: 카프카 프로듀서
---

# Producer



## 카프카에 메시지를 쓰는 과정&#x20;

> 카프카에 메시지를 쓰는 작업은 ProducerRecord 객체를 생성함으로써 시작된다.&#x20;
>
> '토픽'과 '밸류' 지정은 필수, '키'와 '파티션' 지정은 선택사항

{% stepper %}
{% step %}
### 시리얼라이저

메시지의 키와 값을 네트워크로 보낼 수 있도록 바이트 배열로 변환한다.
{% endstep %}

{% step %}
### 파티셔너

메시지 키의 해시값 등을 이용해 어떤 파티션으로 보낼지 결정한다.
{% endstep %}

{% step %}
### 레코드 배치

파티션이 결정되면 메시지를 바로 보내지 않고, 레코드들을 모은 레코드 배치에 추가 한다.
{% endstep %}

{% step %}
### 전송

별도의 스레드가 배치로 묶인 메시지를 브로커로 전송한다.
{% endstep %}

{% step %}
### 실패 및 재시도

전송이 실패하면, 설정된 횟수만큼 자동으로 재시도를 한다.
{% endstep %}

{% step %}
### 브로커 저장 및 응답

전송에 성공하면 브로커는 해당 토픽의 파티션에 메시지를 저장하고, 성공했다는 응답을 프로듀서에게 보낸다.
{% endstep %}
{% endstepper %}



## 메시지 전송 방법

* 파이어 앤 포겟(Fire and forget)

메시지를 보내기만 하고 성공 여부는 전혀 확인하지 않는 방식

예시) 이벤트 광고 알림

* 동기적 전송

&#x20;메시지 한 건 마다 브로커로부터 성공 응답이 올 때까지 기다리는 방식

* 비동기적 전송

&#x20;메시지를 보낸 후 결과를 기다리지 않고 바로 다음 작업을 처리하며, 나중에 별도의 콜백 함수를 통해 성공/실패 결과를 통지받는 방식

&#x20;예시) DLQ에 넣고 추후에 처리





## 프로듀서 설정

* acks=all: 프로듀서는 메시지가 모든 인-싱크 레플리카에 전달된 뒤에야 브로커로부터 성공했다는 응답을 받는다. 하지만, 레플리카중 하나가 죽었다면 성공응답을 받을 수 없으므로 인-싱크 레플리카의 갯수를 지정해줄 수 있다.
* `max.block.ms`: `send()`를 호출했을 때 프로듀서의 전송 버퍼가 가득 차거나 메타데이터가 아직 사용 가능하지 않을 때 블록된다. 이 상태에서 `max.block.ms`만큼 시간이 흐르면 예외가 발생한다.
* `delivery.timeout.ms`: 프로듀서가 `send()`를 호출한 시점부터 브로커로부터 최종 성공 응답(Ack)을 받기까지 허용되는 전체 시간이다. 전송 시도, 재시도를 포함한 모든 과정의 총괄 타임아웃이다.
* `linger.ms`: 레코드가 전송될 배치를 형성하기 위해 프로듀서 내부 버퍼에서 대기하는 최대 시간이다.
* `request.timeout.ms`: 프로듀서가 브로커에게 단일 요청을 보낸 후 응답을 기다리는 최대 시간이다. 이 시간이 지나면 해당 요청은 실패로 간주하고 재시도를 수행한다.

#### _d**elivery.timeout.ms 는 linger.ms와 request.timeout.ms 보다 커야한다**_

* 부등식 표현

```
delivery.timeout.ms >= linger.ms + retry.backoff.ms + request.timeout.ms
```

메시지 전송이 성공하기 위한 최소 시간은 '버퍼 대기 시간(`linger.ms`)'과 최소 '한 번의 네트워크 요청 시간(`request.timeout.ms`)'의 합이다. `delivery.timeout.ms`는 이 전체 과정을 포괄하는 상위 개념의 타임아웃이다.





## 카프카 에이브로(Avro)





## 커스텀 파티셔너

&#x20;샤드, 샤드키, 샤드 라우터?





## 쿼터, 스로틀링

&#x20;











## 스프링부트에  프로듀서 설정하기

```yaml
spring:
  application:
    name: kafka-practice

  kafka:
    bootstrap-servers: localhost:9092 # 1. 카프카 클러스터 접속 주소

    # 2. 프로듀서(Producer) 설정
    producer:
      # 메시지 Key/Value를 바이트로 변환하는 직렬화(Serializer) 클래스 지정
      key-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
```

* application.yml 파일을 위와 같이 설정해주자



```java
@Service
@RequiredArgsConstructor
public class KafkaProducerService {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private static final String TOPIC_NAME = "my-topic";

    public void sendMessage(String message) {
        System.out.println("Sending message: " + message);
        kafkaTemplate.send(TOPIC_NAME, message);
    }
}
```









