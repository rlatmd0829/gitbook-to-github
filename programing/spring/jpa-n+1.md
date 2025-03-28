# JPA 연관관계 N+1 문제

### N+1 문제

***

JPA를 사용하다면 보면 N+1문제를 많이 만나게 됩니다. 그에 따른 발생 원인과 해결법을 정리해보았습니다.

아래 예시 코드는 한명의 User 여러개의 Article을 가질 수 있는 구조(User : 1, Article : N)입니다.

```java
@Entitiy
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(length = 10, nullable = false)
    private String name;
    
    @OneToMany(mappedBy = "user")
    private Set<Article> articles = new HashSet<>();
}
```

```java
@Entitiy
public class Article {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(length = 50, nullable = false)
    private String title;
    
    @ManyToOne
    private User user;
}
```

모든 유저를 조회하는 findAll 쿼리를 날려서 List를 받아왔을때 리스트를 순회하며 각 사용자(User)가 가진 기사들(Article)을 조회합니다.

사용자마다 Set

에 접근할 때마다 해당 사용자의 기사들을 불러오는 쿼리가 실행됩니다.

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public void displayUserArticles() {
        // 모든 User 조회
        List<User> users = userRepository.findAll();

        for (User user : users) { 
            // user가 50명이라면 for문돌때마다 조회하므로 50번 쿼리발생
            Set<Article> articles = user.getArticles(); // Lazy Loading 발생 
            for (Article article : articles) {
                // Article 정보 처리
            }
        }
    }
}
```

만약 50명의 User가 있으면, 처음에 사용자 목록을 조회하는 쿼리 1개와 각 User마다 Article 목록을 조회하는 쿼리 50개가 발생합니다. 이렇게 총 51개의 쿼리가 실행되는 상황이 발생하는 것이 N+1 문제입니다.

\


### Fetch join으로 N+1 문제해결

***

lazyLoding이 걸려있고 Fetch join을 사용하지 않으면 해당엔티티만 조회를 해오고, 지연 로딩으로 설정되어 있는 다른엔티티는 프록시 객체로 가져옵니다.

> **Eager로딩도 마찬가지로 N+1 문제가 발생한다 즉시,지연의 차이이고 해결하려면 fetch join을 써야한다.**

이후 다른엔티티의 정보를 조회하면 다른엔티티의 정보를 위한 SELECT 쿼리가 별도로 나갑니다.

이러한 문제를 해결하기 Fetch join을 사용합니다.

```java
public List<User> findAllUsersWithArticles() {
    return queryFactory
        .selectFrom(user)
        .leftJoin(user.articles, article).fetchJoin()
        .fetch();
}
```

Fetch Join은 N + 1 문제를 해결하는 방법입니다. 연관된 엔티티들을 한 번의 쿼리로 함께 가져오게 하여, 필요한 모든 데이터를 효율적으로 로딩합니다.

\


### EntityGraph로 N+1 문제해결

***

@EntityGraph를 사용하면 특정 엔티티의 연관 관계를 미리 정의된 방식으로 로딩할 수 있으며, 이를 통해 N+1 문제를 해결할 수 있습니다.

엔티티 클래스에 @NamedEntityGraph를 사용하여 엔티티 그래프를 정의합니다.

```java
@Entity
@NamedEntityGraph(name = "User.withArticles",
                 attributeNodes = @NamedAttributeNode("articles"))
public class User {
    @Id
    private Long id;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private Set<Article> articles;
}
```

스프링 데이터 JPA 리포지토리에서 @EntityGraph 어노테이션을 사용하여 해당 엔티티 그래프를 적용합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(value = "User.withArticles", type = EntityGraph.EntityGraphType.FETCH)
    List<User> findAll();
}
```

findAll() 메소드는 각 User 엔티티와 함께 관련된 Article 엔티티들을 한 번의 쿼리로 로딩하여 가져옵니다.

\


### @BatchSize 로 N+1 문제해결

***

@BatchSize를 엔티티 클래스에 적용하여, 해당 엔티티의 인스턴스를 로드할 때 일정 수량(batch size)만큼 한 번에 로드합니다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ...
	
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @BatchSize(size = 10) // 여기서 BatchSize를 지정합니다.
    private Set<Article> articles = new HashSet<>();
}
```

User가 10명이 있고 각각 Article을 5개씩 가지고 있을 경우 예시로 @BatchSize가 어떻게 N+1을 해결하는지 확인해보겠습니다.

#### BatchSize 없이 Article 로드 쿼리

```sql
SELECT * FROM Article WHERE userId = 1;
SELECT * FROM Article WHERE userId = 2;
...
SELECT * FROM Article WHERE userId = 10;
```

#### @BatchSize(size = 10) 적용 시 Article 로드 쿼리

```sql
SELECT * FROM Article WHERE userId IN (1, 2, ..., 10);
```

* @BatchSize 설정 없이: 각 유저에 대해 별도의 아티클 로드 쿼리가 필요합니다.
* @BatchSize(size = 10) 설정 시 하나의 쿼리로 여러 유저의 아티클을 효율적으로 로드할 수 있습니다.

이렇게 함으로써, 관련된 Article 엔티티를 user수 만큼 조회하는 대신, 더 적은 수의 쿼리로 필요한 데이터를 가져올 수 있게 됩니다.

> **yml에 default\_batch\_fetch\_size 설정으로 전역적으로 BatchSize를 설정할 수 있습니다.**

#### batch\_size와 default\_batch\_fetch\_size 차이

```java
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20
        default_fetch_batch_size: 30
```

yml에서 이런식으로 설정하는데 두 설정은 각각 다른 상황과 목적에 사용되므로 헷갈리지 않도록 주의가 필요합니다.

* **jdbc batch\_size**는 주로 벌크 작업(Bulk Operations)과 관련이 있습니다. 예를 들어, 한 번에 여러 개의 엔터티를 데이터베이스에 삽입하거나 업데이트할 때 사용됩니다.
* **default\_fetch\_batch\_size**는 주로 조회(select)시 연관된 엔터티를 함께 로딩할 때 사용되는 설정입니다.

\


### Fetch Join을 쓸 경우 발생하는 문제상황

***

#### Fetch Join과 Pageable을 함께 사용할 때 Limit이 적용되지 않는 이유

**Fetch Join과 Pageable의 충돌**

* Pageable은 페이지 단위로 데이터를 조회하는 기능입니다. 쿼리 결과의 특정 부분만을 가져오는데 사용되며, 주로 limit과 offset을 SQL 쿼리에 추가하여 구현됩니다.
* Fetch Join은 연관된 모든 엔티티를 로드합니다. 이 때문에 결과 집합의 크기가 증가할 수 있습니다.
* Pageable은 결과 집합에 대해 limit과 offset을 적용합니다. 그러나 Fetch Join으로 인해 결과 집합의 크기가 커지면, 예상치 못한 방식으로 limit과 offset이 작동할 수 있습니다.

**Limit 적용 문제**

* Fetch Join 사용 시, 여러 Article이 연결된 각 User가 하나의 결과 행으로 처리되어 Pageable의 limit에 의해 실제 반환되는 User 수가 기대보다 적을 수 있습니다.

\


#### Fetch Join 카테시안 곱 문제

기본적으로 Fetch Join은 Inner Join과 유사한 동작을 합니다.

> **INNER JOIN은 두 테이블에서 일치하는 행만 반환합니다. 즉, 두 테이블 모두에 해당 조건을 만족하는 행이 있을 때만 결과에 포함됩니다.**
>
> **LEFT JOIN은 왼쪽 테이블의 모든 행과, 오른쪽 테이블에서 조건을 만족하는 행을 반환합니다. 오른쪽 테이블에 일치하는 행이 없는 경우, 해당 필드는 NULL로 채워집니다.**

Fetch Join을 사용하여 Post와 Comment를 조인하는 경우, 다음과 같은 상황이 발생할 수 있습니다

* 각 Post 엔터티에 대해 연관된 모든 Comment 엔터티가 조회됩니다.
* 만약 한 Post가 5개의 Comment와 연관되어 있다면, 쿼리 결과는 해당 Post를 5번, 각각 다른 Comment와 함께 반환합니다.
* 결과적으로, 결과 집합에서 Post 엔터티가 중복되어 나타납니다.

```java
SELECT p, c
FROM Post p
INNER JOIN FETCH p.comments c

// 예상되는 쿼리 결과
| p.post_id | p.title | c.comment_id | c.content    |
|-----------|---------|--------------|--------------|
| 1         | Post 1  | 1            | Comment 1-1  |
| 1         | Post 1  | 2            | Comment 1-2  |
| 2         | Post 2  | 3            | Comment 2-1  |
```

**카테시안곱 해결 방법**

* 컬렉션 Set 사용
* distinct() 사용

Set을 사용하면 위에서 Post가 중복으로 생기는 현상을 해결할 수 있습니다.

distinct는 결과 집합에서 동일한 엔터티 인스턴스를 제거하는데 사용되며, 이는 결과를 메모리에 로드한 후에 적용됩니다. 따라서, 데이터베이스 수준에서는 여전히 중복된 행이 반환되지만, 애플리케이션 수준에서는 중복이 제거됩니다.

```java
// DISTINCT 사용
SELECT DISTINCT p, c
FROM Post p
INNER JOIN FETCH p.comments c

// 예상되는 쿼리 결과
| p.post_id | p.title | c.comment_id | c.content    |
|-----------|---------|--------------|--------------|
| 1         | Post 1  | 1            | Comment 1-1  |
| 1         | Post 1  | 2            | Comment 1-2  |
| 2         | Post 2  | 3            | Comment 2-1  |
```

distinct의 효과는 SQL 쿼리 레벨과 ORM 레벨에서 다를 수 있으며, 특히 Fetch Join을 사용하는 경우 ORM 레벨에서 중복 제거의 중요성이 더 커집니다.

ORM 프레임워크는 이러한 결과를 메모리에 로드한 후, 동일한 Post 엔터티 인스턴스를 중복으로 간주하고 하나만 유지합니다. 결과적으로 애플리케이션에는 각 Post 엔터티가 한 번씩만 표현됩니다.

\


#### MultipleBagFetchException 에러

2개 이상의 OneToMany 자식 테이블에 Fetch Join을 선언했을때 MultipleBagFetchException 발생합니다.

JPA는 한 쿼리에서 하나의 컬렉션만 효율적으로 처리할 수 있으며, 둘 이상의 컬렉션을 처리하려고 하면 결과 집합의 크기가 기하급수적으로 증가할 수 있기 때문입니다.

```java
@Entity
public class User {
    // ... 

    @OneToMany(mappedBy = "user")
    private Set<Article> articles;

    @OneToMany(mappedBy = "user")
    private Set<Comment> comments;
}
```

**JPA에서 Fetch Join의 조건은 다음과 같습니다.**

* ToOne은 몇개든 사용 가능
* ToMany는 1개만 가능

Hibernate는 내부적으로 컬렉션을 bag으로 처리합니다. bag은 순서가 정해져 있지 않고, 중복을 허용하는 컬렉션입니다.

하나의 엔티티에 여러 bag 타입의 컬렉션을 동시에 Fetch Join으로 조회하려고 하면, Hibernate는 이를 효율적으로 처리할 수 없어 MultipleBagFetchException을 발생시킵니다.

**해결방법**

* 컬렉션 Set 사용
* BatchSize 사용

각각의 List를 fetch join할 때, 결과 집합은 카테시안 곱(Cartesian product)을 형성하여 크게 증가할 수 있습니다. 이때 위에서 말한 카테시안곱 해결방안 처럼 Set을 사용하면 중복을 제거 할 수 있습니다.

BatchSize 방식은 직접적으로 해결하는 것은 아니지만 MultipleBagFetchException을 피하면서도 성능에 미치는 영향을 최소화하는 데 도움이 됩니다.

\


***

### Reference

* [https://velog.io/@wnguswn7/Project-%EC%B9%B4%ED%85%8C%EC%8B%9C%EC%95%88-%EA%B3%B1%EC%9D%98-%EB%B0%9C%EC%83%9D%EA%B3%BC-%ED%95%B4%EA%B2%B0](https://velog.io/@wnguswn7/Project-%EC%B9%B4%ED%85%8C%EC%8B%9C%EC%95%88-%EA%B3%B1%EC%9D%98-%EB%B0%9C%EC%83%9D%EA%B3%BC-%ED%95%B4%EA%B2%B0)
* [https://jojoldu.tistory.com/457](https://jojoldu.tistory.com/457)
