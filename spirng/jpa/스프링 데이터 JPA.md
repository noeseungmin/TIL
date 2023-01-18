# 스프링 데이터 JPA
### 스프링 데이터 JPA 설정
application.yml
```java
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/datajpa
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
      # show_sql: true
        format_sql: true

logging.level:
  org.hibernate.SQL: debug
  #org.hibernate.type: trace
```
* `show_sql` 옵션은 System.out에 하이버네이트 실행 SQL을남긴다. 
* `org.hibernate.SQL` 옵션은 logger를 통해 하이버네이트 실행 SQL을 남겨준다.

### 도메인
```java
//Member
 @ManyToOne(fetch = FetchType.LAZY) //항상 lazy
 @JoinColumn(name = "team_id")
 private Team team;
//...
public void changeTeam(Team team) {
 this.team = team;
 team.getMembers().add(this);
 }
```
* @Setter는 실무에서 가급적 사용 X, 비즈니스 로직을 반영
* @NoArgsConstructor(access=AccessLevel.PROTECTED) : 기본 생성자는 막아야 하는데, jpa에서 프록시를 만들기 위해서는 protected까지는 열어놓아야 함.
* @ToString : 가급적 내부 필드만 사용(연관관계 없는 필드만)
* `changeTeam()`으로 양방향 연관관계 한번에 처리하기!

```java
//Team
@OneToMany(mappedBy = "team")
 List<Member> members = new ArrayList<>();
```
* Member와 Team 양방향 연관관계로 `Member.team`는 외래키 값을 관리하는 주인 `Team.members`는 주인이 아님
* @mappedBy 사용하고 Team은 읽기만 가능하다.

### 리포지토리
### 순수 JPA 기반 리포지토리
```java
@Repository
public class TeamJpaRepository {
@PersistenceContext
 private EntityManager em;
public Member save(Member member) {
 em.persist(member);
 return member;
 }
 public void delete(Member member) {
 em.remove(member);
 }
 public List<Member> findAll() {
 return em.createQuery("select m from Member m", Member.class)
 .getResultList();
 }
//...
```
### 스프링 데이터 JPA
```java
public interface MemberRepository extends JpaRepository<Member, Long> {}
```
스프링 Data Jpa에서 구현 클래스를 만들어줘 사용자는 인터페이스를 만들고 의존성 주입을 받아 사용.
* @Repository 애노테이션 생략 가능
  * 컴포넌트 스캔을 스프링 데이터 JPA가 자동으로 처리
  * JPA 예외를 스프링 예외로 변환하는 과정도 자동으로 처리

* Generic
  * T: 엔티티 타입
  * ID: 식별자 타입(PK)

#### 주요 메서드
* save(S) : 새로운 엔티티는 저장하고 이미 있는 엔티티는 병합한다.
* delete(T) : 엔티티 하나를 삭제한다. 내부에서 EntityManager.remove() 호출
* findById(ID) : 엔티티 하나를 조회한다. 내부에서 EntityManager.find() 호출
* getOne(ID) : 엔티티를 프록시로 조회한다. 내부에서 EntityManager.getReference() 호출
* findAll(…) : 모든 엔티티를 조회한다. 정렬(`Sort`)이나 페이징(`Pageable`) 조건을 파라미터로 제공할 수 있다

## 쿼리 메소드 기능
메소드 이름만으로 쿼리 생성, 인터페이스에 메소드만 선언하면 해당 메소드의 이름으로 적절한 JPQL 쿼리를 생성해서 실행

쿼리 메소드 기능 3가지
* 메소드 이름으로 쿼리 생성
* 메소드 이름으로 JPA NamedQuery 호출
* `@Query` 어노테이션을 사용해서 리파지토리 인터페이스에 쿼리 직접 정의

### 메소드 이름으로 쿼리 생성
순수 JPA
```java
public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
 return em.createQuery("select m from Member m where m.username = :username 
and m.age > :age")
 .setParameter("username", username)
 .setParameter("age", age)
 .getResultList();
}
```
//스프링 데이터 JPA
```java
List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
```
스프링 데이터 jpa가 메소드 이름을 분석해서 쿼리를 날려준다.

#### 스프링 데이터 JPA가 제공하는 쿼리 메소드 기능
* 조회: find…By ,read…By ,query…By get…By 등등
* count…By delete…By, exists…By 등등 ...에는 넣고싶은 명칭
* LIMIT: findFirst3, findFirst, findTop, findTop3
이 기능은 필드명 변경시 인터페이스에 정의한 메서드 이름도 꼭 변경해야 한다. 그렇지 않으면 애플리케이션을 시작하는 시점에 오류가 발생, 이렇게 애플리케이션 로딩 시점에 오류를 인지할 수 있는 것이 스프링 데이터 JPA의 매우 큰 장점이다.

### JPA NamedQuery
#### @NamedQuery 어노테이션으로 Named 쿼리 정의
```java
@Entity
@NamedQuery(
 name="Member.findByUsername",
 query="select m from Member m where m.username = :username")
public class Member {
```

#### JPA 직접 사용, 스프링 JPA 사용
```java
public List<Member> findByUsername(String username) {
 ...
 List<Member> resultList =
 em.createNamedQuery("Member.findByUsername", Member.class)
 .setParameter("username", username)
 .getResultList();
 ```
 ```java
 @Query(name = "Member.findByUsername")
List<Member> findByUsername(@Param("username") String username);
```
`@Query` 생략하고 메서드 이름만으로 Named 쿼리를 호출 할 수있다.
* 스프링 데이터 JPA는 선언한 "도메인 클래스 + .(점) + 메서드 이름"으로 Named 쿼리를 찾아서 실행
* 만약 실행할 Named 쿼리가 없으면 메서드 이름으로 쿼리 생성 전략을 사용한다. 
* NamedQuery를 직접 등록해서 사용하는 일은 드물다 @Query를 사용해서 리파지토리 메소드에 쿼리를 직접 정의하는 방식을 사용한다.

### Query 리포지토리 메소드에 쿼리 정의하기
#### 메서드에 JPQL 쿼리 작성
```java
@Query("select m from Member m where m.username= :username and m.age = :age")
List<Member> findUser(@Param("username") String username, @Param("age") int age);
```
* 실행하는 메서드에 정적 쿼리를 직접 작성하기 때문에 이름없는 Named Query에 가깝다
* JPA NamedQuery처럼 어플리케이션 실행 시점에 문법 오류를 발견할 수 있다.
* 실무에서 가장 많이 사용한다.

### @Query, 값, DTO 조회하기
#### 단순히 값 하나 조회
```java
@Query("select m.username from Member m")
List<String> findUsernameList();
```
JPA값 타입 @Enbedded도 이방식으로 조회 가능
#### DTO로 직접 조회
```java
@Query("select new study.datajpa.dto.MemberDto(m.id, m.username, t.name) " + "from Member m join m.team t")
List<MemberDto> findMemberDto();
```
DTO로 직접 조회 하려면 JPA의 new 명령어를 사용해야 한다. 그리고 생성자가 맞는 DTO가 필요하다. (JPA와 사용방식이 동일)
