# 스프링부트와 JPA 활용

> [[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발]](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1/dashboard) 강의 학습 내용입니다.

### 프로젝트 환경 세팅
* project : Gradle - Groovy Project
* 버전정보 : java 11, 스프링부트 2.7.7, h2 2.1.214
* 사용 기능 : web, thymeleaf, jpa, h2, lombok, validation
  * groupId : jpabook
  * artifactId : jpashop

### JPA와 DB 설정

```java
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        format_sql: true

logging.level:
  org.hibernate.SQL: debug
  org.hibernate.type: trace
```
* `org.hibernate.SQL:` 옵션을 통해 logger로 하이버네이트 실행 SQL을 남겨준다.
* `yml` 파일은 띄어쓰기(스페이스) 2칸으로 계층을 만들기 때문에 띄어쓰기 2칸을 필수로 적어주어야 한다.
* 쿼리 파라미터 로그 남기기 위해서 `org.hibernate.type: trace` 추가

#### 실제 동작 확인
회원 엔티티
```java
@Entity
@Getter @Setter
public class Member {

    @Id @GeneratedValue
    private Long id;
    private String username;

}
```

회원 레파지토리
```java
@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager em;

    public Long save(Member member){
        em.persist(member);
        return member.getId();
    }

    public Member find(Long id){
        return em.find(Member.class, id);
    }
}
```

테스트
```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class MemberRepositoryTest {

    @Autowired
    MemberRepository memberRepository;

    @Test
    @Transactional
    @Rollback(value = false)
    void testMember() {
        Member member = new Member();
        member.setUsername("memberA");

        Long savedId = memberRepository.save(member);
        Member findMember = memberRepository.find(savedId);

        assertThat(findMember.getId()).isEqualTo(member.getId());
        assertThat(findMember.getUsername()).isEqualTo(member.getUsername());
        assertThat(findMember).isEqualTo(member);
    }
}
```
* `@Transactional` 어노테이션이 테스트케이스에 있으면 테스트 끝난 뒤 반복적인 테스트를 위해 DB 롤백하기 때문에 `@Rollback(value = false)` 추가하여 실제 DB 저장 여부 확인
