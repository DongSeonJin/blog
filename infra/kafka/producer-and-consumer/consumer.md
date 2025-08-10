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
