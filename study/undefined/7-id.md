# 7장 분산 시스템을 위한 유일ID 생성기 설계

## 분산 시스템을 위한 유일ID 생성기 설계

분산시스템에서 auto increment를 사용하면 문제가 발생한다. 이는 여러 서버에서 동시에 auto increment를 사용할 때, ID가 중복되는 문제가 발생하기 때문이다.

### 문제 이해 및 설계 범위 확정

요구사항을 정리하면 아래와 같다.

* ID는 유일해야한다.
* ID는 숫자로만 구성되어야 한다.
* ID는 64비트로 표현될 수 있는 값이어야 한다.
* ID는 발급 날짜에 따라 정렬 가능해야 한다.
* 초당 10,000개의 ID를 만들 수 있어야 한다.

### 설계안

분산 시스템에서 유일성이 보장되는 ID를 만드는 방법은 여러가지다.

* 다중 마스터 복제
* UUID
* 티켓 서버
* 트위터 스노플레이크(twitter snowflake) 접근법

#### 다중 마스터 복제

이 접근법은 데이터베이스의 auto\_increment 기능을 활용하는 것이다.

다만 다음 ID의 값을 구할 때 1만큼 증가시켜 얻는 것이 아니라, 현재 사용중인 데이터베이스의 서버 수 k만큼 증가 시킨다.

단점으로는 ID의 유일설은 보장되겠지만 그 값이 시간 흐름에 맞추어 커지도록 보장할 수 없고

서버를 추가하거나 삭제할 때도 잘 동작하도록 만들기 어렵다.

#### UUID

UUID는 유일성이 보장되는 ID를 만드는 또 하나의 간단한 방법이이고 UUID는 128비트의 숫자로 되어있다.

ID가 128비트로 길다. 요구사항은 64비트이다. 또한 숫자로만 구성되어 있지 않으며, 발급 날짜에 따라 정렬 가능하지 않다.

따라서 해당 요구사항을 만족하지 않는다.

#### 티켓 서버

티켓서버는 auto\_increment 기능을 갖춘 데이터베이스 서버, 즉 티켓 서버를 중앙 집중형으로 하나만 사용하는 것이다.

이 방법은 유일성이 보장되고, 숫자로만 구성된, 64비트로 표현가능한, 발급 날짜에 따라 정렬 가능한 ID를 생성할 수 있다. 구현도 간단하다.

하지만, 단점으로는 **티켓 서버 자체가 SPOF**(Single Point of Failure, 단일 장애점)이 되는 것이다.

이 이슈를 피하기 위해 **티켓 서버를 다중화**해버리면, 데이터 **동기화 같은 새로운 문제**가 발생한다.

#### 트위터 스노플레이크(twitter snowflake) 접근법

지금까지 여러 ID 생성기 구현을 살펴봤지만, 트위터 스노플레이크는 가장 널리 사용되는 방법이다.

트위터 스노플레이크는 64비트의 숫자로 구성된 ID를 만드는 방법이다. 아래는 스노플레이크의 구조이다.

![image](https://github.com/user-attachments/assets/a4399da4-65af-4976-9821-3bd6a6f4f650)

* **사인(sign) 비트** : 1비트를 할당한다. 현재 사용되지 않는다. 나중을 위해 확보해둔 비트로, 음수/양수를 구분하는데 사용할 수 있다.
* **타임스탬프(timestamp)**: 41비트를 할당한다. 기원 시각(epoch) 이후로 몇 밀리초가 경과했는지 나타내는값이다.
* **데이터센터 ID**: 5비트를 할당한다. 2^5, 즉 32개의 데이터 센터를 사용할 수 있다.
* **서버 ID**: 5비트를 할당한다. 2^5, 즉 데이터센터별 32개의 서버를 사용할 수 있다.
* **일련번호**: 12비트를 할당한다. 각 서버는 ID를 생성할 때 마다 이 일련번호를 1씩 증가시킨다. 이 값은 1ms가 경과할 때 마다 0으로 초기화 된다.

스노우플레이크는 타임스탬프 값으로 시간순 정렬을 보장하며, 데이터 센터 ID와 서버 ID를 통해 서로 다른 서버에서 같은 시각에 생성된 ID의 충돌을 방지한다.

또한, 일련번호를 활용하여 같은 서버에서 동일 밀리초에 생성된 ID의 중복을 방지하며, 서버 간 동기화가 필요 없어 확장성이 뛰어나다.