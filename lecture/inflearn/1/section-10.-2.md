# Section 10. 생산자 소비자 문제2

## 생산자 소비자 문제2

생산자가 생산자를 깨우고, 소비자가 소비자를 깨우는 비효율 문제를 어떻게 해결할 수 있을까?

위에 문제가 발생하는 이유는 스레드 대기집합을 한개를 같이 쓰기 때문이다.

![Image](https://github.com/user-attachments/assets/4e937f5d-02ff-4ab6-92ba-32b53cb71654)

Lock 인터페이스와 ReentrantLock 구현체를 사용해 스레드 대기 집합을 나눠서 해결 할 수 있다.

\


### Lock Condition

synchronized, object.wait()를 사용하면 해당 객체 안에 있는 스레드 대기집합에 들어갔는데 ReentrantLock를 사용할때는 대기집합 객체를 따로 만들어서 사용한다.

```java
private final Lock lock = new ReentrantLock();
private final Condition producerCond = lock.newCondition(); // 생산자 스레드 대기 집합
private final Condition consumerCond = lock.newCondition(); // 소비자 스레드 대기 집합
private final Queue<String> queue = new ArrayDeque<>();

...
public void put(String data) {
  lock.lock();
  try {
    while(queue.size() == max) {
      log("[put] 큐가 가득 참, 생산자 대기");
      try {
        producerCond.await(); // 생산자 대기
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    }
    queue.offer(data);
    consumerCond.signal(); // 소비자 깨움
  } finally {
    lock.unlock();
  }
}
...
```

#### **condition.await()**

* Object.wait()와 유사한 기능, 지정한 condition에 현재 스레드를 대기(WAITING) 상태로 보관한다.

#### **condition.signal()**

* Object.notify()와 유사한 기능, 지정한 condition에서 대기중인 스레드를 하나 깨운다.

![Image](https://github.com/user-attachments/assets/fc491901-745b-48f1-b22a-5cc9ed7f86e1)

이 그림에서 lock은 synchronized에서 사용하는 객체 내부에 있는 모니터락이 아니라, ReentrantLock을 뜻한다.

\


### BlockingQueue

queue에 넣을때마다 condition 대기집합을 관리해주지 않아도 BlockingQueue 자료구조를 사용하면 다 구현되어 있는 내용을 사용할 수 있다.

* 데이터 추가 차단 : 큐가 가득 차면 추가 작업(`put()`)을 시도하는 스레드는 공간이 생길 때 까지 차단된다.
* 데이터 획득 차단 : 큐가 비어 있으면 획득 작업(`take()`)을 시도하는 스레드는 큐에 데이터가 들어올 때까지 차단된다.

BlockingQueue에 내부 구현을 살펴보면 ReentrantLock, Condition을 사용하는 방법과 거의 똑같이 구현되어있다.

```java
public class ArrayBlockingQueue {
  final Object[] items;
  int count;
  ReentrantLock lock;
  Condition notEmpty; //소비자 스레드가 대기하는 condition
  Condition notFull; //생산자 스레드가 대기하는 condition
  
  public void put(E e) throws InterruptedException {
      lock.lockInterruptibly();
      try {
          while (count == items.length) {
              notFull.await(); // 생산자 대기
          }
          enqueue(e);
      } finally {
          lock.unlock();
      }
  }
  private void enqueue(E e) {
      items[putIndex] = e;
      count++;
      notEmpty.signal(); // 소비자 깨움
  }
}
```

\


### BlockingQueue - 기능

큐에 넣고 꺼내는 여러 기능들을 제공해주는데 해당 기능에 차이를 알아보자.

#### **1. 예외(Throws Exception) 계열**

* `add(e)`: 큐가 여유 공간이 있으면 추가하고, 만약 **가득 차 있으면 예외**(`IllegalStateException`)를 던진다.
* `remove()`: 큐가 비어 있으면 예외(`NoSuchElementException`)를 던진다.
* `element()`: 큐가 비어 있으면 예외(`NoSuchElementException`)를 던진다.

반드시 성공해야 하는 연산 또는 비정상 상황을 명확히 구분하고 싶은 경우 등에 사용될 수 있다.

#### **2. 특수값(Special Value) 계열**

* `offer(e)`: 큐가 가득 차 있으면 `false`를 반환하고, 예외는 발생하지 않는다.
* `poll()`: 큐가 비어 있으면 `null`을 반환하고, 예외는 발생하지 않는다.
* `peek()`: 큐가 비어 있으면 `null`을 반환하고, 예외는 발생하지 않는다.

상황에 따라 성공 여부/결과를 boolean이나 null 등으로 판별하고 싶을 때 사용한다.

#### **3. 블로킹(Blocks) 계열**

* `put(e)`: 큐가 가득 차 있으면 **공간이 생길 때까지** 스레드를 블로킹한다.
* `take()`: 큐가 비어 있으면 **새로운 요소가 들어올 때까지** 스레드를 블로킹한다.

반드시 데이터를 삽입/삭제해야 하며, 대기하는 것이 허용되는 경우에 적합하다.

예를 들어, 생산자-소비자 패턴에서 생산자가 아이템을 반드시 큐에 넣어야 하거나, 소비자가 아이템을 기다렸다가 꼭 꺼내야 하는 시나리오에 사용한다.

> add, remove, offer, poll 등등은 queue 인터페이스 자체가 제공하는 기능들이고
>
> 블로킹 계열인 put, take는 `BlockingQueue` 인터페이스에만 적용된 내용이다.
