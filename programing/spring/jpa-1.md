# JPA 영속성 컨텍스트, 연관관계 편의메서드

### 영속성 컨텍스트

***

영속성 컨텍스트는 엔터티를 데이터베이스에 저장하기 전에 일시적으로 관리하는 역할을 합니다.

#### 1차 캐시 (First-Level Cache)

한 트랜잭션 내에서 같은 엔티티를 조회할 때, 처음에만 데이터베이스에서 데이터를 가져오고, 이후에는 1차 캐시에서 데이터를 제공합니다.

```java
public void test(Long userId) {
    // 첫 번째 조회
    User user = userRepository.findById(userId).orElseThrow();

    // 같은 트랜잭션 내에서 두 번째 조회
    User sameUser = userRepository.findById(userId).orElseThrow();
    // 이 경우, 두 번째 조회는 데이터베이스에 접근하지 않고 1차 캐시에서 가져옴
}
```

#### 더티 체킹 (Dirty Checking)

영속성 컨텍스트는 더티 체킹을 통해 엔티티의 변경 사항을 자동으로 감지하고, 트랜잭션이 커밋될 때 이 변경 사항을 데이터베이스에 반영합니다.

```java
@Transactional
public void updateUserEmail(Long userId, String newEmail) {
    User user = userRepository.findById(userId).orElseThrow();
    user.setEmail(newEmail);
    // 메서드 종료 시, 변경 감지에 의해 트랜잭션이 커밋되고 데이터베이스가 업데이트 됨
}
```

> 트랜잭션은 @Transactional이 적용된 메서드의 실행이 시작될 때 시작되고, 메서드 실행이 정상적으로 종료될 때 커밋됩니다.

#### 쓰기 지연 (Write-Behind)

영속성 컨텍스트는 쓰기 지연 SQL 저장소를 사용하여 엔티티의 변경(INSERT, UPDATE, DELETE)을 즉시 데이터베이스에 반영하지 않고, 트랜잭션이 커밋될 때까지 지연시킵니다.

```java
@Transactional
public void updateUser(Long userId, String newName) {
    User user = userRepository.findById(userId).orElseThrow();
    user.setName(newName);
    // 메서드 종료 시, 트랜잭션이 커밋되고 쓰기 지연이 발생함
}
```

#### 지연 로딩 (Lazy Loding)

지연 로딩은 주로 프록시 객체를 사용하여 구현됩니다. JPA에서 엔티티를 지연 로딩으로 설정하면, 해당 엔티티의 실제 인스턴스 대신 프록시 인스턴스를 생성하고, 이 프록시 인스턴스는 영속성 컨텍스트에 의해 관리됩니다.

```java
public List<Order> getUserOrders(Long userId) {
    User user = userRepository.findById(userId).orElseThrow();
    return user.getOrders(); // 실제로 orders에 접근하는 시점에 데이터베이스에서 로드됨
}
```

> 프록시 인스턴스는 실제 엔티티 대신 사용되며, 엔티티에 대한 첫 번째 접근이 발생할 때까지 실제 데이터베이스 로드를 지연시키는 데 도움을 줍니다.

#### save메서드 호출 이후 flush하지 않았는데 바로 user id를 조회할 수 있는 이유

save 메서드를 호출하면, 해당 엔터티는 영속성 컨텍스트에 저장됩니다. `IDENTITY 전략`을 사용하는 경우, JPA는 엔터티를 데이터베이스에 즉시 저장해야만 ID를 얻을 수 있습니다.

따라서, save 호출 시 데이터베이스에 엔터티가 저장되고, 바로 ID를 반환받을 수 있습니다.

flush는 영속성 컨텍스트의 상태를 데이터베이스에 동기화하는 작업입니다. 하지만 IDENTITY 전략을 사용할 경우, save 메서드 호출 자체가 이미 데이터베이스에 쓰기를 수행하기 때문에, 별도로 flush를 호출하지 않아도 ID를 바로 얻을 수 있습니다.

```java
public Long createUserAndGetId(String name) {
    User user = new User();
    user.setName(name);
    
    user = userRepository.save(user); // 엔터티 저장
    return user.getId(); // 바로 ID 조회
}
```

> IDENTITY 전략을 사용하는 경우에는 쓰기 지연을 사용할 수 없습니다.

\


### JPA Id 전략

***

JPA(Java Persistence API)에서 ID 전략은 엔터티의 기본 키(primary key)를 생성하는 방법을 정의합니다.

#### AUTO

기본값으로, 특정 데이터베이스에 맞게 자동으로 선택됩니다. 데이터베이스가 지원하는 ID 생성 전략(예: 자동 증가, 시퀀스)을 사용합니다.

> @GeneratedValue(strategy = GenerationType.AUTO)

#### IDENTITY

데이터베이스의 자동 증가 기능을 사용하여 ID를 생성합니다. 각 삽입(INSERT) 시 데이터베이스가 ID를 생성하므로, 엔터티가 영속성 컨텍스트에 저장될 때 즉시 데이터베이스에 쓰여집니다.

> @GeneratedValue(strategy = GenerationType.IDENTITY)

#### SEQUENCE

데이터베이스의 시퀀스를 사용하여 ID를 생성합니다. 시퀀스는 연속된 값을 제공하며, 다른 트랜잭션과 독립적으로 값을 생성할 수 있습니다.

> @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "sequence-generator")

#### TABLE

별도의 키 생성 테이블을 사용하여 ID를 생성합니다. 이 전략은 모든 데이터베이스에서 사용 가능하지만, 성능이 상대적으로 낮을 수 있습니다.

> @GeneratedValue(strategy = GenerationType.TABLE, generator = "table-generator")

\


### 연관관계 편의메서드

***

양방향 연관관계에서 양쪽 엔터티 모두에 연관관계를 설정해야 할 경우, 중복된 코드를 작성하는 것을 방지합니다.

Post와 Comment 라는 두 엔터티가 있고, 이들 간에 양방향 연관관계가 있을때 Post 엔터티에 addComment라는 편의 메서드를 작성하여 Comment를 추가할 때 Post와 Comment 양쪽 모두에서 연관관계가 설정되도록 할 수 있습니다.

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post")
    private List<Comment> comments = new ArrayList<>();

    // 연관관계 편의 메서드
    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }
}
```

연관관계 편의 메서드의 사용은 데이터베이스에 직접적으로 저장되는 값과는 다릅니다. 예를 들어, `@OneToMany`로 매핑된 List는 데이터베이스 테이블에 직접 저장되는 필드가 아니라, 객체 간의 관계를 나타내는 것입니다.

그래서 관계 설정이후 바로 트랜잭션이 끝나버리면 db와 상관없으므로 낭비라 생각할 수 있지만

한 트랜잭션 안에서 객체를 다시 사용하는 경우, 관계설정이 되어있지 않을 경우 NullpointerException이 발생하므로 보통 연관관계 맺을때 연관관계 편의메서드는 많이 사용합니다.

#### 연관관계 편의메서드를 통해 save안하고 저장하는법

`영속성 전이(Cascading)` 기능을 사용하여 관련 엔터티를 자동으로 저장할 수 있습니다. 이 기능을 사용하면, 한 엔터티를 저장할 때 연관된 엔터티도 함께 저장됩니다.

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL)
    private List<Comment> comments = new ArrayList<>();

    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }
}
```

CascadeType.ALL은 모든 종류의 영속성 전이를 포함합니다(생성, 갱신, 삭제 등). 이 설정을 통해 Post 엔터티를 저장할 때 연관된 모든 Comment 엔터티도 자동으로 저장됩니다.

\


### OSIV 설정

***

`OSIV(Open Session in View)`는 하이버네이트나 JPA 같은 ORM(Object-Relational Mapping) 기술을 사용하는 웹 애플리케이션에서 자주 사용되는 패턴 중 하나입니다.

이 설정의 목적은 웹 요청의 수명 주기 동안 하나의 세션(또는 영속성 컨텍스트)을 열어두고 유지하는 것입니다 (Default가 true이다)

#### OSIV의 장점

* 지연 로딩을 통해 필요한 시점에 데이터를 불러올 수 있어, 애플리케이션의 유연성이 향상됩니다.
* 애플리케이션 계층 간의 데이터 접근이 용이해지며, 코드가 간결해질 수 있습니다. (controller단에서도 영속성 컨텍스트가 살아있다)

#### OSIV의 단점

* 요청 처리 시간이 길어질수록 데이터베이스 커넥션을 오래 점유하게 되어, 시스템 리소스에 부담을 줄 수 있습니다.
* 잘못 관리될 경우, 데이터베이스의 과부하나 성능 저하를 일으킬 수 있습니다.

> 지연로딩은 영속성 컨텍스트가 살아있어야 가능하고, 영속성 컨텍스트는 기본적으로 데이터베이스 커넥션을 유지한다.

![image](https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/5e6a0103-9a23-4bca-b3c7-a09bae3e8696)

![image](https://github.com/rlatmd0829/rlatmd0829.github.io/assets/70622731/b8af157c-c11c-4c9a-8dbe-5930e05e9e0a)

\


***

### Reference

* [https://velog.io/@suk13574/JPA-%EC%98%81%EC%86%8D%EC%84%B1-%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8%EC%9D%98-%EC%A0%84%EB%B0%98%EC%A0%81%EC%9D%B8-%EC%9D%B4%ED%95%B4%EA%B0%9C%EB%85%90-%EC%9E%A5%EC%A0%90-%EB%8F%99%EC%9E%91-%EB%B0%A9](https://velog.io/@suk13574/JPA-%EC%98%81%EC%86%8D%EC%84%B1-%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8%EC%9D%98-%EC%A0%84%EB%B0%98%EC%A0%81%EC%9D%B8-%EC%9D%B4%ED%95%B4%EA%B0%9C%EB%85%90-%EC%9E%A5%EC%A0%90-%EB%8F%99%EC%9E%91-%EB%B0%A9)
