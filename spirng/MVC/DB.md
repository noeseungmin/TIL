# DB
## JDBC
JDBC(Java Database Connectivity)는 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다. JDBC는 데이터베이스에서 자료를 쿼리하거나 업데이트하는 방법을 제공한다
#### 등장 배경
애플리케이션 개발시 중요한 데이터는 대부분 데이터 베이스에 보관하는데 클라이언트가 애플리케이션 서버를 통해 데이터를 저장하거나 조회시 앺플리케이션 서버는 다음과 같은 과정을 통해 데이터베이스를 사용한다.
* 커넥션 연결: TCP/IP를 이용해 커넥션 연결
* SQL 전달: 애플리케이션 서버는 DB가 이해할 수 있는 SQL을 연결된 커넥션을 통해 DB에 전달
* 결과 응답: DB는 전달된 SQL을 수행하고 그 결과를 응답한다. 애플리케이션 서버는 응답 결과를 활용한다.
관계형 DB는 각각 커넥션 연결, SQL 전달방식 결과를 얻는 방법이 모두 달라 DB변경시 애플리케이션 서버에 개발된 데이터 베이스 코드도 변경해 줘야하며 개발자가 각 데이터베이스마다 커넥션 연결, SQL 전달 및 응답 결과를 받는 방법을 새로 배워야 한다.
이러한 불편함 때문에 JDBC라는 자바 표준 기술이 등장하게 되었다!

### JDBC 표준 인터페이스
대표적으로 다음 3가지 기능을 표준 인터페이스로 정의해서 제공한다.
* java.sql.Connection - 연결
* java.sql.Statement - SQL을 담은 내용
* java.sql.ResultSet - SQL 요청 응답
이 JDBC 인터페이스를 각각의 DB 벤더 (회사)에서 자신의 DB에 맞도록 구현해서 라이브러리로 제공하는 JDBC 드라이버를 가져다 쓰면 된다.
EX) 현재 Oracle JDBC 드라이버를 사용해  Oracle JDBC 데이터베이스를 사용중에 만약 MySQL로 변경한다면 MySQL JDBC 드라이버로 변경만 해주면 된다.

JDBC의 등장으로 많은 것이 편리해졌지만, 각각의 데이터베이스마다 SQL, 데이터타입 등의 일부 사용법 다르다. 데이터베이스를 변경하면 JDBC 코드는 변경하지 않아도 되지만 SQL은 해당 데이터베이스에 맞도록 변경해야한다.

### JDBC 최신 데이터 접근 기술
JDBC를 직접 사용하는 것 보다 JDBC를 좀 더 편리하게 사용할 수 있도록 하는 기술들을 사용하는데 대표적 기술에는 SQL Mapper, ORM 이 있다.

#### SQL Mapper
대표 기술: 스프링 JdbcTemplate, MyBatis
* 장점: JDBC를 편리하게 사용하도록 도와준다.
  * SQL 응답 결과를 객체로 편리하게 변환해준다.
  * JDBC의 반복 코드를 제거해준다.
* 단점: 개발자가 SQL을 직접 작성해야한다.

#### ORM
ORM은 객체를 관계형 데이터베이스 테이블과 매핑해주는 기술이다. 앞서 나온 번거로움들이 사라지고 ORM기술이 SQL을 동적으로 만들어 실행해준다. 그리고 데이터베이스마다 다른 SQL을 사용하더라도 알아서 해결해준다. 
Java에서는 JPA 인터페이스가 있으며 구현체로는 하이버네이트, 이클립스링크 등이 있다.

#### SQL Mapper vs ORM 기술
SQL Mapper, ORM은 각각 장, 단점을 갖고 있다.
먼저 SQL Mapper는 SQL만 직접 작성하면 나머지 일들은 SQL Mapper가 대신 해결해준다. 또한 SQL만 알면 되므로 금방 배워서 사용 가능하다.
반면에, ORM은 SQL을 사용하지 않아도 되기 때문에 개발 생산성을 엄청나게 높일 수 있다. 하지만 쉬운 기술이 아니기 때문에 깊이있게 공부해야 실무에 적용할 수 있다.

## DB연결
### JDBC DriverManager
데이터 베이스에 연결시 JDBC가 제공하는 `DriverManager.getConnection(..)`를 사용하면. 라이브러리에 있는 데이터베이스 드라이버를 찾아서 해당 드라이버가 제공하는 커넥션을 반환해준다.

#### 커넥션 요청 흐름
JDBC가 제공하는 `DriverManager` 는 라이브러리에 등록된 DB 드라이버들을 관리하고, 커넥션을획득하는 기능을 제공한다.

애플리케이션 로직에서 커넥션이 필요할때 `DriverManager.getConnection()`을 호출하면 DriverManager 는 라이브러리에 등록된 드라이버 목록을 자동으로 인식한다. 이 드라이버들에게 순서대로 다음 정보를 넘겨서 커넥션을 획득할 수 있는지 확인한다.
  * URL: 예) jdbc:h2:tcp://localhost/~/test
  * 이름, 비밀번호 등 접속에 필요한 추가 정보
  * 여기서 각각의 드라이버는 URL 정보를 체크해서 본인이 처리할 수 있는 요청인지 확인한다. URL이 jdbc:h2 로 시작하면 이것은 h2 데이터베이스에 접근하기 위한 규칙이며 H2 드라이버는 본인이 처리할 수 있기 때문에 실제 데이터베이스에 연결해서 커넥션을 획득하고 이 커넥션을 클라이언트에 반환한다. 반면에 URL이 jdbc:h2 로 시작했는데 MySQL 드라이버가 먼저 실행되면 이 경우 본인이 처리할 수 없다는 결과를 반환하게 되고, 다음 드라이버에게 순서가 넘어간다.
* 이렇게 찾은 커넥션 구현체가 클라이언트에 반환된다.
* H2 데이터베이스 드라이버만 라이브러리에 등록했기 때문에 H2 드라이버가 제공하는 H2 커넥션을 제공받는다.

### JDBC 개발
#### save() - 저장
```java
private Connection getConnection() {
 return DBConnectionUtil.getConnection();
 }
 ```
`getConnection()` : 이전에 만들어둔 DBConnectionUtil 를 통해서 데이터베이스 커넥션을 획득한다.

```java
public Member save(Member member) throws SQLException {
 String sql = "insert into member(member_id, money) values(?, ?)";
 Connection con = null;
 PreparedStatement pstmt = null;
 try {
 con = getConnection();
 pstmt = con.prepareStatement(sql);
 pstmt.setString(1, member.getMemberId());
 pstmt.setInt(2, member.getMoney());
 pstmt.executeUpdate();
 return member;
 } catch (SQLException e) {
 log.error("db error", e);
 throw e;
 } finally {
 close(con, pstmt, null);
 }
 }
 ```
* `sql` : 데이터베이스에 전달할 SQL을 정의한다. 데이터 등록을 위해 insert sql
`con.prepareStatement(sql)` : 데이터베이스에 전달할 SQL과 파라미터로 전달할 데이터들을 준비한다.
  * sql : insert into member(member_id, money) values(?, ?)"
  * pstmt.setString(1, member.getMemberId()) : SQL의 첫번째 ? 에 값 지정, 문자이므로 `setString`
  * pstmt.setInt(2, member.getMoney()) : SQL의 두번째 ? 에 값을 지정, Int 형 숫자이므로 `setInt` 
* pstmt.executeUpdate() : Statement 를 통해 준비된 SQL을 커넥션을 통해 실제 데이터베이스에 전달한다. 참고로 executeUpdate() 은 int 를 반환하는데 영향받은 DB row 수를 반환한다. 여기서는 하나의 row를 등록했기 때문에 1 반환

#### findById() - 조회
`String sql = "select * from member where member_id = ?";` 
`rs = pstmt.executeQuery();` : 데이터를 변경할 때는 `executeUpdate()` 를 사용하지만, 데이터를 조회할 때는 `executeQuery()` 를 사용한다. `executeQuery()` 는 결과를 `ResultSet` 에 담아서 반환한다.

#### ResultSet
ResultSet은 select 쿼리의 결과가 순서대로 들어간다 .
* `select member_id, money` 라고 지정하면 member_id , money 라는 이름으로 데이터가 저장되며 `select *` 을 사용하면 테이블의 모든 컬럼을 다 지정한다.
* ResultSet은 내부에 있는 커서( cursor )를 이동해서 다음 데이터를 조회할 수 있다.
  * `rs.next()` : 이것을 호출하면 커서가 다음으로 이동한다. 최초의 커서는 데이터를 가리키고 있지 않기 때문에 rs.next() 를 최초 한번은 호출해야 데이터를 조회할 수 있다.
  * rs.next() 의 결과가 true 면 커서의 이동 결과 데이터 존재, rs.next() 의 결과가 false 면 더이상 커서가 가리키는 데이터 미존재.
* rs.getString("member_id") : 현재 커서가 가리키고 있는 위치의 member_id 데이터를 String타입으로 반환한다.
* rs.getInt("money") : 현재 커서가 가리키고 있는 위치의 money 데이터를 int 타입으로 반환한다.

#### update(), delete() - 수정, 삭제
수정과 삭제는 등록과 비슷하다. 등록, 수정, 삭제처럼 데이터를 변경하는 쿼리는 executeUpdate() 를 사용하면 된다


## 커넥션풀과 데이터소스
### 커넥션 풀의 이해
JDBC 를 사용한다면 데이터베이스에 접근할 때 마다 매번 커넥션을 획득해야하고 다음과 같은 복잡한 커넥션 과정을 거쳐야한다.
* 애플리케이션 로직은 DB 드라이버를 통해 커넥션을 조회한다.
* DB 드라이버는 DB와 TCP/IP 커넥션을 연결한다. 물론 이 과정에서 3 way handshake 같은 TCP/IP 연결을 위한 네트워크 동작이 발생한다.
* DB 드라이버는 TCP/IP 커넥션이 연결되면 ID, PW와 기타 부가정보를 DB에 전달한다.
* DB는 ID, PW를 통해 내부 인증을 완료하고, 내부에 DB 세션을 생성한다.
* DB는 커넥션 생성이 완료되었다는 응답을 보낸다.
* DB 드라이버는 커넥션 객체를 생성해서 클라이언트에 반환한다.
#### 문제점
커넥션을 새로 만드는 것은 복잡한 과정 + 많은 시간 소요되며 DB는 물론이고 애플리케이션 서버에서도 TCP/IP 커넥션을 새로 생성하기 위한 리소스를 매번 사용해야 한다.

#### 해결방안
이런 문제를 한번에 해결하는 아이디어가 바로 커넥션을 미리 생성해두고 사용하는 커넥션 풀이라는 방법이다.
커넥션 풀은 이름 그대로 커넥션을 관리하는 풀(수영장 풀을 상상하면 된다.)이다.

* 적절한 커넥션 풀 숫자는 서비스의 특징과 애플리케이션 서버 스펙, DB 서버 스펙에 따라 다르기 때문에 성능 테스트를 통해서 정해야 한다.
* 커넥션 풀은 서버당 최대 커넥션 수를 제한할 수 있다. 따라서 DB에 무한정 연결이 생성되는 것을 막아주어서 DB를 보호하는 효과도 있다.
* 대표적인 커넥션 풀 오픈소스는 commons-dbcp2 , tomcat-jdbc pool , HikariCP 등이 있다.
* 성능과 사용의 편리함 측면에서 최근에는 hikariCP 를 주로 사용한다. 스프링 부트 2.0 부터는 기본 커넥션 풀로 hikariCP 를 제공한다. 성능, 사용의 편리함, 안전성 측면에서 이미 검증이 되었기 때문에 커넥션 풀을 사용할 때는 고민할 것 없이 hikariCP 를 사용하면 된다. 실무에서도 레거시 프로젝트가 아닌 이상 대부분 hikariCP 를 사용한다.

### 데이터소스
* 대부분의 커넥션 풀은 DataSource 인터페이스가 구현 되어 있다. 따라서 DataSource 인터페이스에만 의존하도록 애플리케이션 로직을 작성하면 된다. 
커넥션 풀 구현 기술을 변경하고 싶으면 해당 구현체로 갈아끼우기만 하면 된다.
* `DriverManager`는 `DataSource` 인터페이스를 사용하지 않는다. 따라서 `DriverManager`는 직접 사용해야 합니다.
* `DataSource` 기반의 커넥션 풀 ↔️ `DriverManager` 로 변경한다면 관련 코드를 수정해야합니다.

이런 문제를 해결하기 위해 스프링은 `DriverManager` 도 `DataSource` 를 통해서 사용할 수 있도록 `DriverManagerDataSource` 라는 `DataSource` 를 구현한 클래스를 제공한다.

### 데이터 소스 예제1 - DriverManager
#### 드라이버 매니저
```java
public class ConnectionTest {
 @Test
 void driverManager() throws SQLException {
 Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
 Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
 log.info("connection={}, class={}", con1, con1.getClass());
 log.info("connection={}, class={}", con2, con2.getClass());
 }
}
```
#### 데이터소스 드라이버 매니저
```java
@Test
 void dataSourceDriverManager() throws SQLException {
 //DriverManagerDataSource - 항상 새로운 커넥션 획득
 DriverManagerDataSource dataSource = new DriverManagerDataSource(URL,
USERNAME, PASSWORD);
 useDataSource(dataSource);
 }
 private void useDataSource(DataSource dataSource) throws SQLException {
 Connection con1 = dataSource.getConnection();
 Connection con2 = dataSource.getConnection();
 log.info("connection={}, class={}", con1, con1.getClass());
 log.info("connection={}, class={}", con2, con2.getClass());
 }
}
```
`DriverManagerDataSource`는 스프링이 제공하는 코드로 `DataSource`를 통해서 커넥션을 획득할 수있다.
* 차이점
`DriverManager` 는 커넥션을 획득할 때 마다 파라미터를 계속 전달해야 한다. 반면에 `DataSource`를 사용하는 방식은 처음 객체를 생성할 때만 필요한 파리미터를 넘겨두고, 커넥션을 획득할 때는 단순히`dataSource.getConnection()` 만 호출하면 된다
* 설정과 사용의 분리
 * 설정 : DataSource 를 만들고 필요한 속성들을 사용해서 `URL , USERNAME , PASSWORD` 같은 부분을 입력하는 것을 말한다. 
 * 사용 : 설정은 신경쓰지 않고, `DataSource` 의 `getConnection()` 만 호출해서 사용하면 된다.

필요 데이터를 `DataSource` 가 만들어지는 시점에 미리 다 넣어두게 되면 속성들에 의존하지 않아도 되서 DataSource만 주입받아 `getConnection()`을 호출하면된다. 리포지토리(Repository)는 DataSource 만 의존하고 덕분에 객체를 설정하는 부분과, 사용하는 부분을 좀 더 명확하게 분리할 수 있다.

### 데이터 소스 예제1 - 커넥션 풀
```java
@Test
void dataSourceConnectionPool() throws SQLException, InterruptedException {
 //커넥션 풀링: HikariProxyConnection(Proxy) -> JdbcConnection(Target)
 HikariDataSource dataSource = new HikariDataSource();
 dataSource.setJdbcUrl(URL);
 dataSource.setUsername(USERNAME);
 dataSource.setPassword(PASSWORD);
 dataSource.setMaximumPoolSize(10);
 dataSource.setPoolName("MyPool");
 useDataSource(dataSource);
 Thread.sleep(1000); //커넥션 풀에서 커넥션 생성 시간 대기
}
```
* `HikariDataSource` 는 `DataSource` 인터페이스를 구현하고 있다.
* 풀 이름은 Mypool, 사이즈는 최대 10으로 지정 해주었다.
* 커넥션 풀에서 커넥션을 생성하는 작업은 애플리케이션 실행 속도에 영향을 주지 않기 위해 별도의 쓰레드에서 동작하기 때문에 테스트가 먼저 종료되어 버린다. Thread.sleep 을 통해 대기 시간을 주어야 쓰레드 풀에 커넥션이 생성되는 로그를 확인할 수 있다.

#### MyPool connection adder
별도의 쓰레드 사용해서 커넥션 풀에 커넥션을 채워준다. 이 쓰레드는 커넥션 풀에 커넥션을 최대 풀 수( 10 )까지 채운다.
애플리케이션을 실행할 때 커넥션 풀을 채울 때 까지 마냥 대기하고 있다면 애플리케이션 실행 시간이 늦어진다. 따라서 이렇게 별도의 쓰레드를 사용해서 커넥션 풀을 채워야 애플리케이션 실행 시간에 영향을 주지 않는다.

### 데이터 소스 적용
```java
private final DataSource dataSource;
 public MemberRepositoryV1(DataSource dataSource) {
 this.dataSource = dataSource;
 }
//....
 private void close(Connection con, Statement stmt, ResultSet rs) {
 JdbcUtils.closeResultSet(rs);
 JdbcUtils.closeStatement(stmt);
 JdbcUtils.closeConnection(con);
 }
 private Connection getConnection() throws SQLException {
 Connection con = dataSource.getConnection();
 log.info("get connection={}, class={}", con, con.getClass());
 return con;
 }
 ```
* `DataSource` 의존관계 주입
  * 외부에서 `DataSource` 를 주입 받아서 사용한다. 이제 직접 만든 `DBConnectionUtil` 을 사용하지 않아도 된다.
  * `DataSource` 는 표준 인터페이스 이기 때문에 `DriverManagerDataSource` 에서 `HikariDataSource` 로 변경되어도 해당 코드를 변경하지 않아도 된다.
* `JdbcUtils` 편의 메서드 : 스프링은 JDBC를 편리하게 다룰 수 있는 `JdbcUtils` 라는 편의 메서드를 제공한다.

## 트랜잭션
데이터베이스에서 트랜잭션은 하나의 거래를 안전하게 처리하도록 보장해주는 것을 뜻한다. 2가지 작업을 하나의 작업처럼 동작해야 할때 데이터베이스가 제공하는 트랜잭션 기능을 사용하면 둘다 함께 성공해야 저장하고 중간에 하나라도 실패하면 거래 전의 상태로 돌아갈 수 있다. 
* `Commit` : 모든 작업이 성공해서 데이터베이스에 정상 반영
* `Rollback` : 작업 중 하나라도 실패해서 거래 이전으로 되돌리는것

### 트랜잭션 ACID
트랜잭션은 원자성, 일관성, 격리성, 지속성을 보장해야한다.
* 원자성(Atomicity) : 마치 하나의 작업인 것처럼 모두 성공 하거나 실패해야 한다.
* 일관성(Consistency) : 일관성 있는 데이터베이스 상태를 유지해야 한다. 데이터베이스에서 정한 무결성 제약 조건을 항상 만족해야 한다.
* 격리성(Isolation) : 동시에 실행되는 트랜잭션이 서로에게 영향을 미치지 못하게 격리한다. 동시성과 관련된 성능 이슈로 인해 트랜잭션 격리수준을 선택 가능하다.
* 지속성(Durability) : 성공적으로 끝내면 그 결과가 항상 기록되어야 한다. 중간에 시스템엥 문제가 생겨도 데이터베이스 로그등를 사용해서 공공한 트랜잭션 내용을 복구해야 한다.

트랜잭션은 격리성을 완벽히 보장하려면 거의 순서대로 실행해야 한다. 이렇게 처리시 성능이 매우 나빠지므로 ANSI표준은 4단계로 나누어 정의한다.

트랜잭션 격리 수준 - Isolation level
* READ UNCOMMITED(커밋되지 않은 읽기) 
* READ COMMITTED(커밋된 읽기) 
* REPEATABLE READ(반복 가능한 읽기) 
* SERIALIZABLE(직렬화 가능) 
