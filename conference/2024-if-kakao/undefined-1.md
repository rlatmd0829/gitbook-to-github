# 지연이체 서비스 개발기: 은행 점검 시간 끝나면 송금해 드릴게요!

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

## 지연이체 서비스 개발기

### Kafka 기반 지연이체 실행

기존에는 rabbitmq로 딜레이 메시지로 구현되어 있었지만 사내 메시지큐가 kafka로 변경되면서 kafka 기반으로 재구성

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

#### 문제 1. kafka 토픽에 같은 송금건이 쌓일 수 있다.

Kafka 토픽에 중복된 송금 데이터가 쌓이는 문제가 발생할 수 있다.&#x20;

스케줄러가 DB에서 송금 정보를 가져와 Kafka에 produce했지만, 아직 consume되지 않은 상태에서 스케줄러가 다시 실행되면 처리되지 않은 데이터를 다시 가져와 중복된 내용이 Kafka에 쌓이게 된다.

consume 에서 처리할때 상태체크를 진행해서 해결한다.

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

#### 문제 2. 컨슈머 동시 소비로 인한 중복 송금 방지

user Lock 잡기

컨슈머가 여러대가 존재하므로 동시에 가져가더라도 lock을 잡아서 동시에 실행되지 않도록 진행

<figure><img src="../../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

#### 문제 3. User Lock 영향으로 송금 실패가 늘어나겠다.

한명의 사용자가 여러 송금을 예약할 수 있으므로 현재 방식으로는 한개의 송금만 가능하고 나머지는 실패하게된다.

같은 사용자의 송금 건들은 같은 컨슈머에서 처리되도록 해야한다.

스케줄러에서 produce 할때 key 값에 userId를 넣어서 같은 partition에 쌓이도록 하여 같은 컨슈머에서 처리되도록 진행

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

#### 문제 4. 실행속도

* 컨슘하는 과정에서 1건씩 가져오고 있어서 속도가 느렸다. → 배치성으로 20건씩 가져오도록 구현
* 컨슘하고 api 처리 시 1건씩 처리해서 느렸다 → Multi thread 적용
* partition 3대, consume 2대 → 파티션 수 한번 늘리면 다시 줄이기 힘드므로 컨슘만 파티션 수 와 맞춤

<figure><img src="../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>
