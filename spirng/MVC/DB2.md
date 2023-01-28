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


## MyBatis
MyBatis는 JdbcTemplate보다 많은 기능을 제공하는 SQL Mapper이다. SQL을 XML에 편리하게 작성할 수 있고 동적 쿼리를 매우 편리하게 작성 할 수 있다.

#### 설정
`implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'` 다음과 같은 라이브러리가 추가된다.
* mybatis-spring-boot-starter : MyBatis를 스프링 부트에서 편리하게 사용할 수 있게 시작하는 라이브러리
* mybatis-spring-boot-autoconfigure : MyBatis와 스프링 부트 설정 라이브러리
* mybatis-spring : MyBatis와 스프링을 연동하는 라이브러리
* mybatis : MyBatis 라이브러리
 
 ```java
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```
* `mybatis.type-aliases-package` 마이바티스에서 타입 정보를 사용할 때는 패키지 이름을 적어주어야 하는데, 여기에 명시하면 패키지 이름 생략가능, 지정한 패키지와 그 하위 패키지 자동인식, 여로위치를 지정하려면 ,(콤마), ;(세미콜론)로 구분 하면된다.
* `mybatis.configuration.map-underscore-to-camel-case` JdbcTemplate의 BeanPropertyRowMapper 에서 처럼 언더바를 카멜로 자동 변경해주는 기능을 활성화 한다.
* `logging.level.hello.itemservice.repository.mybatis=trace` MyBatis에서 실행되는 쿼리 로그를 확인할 수 있다

#### 관례의 불일치
자바 객체에는 주로 카멜표기법을 사용(itemName) 중간에 낙타 봉이 올라와 있는 표기법, 반면 관계형 데이터베으슨는 스네이크표기법을 사용(item_name) 중간에 언더스코어를 사용한다. `map-underscore-to-camel-case` 기능을 활성화 하면  select item_name으로 조회해도 객체의 itemName(setItemName()) 속성에 값이 정상 입력된다. 컬럼이름과 객체 이름이 완전히 다른 경우에는 조회 SQL에서 별칭을 사용하도록 하자


### 적용 - 기본1
```java
@Mapper
public interface ItemMapper {
 void save(Item item);
 void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto 
updateParam);
 Optional<Item> findById(Long id);
 List<Item> findAll(ItemSearchCond itemSearch);
}
```
`@Mapper` 애노테이션을 붙여주어야 MyBatis에서 인식할 수 있다. 메서드를 호출하면 같은 위치(같은 패키지)에 실행할 SQL이 있는 xml의 해당 SQL을 실행하고 결과를 돌려준다.

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
 "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">
 <insert id="save" useGeneratedKeys="true" keyProperty="id">
 //...
 </insert>
 <update id="update">
  //...
 </update>
 <select id="findById" resultType="Item">
 //...
 </select>
 <select id="findAll" resultType="Item">
 //...
 </select>
</mapper>
```
* XML경로 수정을원하면 application.properties에 `mybatis.mapper-locations=classpath:mapper/**/*.xml` 이렇게 할 경우 resources/mapper를 포함한 그 하위 폴더에 있는 XML을 XML매핑 파일로 인식한다. 

#### insert - save
```java
<insert id="save" useGeneratedKeys="true" keyProperty="id">
 insert into item (item_name, price, quantity)
 values (#{itemName}, #{price}, #{quantity})
</insert>
```
* id에는 매퍼 인터페이스에 설정한 메서드 이름을 지정한다. 메서드 이름으 save()이기 때문에 save로 지정
* 파라미터는 `#{}` 문법을 사용하면 PreparedStatement를 사용한다. JDBC의 ? 를 치환해주는 역할, 매퍼에 넘긴 객체의 프로퍼티 이름을 적어준다. 
* userGeneatedKeys는 데이터베이스가 키를 생성해주는 IDENTITY 전략일 때 사용되며 KeyPropert는 생성되는 키의 속성을 지정한다. Insert가 끝나면 item 객체의 id 속성에 생성된 값이 입력된다.

#### update - update
```java
<update id="update">
 update item
 set item_name=#{updateParam.itemName},
 price=#{updateParam.price},
 quantity=#{updateParam.quantity}
 where id = #{id}
</update>
```
* 파라미터가 `Long id`, `ItemUpdateDto updateParam`으로 파라미터가 2개이상이면 `@Param`으로 이름을 지정해서 파라미터 구분이 필요하다.

#### select - findById
```java
<select id="findById" resultType="Item">
 select id, item_name, price, quantity
 from item
 where id = #{id}
</select>
```
* `resultType`은 반환 타입을 명시해줘야 한다. 여기서는 결과를 Item객체에 매핑한다.
* `application.properties`에 `mybatis.type-aliases-package=hello.itemservice.domain` 속성을 지정해주게 되면 패키지 명을 다 적지 않아도 된다.
* `mybatis.configuration.map-underscore-to-camel-case=true` 속성을 미리 지정 해주었기 떄문에 언더스코어를 카멜 표기법으로 자동으로 처리해준다.
* 자바 코드에서 반환 객체가 하나면 Item, Optional<Item>과 같이 사용하면 되고, 하나 이상일경우 컬렉션을 사용하면 된다.(주로 List사용)
 
 #### select - findAll
 ```java
 <select id="findAll" resultType="Item">
 select id, item_name, price, quantity
 from item
 <where>
 <if test="itemName != null and itemName != ''">
 and item_name like concat('%',#{itemName},'%')
 </if>
 <if test="maxPrice != null">
 and price &lt;= #{maxPrice}
 </if>
 </where>
</select>
 ```
* MyBatis는 `<where>`, `<if>` 같은 동적 쿼리 문법을 통해 편리한 동적 쿼리를 지원한다.
* `<if>`는 해당 조건이 만족하면 구문을 추가한다.
* `<where>`은 적절하게 where문장을 만들어준다.
  * `<if>`가 모두 실패하게 되면 where을 만들지 않는다.
  * `<if>`가 하나라도 성공하면 처음 나타나는 and를 whree로 변환 해준다.
 
 ## JPA
 ```java
 @Data
@Entity
public class Item {
 @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
 private Long id;
 @Column(name = "item_name", length = 10)
 private String itemName;
 private Integer price;
 private Integer quantity;
 public Item() {
 }
 public Item(String itemName, Integer price, Integer quantity) {
 this.itemName = itemName;
 this.price = price;
 this.quantity = quantity;
 }
}
 ```
 `public Item() { }` JPA는 public 또는 protected 기본 생성자가 필수다.
* `@Entity`: JPA가 사용하는 객체라는 뜻 JPA가 인식하기 위해 필수 
* `@Id`: 테이블의 PK와 해당 필드를 매핑한다.
* `@GeneratedValue(strategy = GenerationType.IDENTITY)`: PK 생성 값을 데이터베이스에서 생성하는 IDENTITY 방식을 사용
* `@Coluimn`: 객체의 필드를 테이블의 컬럼과 매핑한다.
 * `name`: 객체는 itemName이지만 테이블 컬럼은 item_name이므로 이렇게 매핑
 * `length`: JPA매핑 정보로 DDL도 생성 할 수 있는데, 그때 컬럼의 길이 값으로 활용한다.(varchar 10)
* `@Column` 을 생략할 경우 필드의 이름을 테이블 컬럼이름으로 사용한다. 객체 필드의 카멜 케이스를 테이블 컬럼의 언더스코어로 자동으로 변환 해준다. `itemName` -> `item_name`
 
 ```java
 @Repository
@Transactional
public class JpaItemRepositoryV1 implements ItemRepository {
 private final EntityManager em;
 public JpaItemRepositoryV1(EntityManager em) {
 this.em = em;
 }
 ```
JPA의 모든 동작은 엔티티 매니저를 통해 이루어진다. 엔티티 매니저는 내부에 데이터 소스를 가지고 있고 데이터베이스에 접근 할 수 있다.
`@Transactional`: JPA의 모든 데이터 변경(등록,수정,삭제)은 트랜잭션 안에서 이루어져야 한다. JPA에서는 데이터 변경시 트랜잭션이 필수이다.
 
#### save() - 저장
```java
 public Item save(Item item) {
 em.persist(item);
 return item;
}
```
`em.persist(item)`: JPA에서 객체를 테이블에 저장할 때는 엔티티 매니저가 제공하는 persist()메서드를 사용하면 된다.

 #### update() - 수정
 ```java
 public void update(Long itemId, ItemUpdateDto updateParam) {
 Item findItem = em.find(Item.class, itemId);
 findItem.setItemName(updateParam.getItemName());
 findItem.setPrice(updateParam.getPrice());
 findItem.setQuantity(updateParam.getQuantity());
}
 ```
 JPA는 트랜잭션이 커밋되는 시점에 변경된 엔티티 객체가 있는지 확인하고 변경된 경우 UPDATE SQL을 실행 한다.
 
#### findAll() - 목록 조회
 ```java
 public List<Item> findAll(ItemSearchCond cond) {
 String jpql = "select i from Item i";
 //...
 TypedQuery<Item> query = em.createQuery(jpql, Item.class);
 return query.getResultList();
}
 ```
JPA는 JPQL이라는 객체 지향 쿼리 언어를 제공한다. SQL은 테이블을 대상, JPQL은 엔티티 객체를 대상으로 SQL을 실행
