# 동시성 문제 해결하기

### 동시성 문제란?

***

동시성 문제는 여러 스레드가 공유 자원에 동시에 접근할 때 발생하는 문제입니다. 동시성 문제는 주로 경쟁 조건, 데드락, 교착상태, 세마포어 등의 현상으로 나타날 수 있습니다.

동시성 문제를 해결하는 방법은 다양한데, 주로 다음과 같은 방법을 사용합니다.

* synchronized 키워드를 사용한 동기화
* 낙관적락
* 비관적락
* 분산락

해당 내용들을 각각 어느 상황에서 사용해야 하는지에 대해서 알아보겠습니다.





### synchronized 키워드를 사용한 동기화

***

synchronized 키워드를 사용하여 메소드 또는 블록을 동기화하여 여러 스레드 간의 동시 접근을 막을 수 있습니다.

현재 데이터를 사용하고 있는 해당 스레드를 제외하고 나머지 스레드들은 데이터에 접근 할 수 없도록 막는 개념입니다.

```java
public class SynchronizedExample {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

하지만 이 방법은 같은 JVM 내에서만 유효하며, 멀티 JVM 환경에서는 다른 동시성 제어 방법이 필요합니다.

* 여러 인스턴스가 있을경우에는 동시성 못막아줌 (동일한 서비스를 여러 인스턴스로 운영하여 부하를 분산시키는 경우, 서버 인스턴스가 3개 떠있다면 프로세스 3개임)
* 여러 서버 (account, workspace 등)가 동일한 데이터베이스 레코드나 파일에 동시에 접근하여 변경을 시도할 때는 동기화가 불가능합니다.

#### synchronized vs ReentrantLock

synchronized과 ReentrantLock는 모두 자바에서 멀티스레딩 동기화를 달성하기 위한 메커니즘입니다. (ReentrantLock도 하나의 JVM 인스턴스 내에서만 작동하며, 여러 JVM 간에 락을 공유할 수 없습니다.)

* synchronized: 코드가 간결하고 가독성이 뛰어나며, JVM이 자동으로 락을 관리하므로 개발자는 명시적으로 락을 획득 및 해제할 필요가 없습니다.
* ReentrantLock: 명시적으로 락을 관리해야 하므로 코드가 복잡해질 수 있습니다. 하지만 더 세밀한 제어가 가능합니다.

일반적으로는 synchronized를 사용하는 것이 간단하고 편리하며, 성능에 큰 문제가 없는 경우가 많습니다. 하지만 고급 기능이나 특수한 상황에서 ReentrantLock을 사용할 수 있습니다.





### 낙관적락

***

낙관적락은 데이터를 읽을 때 락을 걸지 않고, 데이터를 수정할 때 락을 거는 방식입니다.

낙관적 락은 충돌이 발생할 경우 트랜잭션을 롤백하며, 충돌이 발생하는 빈도가 낮거나 충돌이 발생하더라도 롤백해도 큰 영향이 없는 경우에 사용하는 것이 좋습니다.

> **읽기 작업이 많고 쓰기 작업이 상대적으로 적은 환경.**

#### JPA에서 낙관적락 구현

낙관적락은 JPA에서 @Version 어노테이션을 활용하여 구현할 수 있습니다.

트랜잭션 내에서 엔터티를 수정할 때 버전이 증가하게 됩니다.

```java
@Entity
public class Item {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private double price;

    @Version
    private Long version; // @Version 어노테이션 추가
}
```

두 트랜잭션이 동시에 동일한 엔터티를 수정하려고 시도하면, 먼저 커밋하는 트랜잭션이 성공하고, 나중에 커밋하는 트랜잭션은 충돌로 감지되어 롤백됩니다.





### 비관적락

***

비관적락은 데이터를 읽을 때 락을 걸고 데이터를 수정할 때 락을 해제하는 방식으로 락을 획득할 때까지 대기하는 방식입니다. 따라서 락을 획득하는 시간이 길어질 수 있습니다.

락 충돌이 잘 발생하지 않는 경우에도 대기할 수 있어야 하는 환경. 즉, 충돌이 발생할 확률이 높은 경우에 사용하는 것이 좋습니다.

> **쓰기 작업이 빈번하게 발생하고 여러 트랜잭션이 동시에 동일한 엔터티를 변경할 가능성이 높은 환경.**

#### JPA에서 비관적락 구현

비관적락은 JPA에서 @Lock 어노테이션을 활용하여 구현할 수 있습니다.

```java
@Repository
public interface ItemRepository extends JpaRepository<Item, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE) // 비관적락 설정
    Optional<Item> findById(Long id);
}
```

@Lock을 사용하여 PESSIMISTIC\_WRITE 락을 설정한 경우, 해당 레코드에 대한 수정 작업을 시도하는 트랜잭션은 해당 레코드에 대한 락을 획득합니다.

이 락은 해당 트랜잭션이 끝날 때까지 해당 레코드에 대한 다른 쓰기 작업을 막습니다.

따라서 다른 트랜잭션에서 해당 레코드에 대한 조회나 수정 작업을 시도하면 락이 설정된 상태이므로 해당 트랜잭션이 완료될 때까지 대기하게 됩니다

#### LockModeType

**PESSIMISTIC\_READ**

* 해당 리소스에 공유락을 겁니다. 타 트랜잭션에서 읽기는 가능하지만 쓰기는 불가능해집니다.

**PESSIMISTIC\_WRITE**

* 해당 리소스에 베타락을 겁니다. 타 트랜잭션에서는 읽기와 쓰기 모두 불가능해집니다. (DBMS 종류에 따라 상황이 달라질 수 있습니다)

#### 단순 조회와 수정 분리하여 사용하기

조회만하고 끝나는 경우에는 락을 사용하지 않는게 좋습니다. 그러므로 조회와 수정에서 사용하는 조회문을 분리하는게 좋습니다.

```java
public interface ItemRepository extends JpaRepository<Item, Long> {
    // 수정 연산에 사용하는 메소드 (비관적 락 설정)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Item> findByIdForWrite(Long id);

    // 조회 연산에 사용하는 메소드
    Optional<Item> findByIdForRead(Long id);
}
```

#### queryDSL 사용 시

Querydsl에서는 setLockMode(LockModeType lockMode) 메서드를 지원해준다.

```java
public Optional<Item> findByIdWithPessimisticWriteLock(Long id) {
    QItem item = QItem.item;
    return Optional.ofNullable(queryFactory
            .selectFrom(item)
            .where(item.id.eq(id))
            .setLockMode(LockModeType.PESSIMISTIC_WRITE)
            .fetchOne());
}
```





### 낙관적락, 비관적락 테스트

***

낙관적락과 비관적락을 테스트하기 위해 다음과 같은 테스트 코드를 작성해보았습니다.

```java
@BeforeEach
void setUp() {
  // 테스트 전에 데이터를 미리 삽입
  Item item = Item.of("Test Item", 100.0, 100);
  itemRepository.save(item);
  }

@Test
void testConcurrentOrderAndChangePrice() throws InterruptedException {
  int numberOfThreads = 100;
  int numberOfIterations = 1;
  int expectedFinalStockQuantity = 0;
  double expectedFinalItemPrice = 50.0;
  
  ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
  CountDownLatch latch = new CountDownLatch(numberOfThreads);

  for (int i = 0; i < numberOfThreads; i++) {
    executorService.submit(() -> {
      try {
        for (int j = 0; j < numberOfIterations; j++) {
          // 주문 수행
          itemService.orderItem(1L, new OrderRequest(1));
          itemService.changeItemPrice(1L, new ChangePriceRequest(50.0));
        }
      } finally {
        latch.countDown();
      }
    });
  }

  latch.await(); // 모든 스레드의 작업이 끝날 때까지 대기
  executorService.shutdown();

  // 최종적인 재고와 가격을 확인
  Item item = itemRepository.findById(1L).orElseThrow(() -> new RuntimeException("Item not found"));
  System.out.println("Final Stock Quantity: " + item.getStockQuantity());
  System.out.println("Final Item Price: " + item.getPrice());

  // 예상 결과와 실제 결과를 비교
  assertEquals(expectedFinalStockQuantity, item.getStockQuantity());
  assertEquals(expectedFinalItemPrice, item.getPrice());
}
```

#### 낙관적락 테스트

위의 테스트 코드는 100개의 스레드가 1번씩 주문을 수행하도록 했습니다.

이때, Item 엔터티에는 @Version 어노테이션이 추가되어 있어서, 동시에 가격 변경을 시도하는 스레드는 충돌로 인해 롤백되어야 합니다.

<figure><img src="../../.gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

테스트 결과를 보면, 100개의 주문을 해서 재고가 0일것 같지만 동시에 주문을 시도하여 롤백되는 트랜잭션이 있어 재고가 0이 아닌것을 확인할 수 있습니다.

#### 비관적락 테스트

위에 낙관적락 테스트코드와 동일한 테스트코드를 사용하여 비관적락을 테스트해보았습니다.

<figure><img src="../../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

비관적락을 사용했을때는 락을 획득할때까지 대기하는 방식이기 때문에 모든 요청이 처리되어 재고가 0이 되었습니다.

#### 결론

즉, **낙관적락**은 충돌이 발생하는 빈도가 낮거나 충돌이 발생하더라도 롤백해도 큰 영향이 없는 경우에 사용하는 것이 좋고 **비관적락**은 시간이 걸리더라도 모든 요청이 처리되는것이 중요한 상황에서 사용하는것이 좋습니다.





### 분산락

***

분산 락(Distributed Lock)은 서로 다른 프로세스가 상호 배타적인 방식으로 공유 리소스와 함께 작동해야 하는 많은 환경에서 유용한 Lock입니다.

낙관적 락(Optimistic Lock)과 비관적 락(Pessimistic Lock)은 싱글 DB 환경인 경우에만 적용 가능한 개념입니다. 샤딩 또는 Replication 등을 통해 DB가 분산되어있는 환경이라면 적용할 수 없습니다.

#### 분산 서버 + 싱글 DB

* 낙관적 락(Optimistic Lock) OR 비관적 락(Pessimistic Lock) OR 분산 락(Distributed Lock) 사용 가능
* 이 경우 보통 제일 유연한 낙관적 락(Optimistic Lock) 사용

#### 분산 서버 + 분산 DB

* 분산 락(Distributed Lock) 사용 가능

즉 분산락(Distributed Lock)은 애플리케이션 서버나, DB 서버가 분산되어 있을 때 사용하는 Lock 이라고 할 수 있다.

#### Redis를 통한 분산락 구현

Redis는 분산 환경에서 락을 구현하는데 효과적으로 사용될 수 있습니다. Redis의 SETNX와 EX(expire) 명령어를 활용하여 간단하게 분산 락을 구현할 수 있습니다. 또한, Redis를 쉽게 사용하기 위해 Redisson 또는 Lettuce와 같은 라이브러리를 활용할 수 있습니다.

Redisson과 Lettuce는 둘 다 Redis 클라이언트 라이브러리이며, Redis와의 상호작용을 쉽게 만들어줍니다

#### Redisson

* Redisson은 Java에서 Redis를 위한 높은 수준의 추상화를 제공하는 클라이언트 라이브러리입니다. Redis에서 제공하는 레드락(Redlock)과 같은 분산 기능을 더 쉽게 사용할 수 있도록 제공됩니다.
* 분산 락 기능: 분산 락 구현에 있어서, Redisson은 내부적으로 SETNX 또는 SET 명령어와 만료 시간 옵션을 사용할 수 있지만, 이는 추상화된 API를 통해 숨겨져 있습니다.

#### Lettuce

* Lettuce는 비동기 및 동기식 Redis 클라이언트로, 다중 스레드 환경에서 안전하게 사용할 수 있습니다. Netty 프레임워크를 기반으로 하여 높은 성능을 제공합니다.
* 분산 락 구현: Lettuce 자체적으로 분산 락 기능을 내장하고 있지는 않지만, 개발자는 SETNX, SET과 같은 명령어를 직접 사용하여 락 로직을 구현할 수 있습니다.

#### Redisson vs Lettuce

Lettuce는 분산락 구현 시 setnx, setex과 같은 명령어를 이용해 지속적으로 Redis에게 락이 해제되었는지 요청을 보내는 스핀락 방식으로 동작합니다. 요청이 많을수록 Redis가 받는 부하는 커지게 됩니다.

이에 비해 Redisson은 Pub/Sub 방식을 이용하기에 락이 해제되면 락을 subscribe 하는 클라이언트는 락이 해제되었다는 신호를 받고 락 획득을 시도하게 됩니다.

그래서 redisson은 가장 무난한 구현 방식으로, Publish/Subscribe 기능을 이용하여 락의 재시도를 최소화할 수 있어 성능과 안정성 두 마리 토끼를 잡을 수 있습니다.

\


***

### Reference

* [https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V1-%EB%82%99%EA%B4%80%EC%A0%81-%EB%9D%BDOptimisitc-Lock-feat.%EB%8D%B0%EB%93%9C%EB%9D%BD-%EC%B2%AB-%EB%A7%8C%EB%82%A8](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V1-%EB%82%99%EA%B4%80%EC%A0%81-%EB%9D%BDOptimisitc-Lock-feat.%EB%8D%B0%EB%93%9C%EB%9D%BD-%EC%B2%AB-%EB%A7%8C%EB%82%A8)
* [https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V2-%EB%B9%84%EA%B4%80%EC%A0%81-%EB%9D%BDPessimistic-Lock](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V2-%EB%B9%84%EA%B4%80%EC%A0%81-%EB%9D%BDPessimistic-Lock)
* [https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V3-%EB%B6%84%EC%82%B0-DB-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-%EB%B6%84%EC%82%B0-%EB%9D%BDDistributed-Lock-%ED%99%9C%EC%9A%A9](https://velog.io/@znftm97/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-V3-%EB%B6%84%EC%82%B0-DB-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-%EB%B6%84%EC%82%B0-%EB%9D%BDDistributed-Lock-%ED%99%9C%EC%9A%A9)
* [https://helloworld.kurly.com/blog/distributed-redisson-lock/](https://helloworld.kurly.com/blog/distributed-redisson-lock/)
* [https://channel.io/ko/blog/distributedlock\_2022\_backend](https://channel.io/ko/blog/distributedlock_2022_backend)
