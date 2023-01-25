# DB 2

## JDBC 템플릿
* JdbcTemplate는 spring-jdbc라이브러리에 포함되어 있으며 스프링 JDBC를 사용할 때 기본적으로 사용되는 라이브러리이므로 별도의 복잡한 설정 없이 바로 사용이 가능하다.
* JdbcTemplate은 템플릿 콜백 패턴을 사용해서, JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 처리해준다.
 * 커넥션 획득
 * statement 를 준비하고 실행
 * 결과를 반복하도록 루프를 실행
 * 커넥션 종료, statement , resultset 종료
 * 트랜잭션 다루기 위한 커넥션 동기화
 * 예외 발생시 스프링 예외 변환기 실행
`implementation 'org.springframework.boot:spring-boot-starter-jdbc'`를 추가하면 jdbcTemplate가 들어있는 spring-jdbc가 라이브러리에 포함된다. 
```java
@Repository
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {
 private final JdbcTemplate template;
 public JdbcTemplateItemRepositoryV1(DataSource dataSource) {
 this.template = new JdbcTemplate(dataSource);
 }
 ```
`JdbcTemplate`은 데이터소스가 필요하다. `JdbcTemplateItemRepositoryV1()` 생성자를 보면 `dataSource`를 의존 관계 주입받고 생성자 내부에서 `JdbcTemplate`를 생성한다. `JdbcTemplate`을 스프링 빈으로 직접 등록해서 주입 받아도 된다.

#### 저장
```java
 @Override
 public Item save(Item item) {
 String sql = "insert into item (item_name, price, quantity) values (?, ?, ?)";
 KeyHolder keyHolder = new GeneratedKeyHolder();
 template.update(connection -> {
 //자동 증가 키
 PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
 ps.setString(1, item.getItemName());
 ps.setInt(2, item.getPrice());
 ps.setInt(3, item.getQuantity());
 return ps;
 }, keyHolder);
 long key = keyHolder.getKey().longValue();
 item.setId(key);
 return item;
 }
 ```
 데이터 저장시 PK생성을 identity방식을 사용하기 떄문에 PK인 ID 값을  비워놓고 저장하면 데이터베이스가 PK인 ID를 대신 생성해준다. 이러한 방식은 `KeyHolder` 와 `connection.prepareStatement(sql, new String[]{"id"})` 를 사용해서 id 를 지정해주면 INSERT 쿼리 실행 이후에 데이터베이스에서 생성된 ID 값을 조회할 수 있다.
 
 ### 조회
 #### 단건 조회
 ```java
 int rowCount = jdbcTemplate.queryForObject("select count(*) from t_actor", Integer.class);
```
하나의 로우를 조회할때는 `queryForObject()`를 사용하면 된다. 객체가 아닌 단순 데이터 하나라면 `String.class`, `Integer.class`와 같이 지정해준다.
* 숫자조회, 파라미터 바인딩
```java
int countOfActorsNamedJoe = jdbcTemplate.queryForObject(
"select count(*) from t_actor where first_name = ?", Integer.class, "Joe");
```
* 문자 조회
```java
String lastName = jdbcTemplate.queryForObject(
 "select last_name from t_actor where id = ?", String.class, 1212L)
 ```
 
### 변경(INSERT, UPDATE, DELETE)
데이터를 변경할 때는 jdbcTemplate.update() 를 사용하면 된다. 참고로 int 반환값을 반환하는데, SQL 실행 결과에 영향받은 로우 수를 반환한다.
#### 등록
```java
jdbcTemplate.update(
 "insert into t_actor (first_name, last_name) values (?, ?)", "Leonor", "Watling");
 ```
#### 수정
```java
jdbcTemplate.update(
 "update t_actor set last_name = ? where id = ?", "Banjo", 5276L);
 ```
#### 삭제
```java
jdbcTemplate.update(
 "delete from t_actor where id = ?", Long.valueOf(actorId));
 ```
 
JdbcTemplate는 가장 간단하고 실용적인 방법으로 SQL을 사용한다. 하지만 동적쿼리 해결에 제한사항이 있고 SQL을 자바로 작성하기 때문에 SQL 라인이 코드를 넘어갈 때마다 문자를 더하기 해주어야한다.

