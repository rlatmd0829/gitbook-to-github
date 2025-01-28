# taxi-meter 개발기

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

## Taxi Meter 소개

미터란?

택시 요금을 계산해주는 시스템

미터의 종류

* 오프라인 미터 : 중형 택시
* 앱 미터 : 블랙, 벤티

앱 미터의 Taxi Meter 역할

* GPS 기반 실시간 요금 계산
* 승객과 기사에게 요금 노출

### 개선 사항

#### 1. Redis Network I/O 개선

주행 데이터를 일정시간 마다 Redis에서 조회하여 메모리 사용량 2GB에 비해 높은 Network I/O 500Mbps 를 사용하고 있음

주행 데이터를 각 인스턴스에서 로컬캐시로 관리하고 Redis에서는 어떤 주행이 어떤 인스턴스에 있는지만 저장하여 한번 주행이 시작하면 하나의 인스턴스로만 값이 들어가도록 구현함

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

위의 결과로 500Mbps → 8Mbps로 98.4%를 감소 시킴

#### 2. 맵 매치 최적화

오프라인 미터기가 있는 경우 마지막에 맞는지 검증하는 용도로만 사용하고 있어서 주행 종료 후 전체 경로 한 번에 맵매치 하도록 개선

#### 3. 도로 정보 라이브 업데이트

기존에는 도로 정보가 업데이트가 되면 배포를 진행했지만 운영 배포 리소스 절감을 위해 라이브 업데이트를 도입하게됨

* Spring Cloud Config 활용
* 각각 인스턴스에서 비동기로 도로 네트워크 다운로드 및 그래프 생성 후 버전 변경 수행

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

#### 정리

<figure><img src="../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>
