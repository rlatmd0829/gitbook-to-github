# item 1. 생성자 대신 정적 팩터리 메서드를 고려하라

### 정적 팩터리 메서드란?

정적 팩터리 메서드란 객체 생성의 역할을 하는 클래스 메서드이다.

```
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

정적 팩터리 메서드는 디자인패턴에서의 팩터리 메서드와 다르다. 디자인 패턴중에는 이와 일치하는 패턴은 없다.

### 정적팩터리 메서드 장점

**1. 이름을 가질 수 있다.**

생성자 자체만으로는 반환될 객체의 특성을 제대로 설명하지 못하지만 정적 팩토리 메서드는 가능하다.

**2. 호출될 때마다 인스턴스를 새로 생성하지는 않아도 된다.**

생성자는 호출할 때마다 새로운 인스턴스를 생성하지만 정적팩토리 메서드는 작성하는거에 따라 계속 새로 안만들 수 있다.

**3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.**

인터페이스를 정적 팩터리 메서드의 반환타입으로 사용하여 반환 타입의 하위 타입 객체를 반환 할 수 있다.

**4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.**

입력 매개 변수에 따라서 인터페이스 안에 있는 여러 클래스중 한개를 선택하여 반환할 수 있다.

**5. 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.**

### 정적팩터리 메서드 단점

**1. 상속을 하려면 public이나 protected 생성자가 필요하니 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.**

**2. 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.**

생성자는 javadoc을 통해 쉽게 파악할 수 있지만 정적팩터리 메서드는 아니다.

### 정적팩터리 메서드 네이밍 컨벤션

* from : 하나의 매개변수를 받아서 객체를 생성
* of : 여러개의 매개변수를 받아서 객체를 생성
* valueOf : from과 of의 더 자세한 버전
* instance, getInstance : 인스턴스를 생성, 이전에 반환했던 것과 같을 수 있음
* create, newInstance : 새로운 인스턴스를 생성
* get\[OtherType] : 다른 타입의 인스턴스를 생성. 이전에 반환했던 것과 같을 수 있음
* new\[OtherType] : 다른 타입의 새로운 인스턴스를 생성
* type : getType과 newType의 간결한 버전
