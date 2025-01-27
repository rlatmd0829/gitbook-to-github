# MMORPG 실시간 알림 서비스 개발기 (feat. Kafka Streams)

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

## MMORPG 실시간 알림 서비스 개발기

개발사가 여러 회사가 있지만 알림 발송은 퍼블리셔인 카카오 게임즈에서 진행하고 있음

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

그런데 각 개발사에서 알림을 보내는 방식이 제각각이라 알림 서비스 플랫폼을 개발하기 시작함

### How?

게임로그를 활용

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### 신규 알림 서비스 조건

* 게임별로 생성되는 분당 **80000건**의 로그를 처리 할 수 있는 시스템
* 알림이 유실되거나 중복되지 않도록 **하나의 로그에 대하여 한번의 처리**
* 게임이 늘어날 때마다 **확장 할 수 있는 시스템**

해당 조건을 충족하기 위해 Kafka Streams를 선택

### Kafka Streams

* Kafka Streams는 kafka 토픽간에 **Stream Processing** 기능을 제공하는 라이브러리
* 높은 확장서과 내결함성 지원
* 카프카의 exactly-once 메시지 전송 전략 지원

> Stream Processing은 데이터를 연속적으로 처리하는 방식을 말한다.

|       | Stream Processing | Batch Processing     |
| ----- | ----------------- | -------------------- |
| 처리 방식 | 연속된 데이터를 하나씩 처리   | 일정기간 단위로 수집하여 한번에 처리 |
| 속도    | (준)실시간            | 수분\~시간의 지연시간         |

### 왜 Kafka Streams 인가?

* Apache Flink : 알림 서비스에서 사용하기에는 너무 복잡함
* Logstash : exactly-once를 지원하지 않음

kafka streams는 하나의 라이브러리로 springboot에서 import해서 사용하고 카프카를 바라봐서 데이터를 가져온다

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

로그토픽에서 로그를 감지하여 가져오고 알림 조건으로 필터링 처리하여 알림 토픽에 넣어주는 방식으로 알림 서비스를 구현하였다.
