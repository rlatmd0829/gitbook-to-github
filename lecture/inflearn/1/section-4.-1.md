# Section 4. 스레드 제어와 생명주기 1

## 스레드 제어와 생명주기 1

### 스레드 상태

* **NEW** : 스레드가 아직 시작되지 않은 상태
* **RUNNABLE** : 스레드가 실행 중이거나 실행될 준비가 된 상태
  * runnable 상태일때만 스케줄러에 들어갈수 있다
* **BLOCKED** : 스레드가 동기화 락을 기다리는 상태
* **WAITING** : 스레드가 다른 스레드의 특정 작업이 완료되기를 기다리는 상태
* **TIMED\_WAITING** : 일정 시간 동안 기다리는 상태
* **TERMINATED** : 스레드가 실행을 마친 상태

![image](https://github.com/user-attachments/assets/bde62483-1689-4a59-81ba-542f6b91ca85)

#### 자바 스레드의 상태 전이 과정

1. **New → Runnable** : start() 메서드를 호출하면 스레드가 Runnable 상태로 전이된다.
2. **Runnable → Blocked/Waiting/Timed Waiting :** 스레드가 락을 얻지 못하거나, wait() 또는 sleep() 메서드를 호출할 때 해당 상태로 전이된다.
3. **Blocked/Waiting/Timed Waiting → Runnable :** 스레드가 락을 얻거나, 기다림이 완료되면 다시 Runnable 상태로 돌아간다.
4. **Runnable → Terminated** : 스레드의 run() 메서드가 완료되면 스레드는 Terminated 상태가 된다.

### 체크예외 재정의

자바에서 메서드를 재정의할때, 재정의 메서드가 지켜야할 예외와 관련된 규칙이 있다.

#### **체크예외**

* 부모 메서드가 체크 예외를 던지지 않는 경우, 재정의된 자식 메서드도 체크 예외를 던질 수 없다.
* 자식 메서드는 부모 메서드가 던질 수 있는 체크 예외의 하위 타입만 던질 수 있다.

```java
class Parent {
    void method() throws InterruptedException {
        // ...
    }
}

class Child extends Parent {
    @Override
    void method() throws Exception {
        // ...
    }
}

public class Test {
    public static void main(String[] args) {
        Parent p = new Child();
        try {
            p.method();
        } catch (InterruptedException e) {
            // InterruptedException 처리
        }
    }
}
```

컴파일 타임에는 Parent p에 method를 호출한다고 생각해서 `InterruptedException` 을 잡게 코드를 짜지만

막상 런타임때는 child에 method를 호출하여 `InterruptedException` 보다 상위 에러가 올 수 있기 때문에 부모 메서드가 던질 수 있는 체크 예외의 하위 타입만 던질 수 있게 만들었다.

#### **언체크예외**

* 예외 처리를 강제하지 않으므로 상관없이 던질 수 있다.

### this의 비밀

인스턴스의 메서드를 호출하면, 어떤 인스턴스의 메서드를 호출했는지 기억하기 위해, 해당 인스턴스의 참조값을 스택 프레임 내부에 저장해둔다. 이것이 바로 우리가 자주 사용하던 this이다.

특정 메서드 안에서 this를 호출하면 바로 스택프레임 안에 있는 this 값을 불러서 참조값을 확인해 힙영역에 있는 인스턴스 중에 어떤 인스턴스인지 확인해 해당 값을 가져오는거다.

![image](https://github.com/user-attachments/assets/8faa67c1-29d4-42b9-a009-86446549a38e)

### Join - 사용

main 스레드에서 다음 코드를 실행하게 되면 main 스레드는 thread-1, thread-2가 종료될 때 까지 기다린다. 이때 main 스레드는 `WAITING` 상태가 된다.

```java
thread1.join();
thread2.join();
```

* Join()을 호출하는 스레드는 대상 스레드가 `TERMINATED` 상태가 될 때 까지 대기한다. 대상 스레드가 `TERMINATED` 상태가 되면 호출 스레드는 다시 `RUNNABLE` 상태가 되면서 다음 코드를 수행한다.
* 하지만 join의 단점은 다른 스레드가 완료될 때 까지 무기한 기다리는 단점이 있다.
  * join에 특정 시간동안만 기다리도록 설정할 수 있고 특정 시간이 지나면 저절로 `RUNNABLE` 상태가 된다.
  * join에 특정 시간을 넣고 기다릴때 상태는 `WAITING`이 아니라 `TIMED_WAITING` 상태가 된다.
