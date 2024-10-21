# 3장 JDK의 함수형 인터페이스

## 네 가지 함수형 인터페이스

### Function

하나의 입력과 출력을 가진 전통적인 함수를 의미한다.

```java
@FunctionalInterface
public interface Function<T,R>{
  R apply(T t);
}

Function<String, Integer> stringLength = str -> str != null ? str.length() : 0;
Integer result = stringLength.apply("Hello, Function!");
```

### Consumer

이름에서 알 수 있는 것처럼 Consumer(소비)는 입력 파라미터를 소비하지만 아무것도 반환하지 않는다.

```java
@FunctionalInterface
public interface Consumer<T>{
  void accept(T t);
}

Consumer<String> println = str -> System.out.println(str);
println.accept("Hello, Consumer!");
```

### Supplier

Supplier는 Consumer의 반대로, 어떠한 입력 파라미터도 받지 않지만, T 타입의 단일값을 반환한다.

```java
@FunctionalInterface
public interface Supplier<T>{
  T get();
}

Supplier<Double> random = () -> Math.random();
Double result = random.get();
```

### Predicate

Predicate는 단일 인수를 받아 그것을 로직에 따라 테스트하고 true 또는 false를 반환하는 함수이다.

```java
@FunctionalInterface
public interface Predicate<T>{
  boolean test(T t);
}

Predicate<Integer> over9000 = i -> i > 9_000;
boolean result = over9000.test(12_345);
```

## 함수형 인터페이스 변형이 많은 이유

### 함수 아라티

함수의 인수 개수를 나타내는 아라티는 함수가 받아들이는 피연산자의 수를 의미한다.

| 인수가 1개인 경우     | 인수가 2개인 경우         |
| -------------- | ------------------ |
| Function\<T,R> | BiFunction\<T,U,R> |
| Consumer\<T>   | BiConsumer\<T,U>   |
| Predicate\<T>  | BiPredicate\<T,U>  |

> 자바의 기본적인 인터페이스들은 인수를 최대 2개까지 지원한다.
>
> 인수를 더 추가하는 방식을 권장하지는 않고 인수가 많은 함수는 객체를 사용하여 인수를 캡슐화하는 방식으로 해결할 수 있다.

#### Operator 함수형 인터페이스

| 아라티 | 연산자                | 슈퍼 인터페이스            |
| --- | ------------------ | ------------------- |
| 1   | UnaryOperator\<T>  | Function\<T,T>      |
| 2   | BinaryOperator\<T> | BiFuinction\<T,T,T> |

두 개의 String 인수를 받아 새로운 String 값을 생성하는 함수가 필요한 경우 BiFunction\<String, String, String>의 타입 정의는 다소 반복적이다.

이를 간단하게 사용하기위해 BinaryOperator\<String>가 생겨났다.

```java
@FunctionalInterface
interface BinaryOperator<T> extends BiFunction<T,T,T> {
    // ...
}
```

#### BiFunction

```java
BiFunction<String, String, String> concat = (s1, s2) -> s1 + s2;
String result = concat.apply("Hello, ", "World!");
System.out.println(result); // Hello, World!
```

#### BinaryOperator

```java
BinaryOperator<String> concat = (s1, s2) -> s1 + s2;
String result = concat.apply("Hello, ", "World!");
System.out.println(result); // Hello, World!
```

### 원시 타입

대부분의 함수형 인터페이스는 제네릭 타입 정의를 가지고 있다.

원사타입(primitive type)은 아직까지 제네릭 타입으로 사용될 수 없다. 그래서 원시 타입에 대해 특화된 함수형 인터페이스가 존재한다.

| 함수형 인터페이스        | 박싱된 변형                |
| ---------------- | --------------------- |
| IntFunction \<R> | Function\<Integer, R> |
| IntConsumer      | Consumer\<Integer>    |
| IntSupplier      | Supplier\<Integer>    |
| IntPredicate     | Predicate\<Integer>   |

객체 래퍼 타입에 대해서는 어떤 제너릭 함수형 인터페이스든 사용할 수 있으며 오토박싱이 나머지를 처리하도록 담당할 수 있다.

하지만 오토박싱은 무료로 처리되지 않기 때문에 성능에 영향을 줄 수 있어 JDK에서 제공하는 많은 함수형 인터페이스가 오토박싱을 피하기 위해 원시 타입을 사용하는 이유이다.

> 프로젝트 발할라와 특수 제네릭
>
> 현재 제네릭 타입 인수는 원시 타입과 호환되지 않는다. 따라서 오토박싱된 타입을 사용해야 한다. 이는 원시 타입을 직접 사용하는 것과 비교했을 때 성능 문제를 포함한 다른 사이드 이펙트가 발생할 수 있다.
>
> 특수 제네릭은 제네릭 타입을 기본 타입과 참조 타입 모두에 대해 최적화하는 것을 목표로 한다. 이는 주로 JDK 20과 이후의 버전에서 실험적으로 구현되고 있다고 한다.

## 함수 합성

함수 합성은 작은 함수들을 결합하여 더 크고 복잡한 작업을 처리하는 함수형 프로그래밍의 주요 접근 방식으로, 자바도 이를 지원한다.

Function\<T,R>의 경우 두 가지 기본 메서드를 사용할 수 있다.

```java
<V> Function<V,R> compose(Function<? super V, ? extends T> before)
<V> Function<T,V> andThen(Function<? super R, ? extends V> after)
```

* compose 메서드는 전달된 함수를 먼저 실행하고, 그 결과를 현재 함수에 적용한다.
  * 순서: before -> this
* andThen: andThen 메서드는 현재 함수를 먼저 실행하고, 그 결과를 전달된 함수에 적용한다.
  * 순서: this -> after

```java
Function<Integer, Integer> addTen = x -> x + 10;
Function<Integer, Integer> multiplyByTwo = x -> x * 2;

// compose: multiplyByTwo를 호출하기 전에 addTen을 호출합니다.
Function<Integer, Integer> composedFunction = multiplyByTwo.compose(addTen);

// andThen: addTen을 호출한 후 multiplyByTwo를 호출합니다.
Function<Integer, Integer> andThenFunction = addTen.andThen(multiplyByTwo);

int result1 = composedFunction.apply(5); // 30
int result2 = andThenFunction.apply(5); // 30
```
