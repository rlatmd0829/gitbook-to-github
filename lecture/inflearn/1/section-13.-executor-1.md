# Section 13. 스레드 풀과 Executor 프레임워크 1

## 스레드 풀과 Executor 프레임워크 1

### 스레드를 직접 사용할 때의 문제점

실무에서 스레드를 직접 생성해서 사용하면 다음과 같은 3가지 문제가 있다.

#### 1. 스레드 생성 시간으로 인한 성능 문제

예를 들어서 어떤 작업 하나를 수행할 때 마다 스레드를 각각 생성하고 실행한다면, 스레드의 생성 비용 때문에, 이미 많은 시간이 소모된다. 아주 가벼운 작업이라면, 작업의 실행 시간보다 스레드의 생성 시간이 더 오래 걸릴 수도 있다.

이런 문제를 해결하려면 생성한 스레드를 재사용하는 방법을 고려할 수 있다. 스레드를 재사용하면 처음 생성할 때를 제외하고는 생성을 위한 시간이 들지 않는다. 따라서 스레드가 아주 빠르게 작업을 수행할 수 있다.

#### 2. 스레드 관리 문제

서버의 CPU, 메모리 자원은 한정되어 있기 때문에, 스레드는 무한하게 만들 수 없다.

우리 시스템이 버틸 수 있는, 최대 스레드의 수 까지만 스레드를 생성할 수 있게 관리해야 한다.

#### 3. Runnable 인터페이스의 불편함

* 반환값이 없다: run() 메서드는 반환 값을 가지지 않는다. 따라서 실행 결과를 얻기 위해서는 별도의 메커니즘을 사용해야 한다.
* 예외 처리: run() 메서드는 체크 예외(checked exception)를 던질 수 없다. 체크 예외의 처리는 메서드 내부에서 처리해야 한다.

#### 해결

지금까지 설명한 1번, 2번 문제를 해결하려면 스레드를 생성하고 관리하는 풀(Pool)이 필요하다.

![Image](https://github.com/user-attachments/assets/620b39d8-a04f-4135-9a54-066f13a59627)

스레드를 관리하는 스레드 풀(스레드가 모여서 대기하는 수영장 풀 같은 개념)에 스레드를 미리 필요한 만큼 만들어둔다.

스레드 풀에 있는 스레드는 처리할 작업이 없다면, 대기(`WAITING` ) 상태로 관리해야 하고, 작업 요청이 오면`RUNNABLE`상태로 변경해야 한다. 막상 구현하려고 하면 생각보다 매우 복잡하다.

이런 문제를 한방에 해결해주는 것이 바로 자바가 제공하는 Executor 프레임워크다.

\


### Executor 프레임워크

자바의 Executor 프레임워크는 멀티스레딩 및 병렬 처리를 쉽게 사용할 수 있도록 돕는 기능의 모음이다. 이 프레임워크는 작업 실행의 관리 및 스레드 풀 관리를 효율적으로 처리해서 개발자가 직접 스레드를 생성하고 관리하는 복잡함을줄여준다.

#### Executor 인터페이스

```java
package java.util.concurrent;

public interface Executor {
	void execute(Runnable command);
}
```

* 가장 단순한 작업 실행 인터페이스로, `execute(Runnable command)` 메서드 하나를 가지고 있다.

#### ExecutorService 인터페이스

```java
public interface ExecutorService extends Executor, AutoCloseable {

	<T> Future<T> submit(Callable<T> task);

	@Override
	default void close(){...}
	
	...
}
```

* Executor 인터페이스를 확장해서 작업 제출과 제어 기능을 추가로 제공한다.
* 주요 메서드로는 `submit()` , `close()` 가 있다.
* Executor 프레임워크를 사용할 때는 대부분 이 인터페이스를 사용한다.

참고로 ExecutorService 인터페이스는 getPoolSize() , getActiveCount() 같은 자세한 기능은 제공하지 않는다. 이 기능은 ExecutorService 의 대표 구현체인 ThreadPoolExecutor 를 사용해야 한다.

#### Runnable의 불편함

앞서 `Runnable` 인터페이스는 다음과 같은 불편함이 있다고 설명했다.

* 반환 값이 없다
* 예외 처리

작업 스레드(`Thread-1` )는 값을 어딘가에 보관해두어야 하고, 요청 스레드(`main` )는 작업 스레드의 작업이 끝날 때까지 `join()` 을 호출해서 대기한 다음에, 어딘가에 보관된 값을 찾아서 꺼내야 한다.

작업 스레드는 간단히 값을 `return` 을 통해 반환하고, 요청 스레드는 그 반환 값을 바로 받을 수 있다면 코드가 훨씬더 간결해질 것이다.

이런 문제를 해결하기 위해 Executor 프레임워크는 `Callable` 과 `Future` 라는 인터페이스를 도입했다

\


### Future

#### Runnable과 Callable 비교

```java
package java.util.concurrent;

public interface Callable<V> {
	V call() throws Exception;
}
```

* `Callable` 의 `call()` 은 반환 타입이 제네릭 `V` 이다. 따라서 값을 반환할 수 있다.
* `throws Exception` 예외가 선언되어 있다. 따라서 해당 인터페이스를 구현하는 모든 메서드는 체크 예외인`Exception` 과 그 하위 예외를 모두 던질 수 있다.

요청 스레드가 결과를 받아야 하는 상황이라면, `Callable` 을 사용한 방식은 `Runnable` 을 사용하는 방식보다 훨씬편리하다. 코드만 보면 복잡한 멀티스레드를 사용한다는 느낌보다는, 단순한 싱글 스레드 방식으로 개발한다는 느낌이들 것이다.

#### 분석

```java
Future<Integer> future = es.submit(new MyCallable());
```

* `Future` 는 작업의 미래 결과를 받을 수 있는 객체이다.
* `submit()` 호출시 `future` 는 즉시 반환된다. 덕분에 요청 스레드는 블로킹 되지 않고, 필요한 작업을 할 수 있다.

```java
Integer result = future.get();
```

* 작업의 결과가 필요하면 `Future.get()` 을 호출하면 된다.
* Future가 완료 상태: `Future` 가 완료 상태면 `Future` 에 결과도 포함되어 있다. 이 경우 요청 스레드는 대기하지 않고, 값을 즉시 반환받을 수 있다.
* Future가 완료 상태가 아님: 작업이 아직 수행되지 않았거나 또는 수행 중이라는 뜻이다. 이때는 어쩔 수 없이요청 스레드가 결과를 받기 위해 블로킹 상태로 대기해야 한다.

#### Future가 필요한 이유

`Future` 라는 개념이 없다면 결과를 받을 때 까지 요청 스레드는 아무일도 못하고 대기해야 한다. 따라서 다른 작\
업을 동시에 수행할 수도 없다.

`Future` 라는 개념 덕분에 요청 스레드는 대기하지 않고, 다른 작업을 수행할 수 있다.

예를 들어서 다른 작업을 더 요청할 수 있다. 그리고 모든 작업 요청이 끝난 다음에, 본인이 필요할 때`Future.get()` 을 호출해서 최종결과를 받을 수 있다.

이런 이유로 ExecutorService는 결과를 직접 반환하지 않고,Future 를 반환한다.

**Future를 적절하게 잘 활용**

```java
Future<Integer> future1 = es.submit(task1); // non-blocking
Future<Integer> future2 = es.submit(task2); // non-blocking

Integer sum1 = future1.get(); // blocking, 2초 대기
Integer sum2 = future2.get(); // blocking, 즉시 반환
```

**Future를 잘못 활용한 예1**

```java
Future<Integer> future1 = es.submit(task1); // non-blocking
Integer sum1 = future1.get(); // blocking, 2초 대기
Future<Integer> future2 = es.submit(task2); // non-blocking
Integer sum2 = future2.get(); // blocking, 2초 대기
```

**Future를 잘못 활용한 예2**

```java
Integer sum1 = es.submit(task1).get(); // get()에서 블로킹
Integer sum2 = es.submit(task2).get(); // get()에서 블로킹
```

\


### CompletableFuture

Java5에 Future가 추가되면서 비동기 작업에 대한 결과값을 반환 받을 수 있게 되었다. 하지만 Future는 다음과 같은 한계점이 있었다.

* 외부에서 완료시킬 수 없고, get의 타임아웃 설정으로만 완료 가능
* 블로킹 코드(get)를 통해서만 이후의 결과를 처리할 수 있음
* 여러 작업을 조합하거나 예외 처리할 수 없음

CompletableFuture는 기존의 Future를 기반으로 외부에서 완료시킬 수 있어서 CompletableFuture라는 이름을 갖게 되었다.

즉, Future의 진화된 형태로써 외부에서 작업을 완료시킬 수 있을 뿐만 아니라 콜백 등록 및 Future 조합 등이 가능하다.

```java
CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(2000); // 작업 수행 (2초 대기)
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "Hello from CompletableFuture!";
})
// 결과가 준비되면 thenAccept() 콜백으로 처리 (논블로킹)
.thenAccept(result -> System.out.println(result));
```

* **Future:** `get()` 호출 시 결과가 준비될 때까지 블로킹되어 실행 흐름이 멈춤
* **CompletableFuture:** 콜백 체인을 사용하여 결과가 준비되면 자동으로 후속 작업이 실행되어 블로킹 없이 처리가 가능함
