# Section 7. 동기화 - synchronized

## 동기화 - synchronized

volatile 키워드로 동시성 문제를 해결할 수 는 없다

volatile은 한 스레드가 값을 변경했을 때 다른 스레드에서 변경된 값을 즉시 볼 수 있게 하는 메모리 가시성 문제를 해결할 뿐이다.

그래서 스레드A가 잔액 1000원 읽어가고 동시에 스레드B가 잔액 1000원 읽어가는걸 막지 못한다.

### 임계영역

임계영역(critical section)은 여러 스레드가 접근하면 데이터 불일치나 예상치 못한 동작이 발생할 수 있는 위험한 코드 부분을 뜻한다.

동시성 문제가 발생하는 근본적인 원인은 여러 스레드가 함께 사용하는 공유 자원을 여러 단계로 나누어 사용하기 때문이다.

```java
출금() {
  1. 검증 단계 : 잔액 확인
  2. 출금 단계 : 잔액 감소
}
```

위에 `출금()` 로직이 바로 임계영역이다. 여기서 잔액은 여러 스레드가 동시에 접근해서는 안되는 공유 자원이다.

검증단계에서 검증한 값이 출금단계 까지 그대로 유지가 되어야 한다. 그런데 이때 다른 스레드가 잔액의 값을 변경하면 동시성 문제가 발생하는거다.

이런 임계 영역은 한 번에 하나의 스레드만 접근할 수 있도록 안전하게 보호해야 한다.

\


### synchronized 메서드

자바의 synchronized 키워드를 사용하면 한번에 하나의 스레드만 실행할 수 있는 코드 구간을 만들 수 있다.

t1이 락을 획득하면 t2 스레드는 `BLOCKED` 상태로 대기한다.

![Image](https://github.com/user-attachments/assets/dc155422-e74d-4803-aa4c-47565544d20b)

모든 객체(인스턴스)는 내부에 자신만의 락을 가지고 있다.

* 모니터 락 이라고 부른다
* 스레드가 synchronized 키워드가 있는 메서드에 진입하려면 반드시 해당 인스턴스의 락이 있어야 한다.

> 락을 획득하는 순서는 보장되지 않는다.
>
> 해당 인스턴스의 락을 기다리는 수 많은 스레드 중에 하나의 스레드만 락을 획득한다.
>
> 이때 어떤 순서로 락을 획득하는지는 자바 표준에 정의되어 있지 않다. 따라서 순서를 보장하지 않고, 환경에 따라서 순서가 달라질 수 있다.

synchronized는 JMM(Java Memory Model)에 의해 다음과 같은 규칙을 따라서 굳이 메모리 가시성문제를 해결하기 위해 volatile 키워드 안써도 된다.

* 한 스레드가 **`synchronized`** 블록에 진입할 때, 다른 스레드가 메인 메모리에 반영한 최신 값을 가져오도록 보장하여 스레드가 항상 최신 상태의 데이터를 읽도록 한다.
* 스레드가 **`synchronized`** 블록을 빠져나갈 때, 블록 내에서 변경된 모든 변수를 메인 메모리에 플러시 하여 다른 스레드가 즉시 최신 값을 볼 수 있도록 한다.

```java
Thread A: synchronized 블록 진입  ---->  메인 메모리에서 최신 값을 가져옴
Thread A: 블록 내부 작업 수행  ---->  변경 내용을 메인 메모리에 기록
Thread B: synchronized 블록 진입  ---->  메인 메모리의 최신 값 가져옴
```

\


### 동시성 문제 - 지역변수

```java
static class MyCounter {
  public void count() {
    int localValue = 0;
    for (int i = 0; i < 1000; i++) {
      localValue = localValue + 1;
    }
  }
}
```

지역변수는 각 스레드마다 가지는 스택영역에 생성되므로 이 메모리 공간은 다른 스레드와 공유하지 않는다.

따라서 지역변수는 동시성 문제를 고려하지 않아도 된다.

![Image](https://github.com/user-attachments/assets/243c172f-2c6d-4110-b7ad-9b93d90e2c3e)\


### 동시성 문제 - final 필드

```java
class Immutable {
  private final int value;
  
  public Imutable(int value) {
    this.value = value;
  }
  
  public int getValue() {
    return value;
  }
}
```

value 필드에 final 키워드가 붙었을때 동시성 문제가 발생할까?

여러 스레드가 공유 자원에 접근하는 것 자체는 문제가 되지않고, 여러 스레드가 사용중에 값을 변경하는게 문제가 된다.

final이 붙으면 어떤 스레드도 값을 변경할수 없으므로 멀티스레드 상황에 문제 없는 안전한 공유 자원이 된다.
