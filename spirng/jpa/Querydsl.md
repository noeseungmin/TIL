# Querydsl

### 설정
스프링 부트 2.6이상 Querydsl 5.0
build.gradle
```
buildscript {
 ext {
 queryDslVersion = "5.0.0"
 }
}
plugins {
/...
//querydsl 추가
 id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}
dependencies {
//querydsl 추가
 implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
 annotationProcessor "com.querydsl:querydsl-apt:${queryDslVersion}"
 //..
 }
 
 //querydsl 추가 시작
def querydslDir = "$buildDir/generated/querydsl"
querydsl {
 jpa = true
 querydslSourcesDir = querydslDir
}
sourceSets {
 main.java.srcDir querydslDir
}
configurations {
 querydsl.extendsFrom compileClasspath
}
compileQuerydsl {
 options.annotationProcessorPath = configurations.querydsl
}
```
#### 검증용 엔티티 생성
```java
@Entity
@Getter @Setter
public class Hello {
 @Id @GeneratedValue
 private Long id;
}
```
* Q타입 생성
Gradle -> Tasks -> build -> clean
Gradle -> Tasks -> other -> compileQuerydsl
* 생성 확인
build -> generated -> querydsl -> `study.querydsl.entity.QHello.java` 파일이 생성 되어있으면 된다.

## 기본 문법
#### 기본 Q-Type 활용
```java
QMember qMember = new QMember("m"); //별칭 직접 지정
QMember qMember = QMember.member; //기본 인스턴스 사용
```
```java
import static study.querydsl.entity.QMember.* //static import -> member == QMember.member
public void findMember1() {
Member findMember = queryFactory
 .select(member) 
 .from(member)
 .where(member.username.eq("member1"))
 .fetchOne();
```
### 검색 조건 쿼리
#### 기본 검색
```java
public void search() {
 Member findMember = queryFactory
 .selectFrom(member)
 .where(member.username.eq("member1")
 .and(member.age.eq(10)))
 .fetchOne();
```
검색 조건은 .and(), or()를 메서드 체인으로 연결 가능하며 select, from을 selectFrom으로 합치는것도 가능하다.

#### AND 조건을 파라미터로 처리
```java
public void searchAndParam() {
 List<Member> result1 = queryFactory
 .selectFrom(member)
 .where(member.username.eq("member1"),
 member.age.eq(10))
 .fetch();
 assertThat(result1.size()).isEqualTo(1);
}
```
where()에 파라미터로 검색 조건을 추가하면 AND 조건이 추가되며 이경우 null값 무시 메서드 추출을 활용해서 동적 쿼리를 깔끔하게 만들 수 있다.

### 결과 조회
```java
//List
List<Member> fetch = queryFactory
 .selectFrom(member)
 .fetch(); //리스트 조회 데이터 없으면 빈 리스트 반환

//단 건
Member findMember1 = queryFactory
 .selectFrom(member)
 .fetchOne(); // 단건 조회 결과가 없으면 null, 둘이상이면 NonUniqueResultException

//처음 한 건 조회
Member findMember2 = queryFactory
 .selectFrom(member)
 .fetchFirst(); // == limit(1).fetchOne()

//페이징에서 사용
QueryResults<Member> results = queryFactory
 .selectFrom(member)
 .fetchResults(); //페이징 정보 호함 total count 쿼리 추가 실행

//count 쿼리로 변경
long count = queryFactory
 .selectFrom(member)
 .fetchCount();  //count 쿼리로 변경해서 count 수 조회
 ```
 
 ### 정렬
 ```java
 List<Member> result = queryFactory
 .selectFrom(member)
 .where(member.age.eq(100))
 .orderBy(member.age.desc(), member.username.asc().nullsLast())
 .fetch();
 ```
 * age = DESC(내림차순), username = ASC(오름차순)
 * nullLast(), nullFirst(): null데이터 순서 부여

### 페이징
```java
public void paging() {
 QueryResults<Member> queryResults = queryFactory
 .selectFrom(member)
 .orderBy(member.username.desc())
 .offset(1) //페이징 시작점 지정(0부터 시작)
 .limit(2) //페이징 크기
 .fetchResults();
```
`fetchREsults()` 사용시 자동으로 Count 쿼리를 날리기 떄문에 성능에 영향을 미치는 경우가 있다. Count 쿼리를 생략할 수 있는 경우는 두가지 경우가 있다.
* 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때 (즉 하나의 페이지에 100개의 컨텐츠를 보여주는데, 총 데이터가 100개가 안되는 경우)
* 마지막 페이지일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈를 구함)

### 그룹화
```java
public void group() throws Exception {
 List<Tuple> result = queryFactory
   .select(team.name, member.age.avg())
   .from(member)
   .join(member.team, team)
   .groupBy(team.name)
   .having() // groupBy의 조건
   .fetch();
 ```
 groupBy 명령어를 통해 특정 컬럼을 기준으로 그룹화가 가능. 조건을 주기 위해 where이 아닌 having절 사용한다.
