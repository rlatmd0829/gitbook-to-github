# Section 12. 동시성 컬렉션

## 동시성 컬렉션

**컬렉션 프레임워크 대부분은 스레드 세이프 하지않다.**

컬렉션에 데이터를 추가하는 `add()` 메서드를 생각해보면, 단순히 컬렉션에 데이터를 하나 추가하는 것처럼 보이지만 그 내부에서는 수 많은 연산들이 함께 사용된다.

배열에 데이터를 추가하고, 사이즈를 변경하고, 배열을 새로 만들어서 배열의 크기도 늘린다.

멀티스레드 상황에서 여러 스레드가 동시에 컬렉션 에 접근하는 경우라면 `java.util` 패키지가 제공하는 일반적인 컬렉션들은 사용하면 안된다.

### 프록시 도입

여러 스레드가 접근해야 한다면 synchronized , Lock 등을 통해 안전한 임계 영역을 적절히 만들면 문제를 해결할 수 있다.

그러나 이런 방식은 코드를 수정해야 하기 때문에 번거롭다.

`ArrayList` , `LinkedList` , `HashSet` , `HashMap` 등의 코드도 모두 복사해서 `synchronized` 기능을 추가한 코드를 만들기 어렵다.

이럴때 사용하는 것이 바로 프록시이다.

![Image](https://github.com/user-attachments/assets/3ac54b75-00e5-49f9-a0a8-7ea03513fd29)

원본 코드인 BasicList 를 전혀 손대지 않고, 프록시인 SyncProxyList 를 통해 동기화 기능을 적용했다는 점이다.

> 참고 : `Vector` 클래스는 지금의 `ArrayList` 와 같은 기능을 제공하는데, 메서드에 `synchronized` 를 통한 동기화가 되어 있다.
>
> 쉽게 이야기해서 동기화된 `ArrayList` 이다.
>
> 그러나 이에 따라 단일 스레드 환경에서도 불필요한 동기화로 성능이 저하되는 문제가 있어 지금은 거의 사용을 안하고 있다.

Collections 가 제공하는 동기화 프록시 기능 덕분에 스레드 안전하지 않은 수 많은 컬렉션들을 매우 편리하게 스레 드 안전한 컬렉션으로 변경해서 사용할 수 있다.

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

하지만 `synchronized` 프록시를 사용하는 방식은 다음과 같은 단점이 있다.

#### **synchronized 프록시 방식의 단점**

* 성능 저하 → 메서드 호출마다 동기화 비용이 발생
* 잠금 범위가 넓음 → 컬렉션 전체를 잠가 병렬 처리 효율 저하
* 정교한 동기화 불가 → 특정 부분만 동기화하기 어려워 과도한 락 발생

자바는 이런 단점을 보완하기 위해 `java.util.concurrent` 패키지에 동시성 컬렉션(concurrent collection)을 제공한다.

\


### java.util.concurrent 패키지

`java.util.concurrent` 패키지에는 고성능 멀티스레드 환경을 지원하는 다양한 동시성 컬렉션 클래스들을 제공한다.

예를 들어, `ConcurrentHashMap` , `CopyOnWriteArrayList` , `BlockingQueue` 등이 있다.

이 컬렉션들은 더 정교한 잠금 메커니즘을 사용하여 동시 접근을 효율적으로 처리하며, 필요한 경우 일부 메서드에 대해서만 동기화를 적용하는 등 유연한 동기화 전략을 제공한다.

> 참고로 `LinkedHashSet` , `LinkedHashMap` 처럼 입력 순서를 유지하는 동시에 멀티스레드 환경에서 사용할 수 있는 `Set` , `Map` 구현체는 제공하지 않는다.
>
> 필요하다면 `Collections.synchronizedXxx()` 를 사용해야 한다.
