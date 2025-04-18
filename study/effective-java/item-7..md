# item 7. 다 쓴 객체 참조를 해제하라

자바는 가비지 컬렉터를 갖춘 언어로 다 쓴 객체를 알아서 회수해준다. 그렇다고 메모리 관리에 더 이상 신경을 쓰지않아도 되는건 아니다.

그 이유는 스택을 간단히 구현한 코드로 살펴보자

```java
public class Stack {
  private Object[] elements;
  private int size = 0;
  private static final int DEFAULT_INITIAL_CAPACITY = 16;
  
  public Stack() {
    elements = new Object[DEFAULT_INITIAL_CAPACITY];
  }
  
  public void push(Object e) {
    ensureCapacity();
    elements[size++] = e;
  }
	
  public Object pop() {
    if (size == 0) {
      throw new EmptyStackException();
    }
    return elements[--size];
}

  private void ensureCapacity() {
    if (elements.length == size)
      elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

pop() 메서드로 참조하는 size가 줄어들게 되면 앞으로는 다시 쓰이지 않게 되지만 래퍼런스가 남아있어 가비지 컬렉터는 사용하는 객체인지 사용하지 않는 객체인지 판단을 못하여 회수하지 않는다.

이처럼 다시 쓰이지 않지만 가비지 컬렉터가 회수하지않는 참조를 다 쓴 참조(obsolete reference)라 한다.

\


#### 다 쓴 참조(obsolete reference) 해법

해법은 해당 참조를 다 썻을 때 null 처리(참조 해제)하면 된다.

다음은 pop 메서드를 제대로 구현한 모습이다.

```java
public Object pop() {
  if (size == 0) 
    throw new EmptyStackException();
  object result = elements[--size];
  elements[size] = null; // 다 쓴 참조 해제
  return result;
}
```

만약 null 처리한 참조를 실수로 사용하려 하면 프로그램은 즉시 NullPointException을 던지며 종료하여 프로그램 오류를 빠르게 잡을 수 있다.

\


### null 처리하는 일은 예외적이어야 한다.

필요 없는 객체를 볼 때마다 null 처리하면, 오히려 프로그램을 필요 이상으로 지저분하게 만든다.

필요없는 객체 레퍼런스를 정리하는 최선책은 그 레퍼런스를 가리키는 변수를 특정한 범위(scope)안에서만 사용하는 것이다. 변수의 범위를 가능한 최소가 되게 정의했다면(item 57) 이 일은 자연스럽게 이뤄진다.

\-> 변수를 지역변수로 선언하여 범위를 벗어나 자연스럽게 가비지 컬렉션에 대상이 되도록 하면 null 처리할 필요가 없다.

```java
public Object pop() {
  Object size = 10;
  ...
}
```

\


### 캐시 역시 메모리 누수를 일으키는 주범이다.

캐시를 사용할 때도 메모리 누수 문제를 조심해야 한다. 객체의 레퍼런스를 캐시에 넣어 놓고, 캐시를 비우는 것을 잊기 쉽다.

여러가지 해결책이 있지만, 캐시의 키에 대한 레퍼런스가 캐시 밖에서 필요 없어지면 해당 엔트리를 캐시에서 자동으로 비워주는 WeakHashMap을 쓸 수 있다.

> 캐시의 경우 시간이 지남에 따라 사용되지 않으면 그 가치를 떨어트리는 방법을 사용한다.

\


### Java Reference 종류

자바는 효율적인 GC를 처리하기 위한 reference 4가지가 있다.

개발자는 적절한 reference를 사용하여 GC에 의해 제거될 데이터 우선순위를 적용하여 좀 더 효율적인 메모리 관리를 할 수 있게 된다.

Reference는 `Strong Reference`,`Soft Reference`,`Weak Reference`, `Phantom Reference` 로 구분되어 있으며, 뒤로 갈수록 GC에 의해 제거될 우선순위가 높다.

#### Strong Reference

우리가 자주 사용하는 객체 만들 때 모든 래퍼런스는 전부 Strong reference이다

```java
Integer prime = 1;
```

이 객체를 가리키는 강한 참조가 있는 객체는 GC대상이 되지않는다.

#### Soft Reference

```java
Printer printer = new Printer();
SoftReference<Printer> softReference = new SoftReference<>(printer); 
printer = null;
```

![image](https://user-images.githubusercontent.com/70622731/180609164-fc7ed36f-aa9b-40e9-b7ab-0e22f878aefb.png)

> https://luckydavekim.github.io/development/back-end/java/soft-reference-in-java

최초 생성 시점에 이용 대상이 되었던 Strong Reference 은 없고 해당 객체를 참조하는 객체가 Soft Reference만 존재할 경우 GC 대상이 된다.

하지만 메모리가 부족하지 않으면 굳이 GC를 수행하지 않는다.

#### Weak Reference

```java
Printer printer = new Printer(); 
WeakReference<Printer> weakReference = new WeakReference<>(printer); 
printer = null;
```

최초 생성 시점에 이용 대상이 되었던 Strong Reference 은 없고 해당 객체를 참조하는 객체가 Weak Reference만 존재할 경우 GC 대상이 된다.

메모리가 부족하지 않아도 바로 GC에 대상이 된다.

#### Phantom Reference

GC 과정은 여러 단계로 나뉜다.

* GC 대상 객체를 찾는 작업
* 객체를 처리(finalize)
* 메모리를 회수하는 작업

Soft Reference, Weak Reference는 GC 대상 객체를 찾는 작업에 개발자가 관여할 수 있게 해준다.

하지만 Phantom Reference는 객체를 처리(finalize) , 메모리를 회수하는 것에 관여한다.

> GC가 객체를 처리하는 순서는 다음과 같다. Strong Ref > Soft Ref > Weak Ref > finalize > phantom Ref > 메모리 회수

객체가 Strong,Soft, Weak 참조에 의해 참조되는지 판단하고 모두 아니면 finalize를 진행하고 phantom 여부를 판단한다. 따라서 finalize() 이후에 처리해야하는 리소스 정리 작업을 할 때 개발자가 관여할 수 있다. 하지만 거의 안쓰인다.

\


### WeakHashMap

HashMap 같은 경우는 어떤 객체가 null이 되어 버리면 해당 객체를 key로 하는 HashMap의 value 값도 더이상 꺼낼 일이 없는 경우가 발생하여 메모리를 차지한다.

하지만 캐시 값이 무의미해진다면 자동으로 처리해주는 WeakHashMap은 key 값을 모두 Weak 레퍼런스로 감싸 Strong reference가 없어진다면 GC의 대상이 된다.

> 주의할점 - WeakHashMap에 key로 String은 적합하지 못하다 String은 JVM에 의해 다른 곳에 store 되어 항상 strong reference로 남아있기 때문이다. (사용하고 싶으면 new로 생성)

\


### 리스너, 콜백

클라이언트 코드가 콜백을 등록할 수 있는 API를 만들고 콜백을 뺼 수 있는 방법을 제공하지 않는다면, 계속해서 콜백이 쌓이기만 할 것이다. 이것 역시 `WeakHashMap`을 사용해서 콜백을 Weak 레퍼런스로 저장하면 GC가 이를 즉시 수거해 해결할 수 있다.
