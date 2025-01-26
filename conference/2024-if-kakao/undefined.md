# 카카오페이는 어떻게 수천만 결제를 처리할까? 우아한 결제 분산락 노하우

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

## 우아한 분산락 노하우

### 일반적인 분산락

스프링 AOP 활용

* 마킹 어노테이션 (@DistributedLock)
* 분산락 Aspect
* 적용 메서드

적용 메서드에 마킹 어노테이션을 붙여주면 분산락 Aspect가 메서드 전/후로 락 획득/해제를 해주는 방식으로 동작한다.

#### 마킹 어노테이션

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface DistributedLock {
    String key();
}
```

#### Aspect 구현

```java
@Aspect
@Component
public class DistributedLockAspect {

    private final RedissonClient redissonClient;

    public DistributedLockAspect(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @Around("@annotation(distributedLock)")
    public Object handleDistributedLock(ProceedingJoinPoint joinPoint, DistributedLock distributedLock) throws Throwable {
        String key = distributedLock.key();
        RLock lock = redissonClient.getLock(key);

        if (lock.tryLock(5, 10, TimeUnit.SECONDS)) { // 락 대기 시간 5초, 락 점유 시간 10초
            try {
                return joinPoint.proceed();
            } finally {
                lock.unlock();
            }
        } else {
            throw new IllegalStateException("Unable to acquire lock");
        }
    }
}
```

#### 적용 메서드

```java
@Service
public class OrderService {

    @DistributedLock(key = "orderLock")
    public void processOrder(Long orderId) {
        // 비즈니스 로직
        System.out.println("Processing order: " + orderId);
    }
}
```

#### AOP 분산락의 불편한점

* 적용 여부 확인 **어려움**
  * aop는 프록시로 적용되므로 내부 함수 호출 또는 private 에서는 분산락 적용안됨
* key 파라미터 적용 **까다로움**
* 락 장시간 점유로 **불리한 성능**
* 기능 수정 및 리팩터링 **주의필요**

### 우아한 분산락 노하우

함수형 프로그래밍 적용

* 분산락 데코레이션 함수
* 적용 메서드

스프링 AOP와는 다르게 마킹 어노테이션이 사라지고 적용메서드에 바로 분산락 데코레이션 함수가 적용된다.

#### Lock 데코레이터

```java
public class LockDecorator {

    private final RedissonClient redissonClient;

    public LockDecorator(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    public <T> T withLock(String key, Supplier<T> action) {
        RLock lock = redissonClient.getLock(key);

        try {
            if (lock.tryLock(5, 10, TimeUnit.SECONDS)) { // 락 대기 시간 5초, 락 점유 시간 10초
                return action.get();
            } else {
                throw new IllegalStateException("Unable to acquire lock");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Thread interrupted while waiting for lock", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

#### 적용 메서드

```java
@Service
public class OrderService {

    private final LockDecorator lockDecorator;

    public OrderService(LockDecorator lockDecorator) {
        this.lockDecorator = lockDecorator;
    }

    public void processOrder(Long orderId) {
        lockDecorator.withLock("orderLock", () -> {
            // 비즈니스 로직
            System.out.println("Processing order: " + orderId);
            return null; // Supplier는 반환값을 요구하므로 null 반환
        });
    }
}
```

#### 함수형 분산락의 개선된 점

* 적용 여부 확인 **쉬움**
* key 파라미터 적용 명확함
* 락 장시간 점유로 **성능개선**
* 기능 수정 및 리팩터링 **자유로움**
