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



