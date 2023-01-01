# JPA 프로그래밍

[자바 ORM 표준 JPA 프로그래밍 - 기본편](https://www.inflearn.com/course/ORM-JPA-Basic/dashboard)

## JPA 시작

### 프로젝트 환경 세팅
* project : maven Project
* 버전정보 : java 11, 스프링부트 2.7.7, h2 2.1.214
* 사용 기능 : web, thymeleaf, jpa, h2, lombok, validation
  * groupId : jpa-basic
  * artifactId : ex1-hello-jpa

### Member 엔티티
```java
@Entity
public class Member {

    @Id
    private Long id;
    private String name;


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```
### JpaMain
```java
public class JpaMain {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello"); 

        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();


        try{
            Member member = new Member();
            member.setId(1L);
            member.setName("HelloA");

            em.persist(member);

            tx.commit();
        } catch (Exception e){
            tx.rollback();
        } finally {
            em.close();
        }

        emf.close();
    }
}
```
* 검색 : `Member findMember = em.find(Member.class, 1L)`
* 삭제 : `em.remove(findMember)`
* 수정 : `Member findMember = em.find(Member.class, 1L)`, `findMember.setName("HelloJPA)` jpa가 관리해서 값 변경 되었으면 update 쿼리 작성 해서 날려준다.
주의점! `EntityManagerFactory` 하나만 생성해서 애플리케이션 전체에서 공유, JPA의 모든 데이터 변경은 트랜잭션 안에서 실행

## JPQL
* JPA는 SQL을 추상화한 객체지향 쿼리 언어 제공
* SQL과 문법 유사 SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
* JPQL은 엔티티 객체를 대상으로 쿼리
* SQL은 데이터베이스 테이블을 대상으로 쿼리

* 가장 단순한 조회 방법
  * `EntityManager.find()
  * 객체 그래프 탐색(a.getB().getC()) 

