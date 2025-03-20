# Section 8. 고급 동기화 - concurrent.Lock

## 고급 동기화 - concurrent.Lock

**synchronized 장점**

* 프로그래밍 언어에 문법으로 제공
* 자동 잠금 해제 : synchronized 메서드가 끝나면 락을 자동 해제한다.

**synchronized 단점**

* 무한 대기 : BLOCKED 상태의 스레드는 락이 풀릴 때 까지 무한 대기한다.
  * 특정 시간까지만 대기하는 타임아웃 X
  * 중간에 인터럽트 X
* 공정성 : 대기중인 스레드 중에 어떤 스레드가 락을 획득할 지 알 수 없다.

결국 더 유연하고, 더 세밀한 제어가 가능한 방법을 위해 자바 1.5부터 `java.util.concurrent` 라는 동시성 문제 해결을 위한 패키지가 추가되었다.

해당 라이브러리에는 수 많은 클래스가 있지만, 먼저 가장 기본이 되는 LockSupport에 대해서 알아보자

### LockSupport 기능

LockSupport는 스레드를 `BLOCKED`가 아닌 `WAITING` 상태로 변경한다

LockSupport를 사용하면 synchronized의 가장 큰 단점인 무한 대기 문제를 해결할 수 있다.

![Image](https://github.com/user-attachments/assets/83c0d23c-917e-4671-b958-40f4773ae865)

* park() : 스레드를 `WAITING` 상태로 변경한다.
* parkNanos(nanos) : 스레드를 나노초 동안만 `TIMED_WAITING` 상태로 변경한다.
* unpark(thread) : `WAITING` 상태의 대상 스레드를 `RUNNABLE` 상태로 변경한다.
  * 해당 스레드가 CPU를 점유하지 못하고 실행 흐름이 정지된 상태이기 때문 다른 스레드가 호출해줘야 해서 매개변수를 받는다.

\


### BLOCKED vs WAITING

**인터럽트**

* `BLOCKED` 상태는 인터럽트가 걸려도 대기 상태를 빠져나오지 못한다.
  * `BLOCKED` 상태는 자바의 synchronized 에서 락을 획득하기 위해 대기할 때 사용한다.
* `WAITING`, `TIMED_WAITING` 상태는 인터럽트가 걸리면 대기 상태를 빠져나온다. 그래서 `RUNNABLE` 상태로 변한다.

\


### ReentrantLock

LockSupport를 사용해서 park, unpark로 직접 스레드를 제어해서 synchronized를 대신 구현하기에는 너무 저수준이다.

synchronized처럼 더 고수준의 기능이 필요하다.

ReentrantLock은 LockSupport를 활용해서 synchronized에 단점을 극복하면서도 매우 편리하게 임계 영역을 다룰 수 있는 다양한 기능을 제공한다.

자바 1.5부터 Lock 인터페이스와 ReentrantLock 구현체를 제공한다.

```java
public interface Lock {
  void lock();
  void lockInterruptibly() throws InterruptedException;
  boolean tryLock();
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  void unlock();
  Condition newCondition();
}
```

#### **void lock()**

* 락을 획득한다. 만약 다른 스레드가 이미 락을 획득했다면, 락이 풀릴때까지 현재 스레드는 `WAITING` 상태가 된다.
  * 이 메서드는 인터럽트에 응답하지 않는다 → lock() 메서드는 내부에서 InterruptedException을 던지도록 구현되지 않고 무시하도록 되어있다.

> 여기서 사용하는 락은 객체 내부에 있는 모니터 락이 아니고 Lock 인터페이스와 ReentrantLock이 제공하는 기능이다.
>
> 모니터 락과 `BLOCKED` 상태는 synchronized에서만 사용된다.

#### **boolean tryLock()**

* 락 획득을 시도하고, 즉시 성공 여부를 반환한다. 만약 다른 스레드가 이미 락을 획득했다면 false를 반환하고, 그렇지 않으면 락을 획득하고 true를 반환한다.

#### **void unlock()**

* 락을 해제한다. 락을 해제하면 락 획득을 대기 중인 스레드 중 하나가 락을 획득할 수 있게 된다.
* 락을 획득한 스레드가 호출해야 하며, 그렇지 않으면 에러가 발생한다.

그리고 또 ReentrantLock은 스레드가 락을 공정하게 얻을 수 있는 모드를 제공한다.

```java
// 비공정 모드 락
private final Lock nonFairLock = new ReentrantLock();
// 공정 모드 락
private final Lock fairLock = new ReentrantLock(true);
```

스레드를 공정하게 분배하기 위해서 성능이 저하될 수 있다.
