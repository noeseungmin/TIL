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

### 파라미터 바인딩
```java
select m from Member m where m.username = ?0 //위치 기반
select m from Member m where m.username = :name //이름 기반
```
가독성과 유지보수를 위해 이름 기반으로 사용하자 위치기반은 순서가 바뀌면 곤란하다.

```java
@Query("select m from Member m where m.username in :names")
List<Member> findByNames(@Param("names") List<String> names);
```
Collectio 타입으로 in절 지원

### 반환 타입
```java
List<Member> findByUsername(String name); //컬렉션
Member findByUsername(String name); //단건
Optional<Member> findByUsername(String name); //단건 Optional
```
스프링 데이터 JPA는 반환 타입이 유연하다.

조회 결과가 많거나 없으면?
* 컬렉션 : 결과 X : 빈 컬렉션 반환.
* 단건 조회 : 결과 X - Null / 결과가 2건 이상 - `NonUniqueResultException`

## 페이징과 정렬
### 순수 JPA
나이가 10살, 이름으로 내림차순, 페이지당 보여줄 데이터 3개, 첫 페이지만!
```java
public List<Member> findByPage(int age, int offset, int limit) {
 return em.createQuery("select m from Member m where m.age = :age order by m.username desc")
 .setParameter("age", age)
 .setFirstResult(offset) //몇번째?
 .setMaxResults(limit) // 몇명까지?
 .getResultList();
}
public long totalCount(int age) {
 return em.createQuery("select count(m) from Member m where m.age = :age", Long.class)
 .setParameter("age", age)
 .getSingleResult();
}
```
### 스프링 데이터 JPA
#### 페이징과 정렬 파리미터
* `org.springframework.data.domain.Sort` : 정렬 기능
* `org.springframework.data.domain.Pageable` : 페이징 기능 (내부에 Sort 포함)
#### 특별한 반환 타입
* `org.springframework.data.domain.Page` : 추가 count 쿼리 결과를 포함하는 페이징
* `org.springframework.data.domain.Slice` : 추가 count 쿼리 없이 다음 페이지만 확인 가능 (내부적으로 limit + 1조회)
* `List` (자바 컬렉션): 추가 count 쿼리 없이 결과만 반환

```java
Page<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용
Slice<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용 안함
List<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용 안함
List<Member> findByUsername(String name, Sort sort);
```
예제 실행 코드
```java
PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC,
"username"));
 Page<Member> page = memberRepository.findByAge(10, pageRequest);
 //then
 List<Member> content = page.getContent(); //조회된 데이터
 assertThat(content.size()).isEqualTo(3); //조회된 데이터 수
 assertThat(page.getTotalElements()).isEqualTo(5); //전체 데이터 수
 ```
* 두 번째 파라미터로 받은 Pageable 은 인터페이스다. 따라서 실제 사용할 때는 해당 인터페이스를 구현한 org.springframework.data.domain.PageRequest 객체를 사용한다. 
* PageRequest(현재 페이지, 조회할 데이터 수, 정렬 정보), 페이지는 0부터 시작한다.

#### count 쿼리를 다음과 같이 분리할 수 있음
```java
@Query(value = “select m from Member m”,
 countQuery = “select count(m.username) from Member m”)
Page<Member> findMemberAllCountBy(Pageable pageable);
```  
#### 페이지를 유지하면서 엔티티를 DTO로 변환하기
```java
Page<Member> page = memberRepository.findByAge(10, pageRequest);
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```

## 벌크성 수정 쿼리
#### JPA 사용
```java
public int bulkAgePlus(int age) {
 int resultCount = em.createQuery(
 "update Member m set m.age = m.age + 1" +
 "where m.age >= :age")
 .setParameter("age", age)
 .executeUpdate(); //업데이트 쿼리 나감
 return resultCount;
}
```
#### 스프링 데이터 JPA 사용
```java
@Modifying(clearAutomatically=true) //벌크성 수정, 삭제 쿼리 어노테이션
@Query("update Member m set m.age = m.age + 1 where m.age >= :age")
int bulkAgePlus(@Param("age") int age);
```

* 벌크 연산은 영속성 컨텍스트를 무시하고 실행하기 때문에 엔티티의 상태와 DB에 엔티티 상태가 달라질 수 있다.
* 벌크성 쿼리를 실행하고 나서 영속성 컨텍스트 초기화 : `clearAutomatically=true`를 넣어 줘서 정리해줘야함.
  * 이 옵션 없이 회원을 findById 로 다시 조회하면 영속성 컨텍스트에 과거 값이 남아서 문제가 될 수 있다. 만약 다시 조회해야 하면 꼭 영속성 컨텍스트를 초기화 하자. 

## @EntityGraph
연관된 엔티티들을 SQL 한번에 조회하는 방법
```java
List<Member> members = memberRepository.findAll();
 for (Member member : members) {
 member.getTeam().getName();
 }
```
* 멤버를 찾기 위해 쿼리가 한번 나감 (findAll) (1)
* 루프를 돌면서 각각의 멤버의 팀 프록시를 찾기 위한 쿼리가 한번씩 더 나감 (N)
=> N+1의 문제. 

#### JPQL 패치 조인
```java
@Query("select m from Member m left join fetch m.team")
List<Member> findMemberFetchJoin();
```
연관된 엔티티를 한번에 조회하려면 페치 조인이 필요하다.

#### EntityGraph
스프링 데이터 JPA는 `@EntityGraph`를 제공, 편리하게 사용하게 도와준다.
```java
//공통 메서드 오버라이드
@Override
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();
//JPQL + 엔티티 그래프
@EntityGraph(attributePaths = {"team"})
@Query("select m from Member m")
List<Member> findMemberEntityGraph();
//메서드 이름으로 쿼리에서 특히 편리하다.
@EntityGraph(attributePaths = {"team"})
List<Member> findByUsername(String username)
```
사실상 페치 조인(FETCH JOIN)의 간편 버전, LEFT OUTER JOIN 사용

## JPA Hint & Lock
### JPA Hint
JPA 구현체에게 제공하는 힌트
```java
@QueryHints(value = { @QueryHint(name = "org.hibernate.readOnly", value = "true")}, forCounting = true)
Page<Member> findByUsername(String name, Pageable pageable);
```
`org.springframework.data.jpa.repository.QueryHints` 어노테이션을 사용
`forCounting` : 반환 타입 Page면 count 쿼리도 쿼리 힌트 적용(기본값 true )

### Lock
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
List<Member> findByUsername(String name);
```
`org.springframework.data.jpa.repository.Lock` 어노테이션을 사용

## 확장 기능
### 사용자 정의 리포지토리 구현
* 스프링 데이터 JPA 리포지토리는 인터페이스만 정의하고 구현체는 스프링이 자동 생성
* 스프링 데이터 JPA가 제공하는 인터페이스를 직접 구현하면 구현해야 하는 기능이 너무 많음
* 다양한 이유로 인터페이스의 메서드를 직접 구현하고 싶다면?
  * JPA 직접 사용( EntityManager )
  * 스프링 JDBC Template 사용
  * MyBatis 사용
  * 데이터베이스 커넥션 직접 사용 등등...
  * Querydsl 사용
  
#### 사용자 정의 인터페이스
```java
public interface MemberRepositoryCustom {
 List<Member> findMemberCustom();
}
```

#### 사용자 정의 인터페이스 구현 클래스
```java
@RequiredArgsConstructor
public class MemberRepositoryImpl implements MemberRepositoryCustom {
 private final EntityManager em;
 @Override
 public List<Member> findMemberCustom() {
 return em.createQuery("select m from Member m")
 .getResultList();
 }
}
```
스프링 부트 2.x 부터는 `MemberRepositoryImpl` 대신에 `MemberRepositoryCustomImpl` 같이 사용자 정의 인터페이스명 + Impl 방식도 지원한다.
#### 사용자 정의 인터페이스 상속
```java
public interface MemberRepository
 extends JpaRepository<Member, Long>, MemberRepositoryCustom {
}
```

스프링 데이터 JPA가 인식해서 스프링 빈으로 등록
* 규칙: 리포지토리 인터페이스 이름 + Impl

## Auditing
엔티티 생성, 변경시 등록일 수정일 등록자 수정자를 추적하기
#### 순수 JPA 사용
```java
@MappedSuperclass
@Getter
public class JpaBaseEntity {
 @Column(updatable = false)
 private LocalDateTime createdDate;
 private LocalDateTime updatedDate;
 @PrePersist
 public void prePersist() {
 LocalDateTime now = LocalDateTime.now();
 createdDate = now;
 updatedDate = now;
 }
 @PreUpdate
 public void preUpdate() {
 updatedDate = LocalDateTime.now();
 }
}
```
### 스프링 데이터 JPA
@EnableJpaAuditing 스프링 부트 설정 클래스에 적용해야함
@EntityListeners(AuditingEntityListener.class) 엔티티에 적용
```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
@Getter
public class BaseEntity extends BaseTimeEntity{
 @CreatedBy
 @Column(updatable = false)
 private String createdBy;
 @LastModifiedBy
 private String lastModifiedBy;
``` 
```java 
public class BaseTimeEntity {
 @CreatedDate
 @Column(updatable = false)
 private LocalDateTime createdDate;
 @LastModifiedDate
 private LocalDateTime lastModifiedDate;
}
```
대부분의 엔티티는 등록시간, 수정시간이 필요하지만, 등록자, 수정자는 없을 수도 있다. 다음과 같이 Base 타입을 분리하고, 원하는 타입을 선택해서 상속한다

```java
@Bean
public AuditorAware<String> auditorProvider() {
	return () -> Optional.of(UUID.randomUUID().toString());

}
//랜덤으로 넣는 코드 실제는 세션 정보나 스프링 시큐리티 로그인 정보에서 아이디를 받음. 
```
등록자, 수정자를 처리해주는 AuditorAware를 스프링 빈으로 등록.
