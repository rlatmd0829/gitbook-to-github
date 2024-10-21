# 9장 Optional을 사용한 null 처리

* 원시타입 기본 값 : 숫자 타입은 0, boolean 타입은 false
* 클래스나 인터페이스, 배열 : null

## null 참조의 문제점

* 예상치 못한 오류 발생
  * Null은 변수, 인수, 반환값에 유효한 값이 될 수 있지만, 이 값이 예상되지 않거나 적절하게 처리되지 않을 경우 프로그램에서 예기치 않은 오류가 발생할 수 있다.
* 타입 모호성
  * Null은 특정 타입을 가지지 않으며, 모든 타입을 대표할 수 있다. 이로 인해 코드에서 null을 사용하면 값의 부재를 나타내기 위한 일관된 타입이나 키워드를 사용할 수 없게 되고, 타입 안정성을 저하시킬 수 있다.

## 자바에서 null을 다루는 방법 (Optional 도입 전)

### 변수를 null로 초기화하지 않도록 주의하기

변수는 항상 null이 아닌 값을 가져야 한다.

```java
// 비권장
String value = null;
if (condition) {
    value = "Condition is true";
} else {
    value = "Condition is false";	
}

// 권장
String asTernary = condition ? "Condition is true" : "Condition is false";
```

### null 값을 전달하거나 수용 또는 반환하지 않아야 한다.

변수가 null이 아니어야 하는 것처럼 모든 인수나 반환값 역시 null을 피해야 한다.

필수가 아닌 값에 대한 특정 메서드와 생성자를 제공한 후 원본 메서드에서 null을 허용하지 않아야 한다.

```java
public record User(long id, String firstname, String lastname) {
    public User{
        Objects.requireNonNull(firstname);
        Objects.requireNonNull(lastname);
    }
}
```

### 도구를 활용한 null 검사

자바의 null 참조 경우 어노테이션을 사용하여 변수, 인수 및 메서드 반환 타입을 @Nullable 또는 @NonNull로 표시할 수 있다.

```java
@NonNull
List<@Nullable String> getListOfNullableStrings();

@Nullalbe
List<@NonNull String> getNullableListOfNonNullStrings();

void doWork(@Nullable String identifier);
```

## Optional 알아보기

자바 8의 새로운 Optional은 null을 일관성 있게 처리하기 위한 전문 타입일 뿐만 아니라, JDK에서 사용할 수 있는 모든 함수적 기능의 혜택을 받는 유사 함수형 파이프라인 이기도 하다.

Optioanl은 스트림처럼 부가적인 래퍼는 메서드 호출과 그 결과로 생기는 스택 프레임에 대한 오버헤드를 발생시킨다.

그러나 null 값을 가진 일반적인 작업 흐름을 더욱 간결하고 직관적으로 표현할 수 있는 추가적인 기능을 제공한다.

### Optional 없이 내용 불러오기

```java
public Content get(String contentId) {
    if (contentId == null) {
        return null;
    }
	
    if (contentId.isBlank()) {
        return null;
    }
    
    var cacheKey = contentId.toLowerCase();
	
    var content = this.cache.get(cacheKey);
    if (content == null) {
        content = loadFromDB(contentId);
    }
    
    if (content != null) {
        return null;
    }
	
    if (!content.isPublished()) {
        return null;
    }
    
    return null;
}
```

### Optional 호출 체인을 사용하여 내용 불러오기

```java
public Optional<Content> get(String contentId) {
    return Optional.ofNullable(contentId)
        .filter(Predicate.not(String::isBlank))
        .map(String::toLowerCase)
        .map(this.cache::get)
        .or(() -> loadFromDB(contentId))
        .filter(Content::isPublished);
}
```

이 방식은 모든 상황에 Null 검사를 하는 전통적인 방식과 대비되는 방식이다.

### Optional 생성

Optional 파이프라인은 스트림과는 다르게 파이프라인에 최종 연산전 느긋하게 연결되지 않고 모든 연산은 호출되는 즉시 수행된다.

* Optional.ofNullable(T value) : 값이 null일 가능성이 있거나 비어 있을 수 도 있다고 생각되면 ofNullable를 통해 Null 값이 허용되는 새 인스턴스를 만든다.
* Optional.of(T value) : 값이 null이 아님을 확신할 때 사용한다. 값이 null이면 NullPointerException이 발생한다.
* Optional.empty() : 비어 있는 Optional 인스턴스를 만든다.

## 원시 타입용 Optional

원시 타입은 결코 null이 될 수 없기 때문에 원시 타입에 대한 Optional이 필요하지 않다고 생각할 수 있다.

기술적으로는 맞지만 Optional은 단순히 값이 null이 되는 것을 방지하는 것이 전부가 아니다. Optional은 기본 자료형에서 표현할 수 없는 아무것도 없는 실제 상태를 나타낸다.

그렇지만 Optional<> 타입과 원시 타입을 함께 사용하는 것은 불가능하다. 원시타입은 제네릭 타입으로 사용될 수 없기 때문이다.

그래서 일반적인 원시 타입들은 전용 Optional 변수를 사용할 수 있다.

* OptionalInt
* OptionalLong
* OptionalDouble

해당 내용을 사용하면 성능을 향상 시키기 위해 불필요한 오토박싱을 제거할 수 있지만, Optional이 제공하는 다양한 기능을 모두 사용할 수 는 없다. (filter, map, flatMap과 같은 메서드를 제공하지 않는다.)
