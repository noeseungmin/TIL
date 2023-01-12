## Bean Validation
* Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다. 
* 검증 애노테이션과 여러 인터페이스의 모음이다. 마치 JPA가 표준 기술이고 그 구현체로 하이버네이트가 있는 것과 같다.
* Bean Validation을 구현한 기술중에 일반적으로 사용하는 구현체는 하이버네이트 Validator이다. 이름이 하이버네이트가 붙어서 그렇지 ORM과는 관련이 없다.
```java
public class Item {
 private Long id;

 @NotBlank
 private String itemName;
 
 @NotNull
 @Range(min = 1000, max = 1000000)
 private Integer price;
 
 @NotNull
 @Max(9999)
 private Integer quantity;
 //...
}
```
이런 검증 로직을 모든 프로젝트에 적용할 수 있게 공통화하고, 표준화 한 것, Bean Validation을 잘 활용하면, 애노테이션 하나로 검증 로직을 매우 편리하게 적용할 수 있다.

* `@NotBlank : 빈값 + 공백만 있는 경우를 허락하지 않는다.
* `@NotNull` : null을 허용 X
* `@Range(min = 1000, max = 10000)` : 범위 안의 값
* `@Max(9999) : 최대 9999까지만 허용한다.

### 의존관계 추가
Bean Validation을 사용하려면 다음 의존관계를 추가해야 한다.
`implementation 'org.springframework.boot:spring-boot-starter-validation'`

#### 스프링 부트 BeanValidator
스프링 부트는 위의 의존관계 추가시 자동으로 Bean Validator를 인지하고 스프링에 통합한다.
* `@Validated`는 스프링 전용 애노테이션이고 내부에 group 기능을 포함한다. 
* `@Valid는 자바 표준 검증 애노테이션이다. 둘중 아무거나 사용해도 동일하게 작동한다.

#### 스프링 부트는 자동으로 글로벌 Validator로 등록한다.
LocalValidatorFactoryBean 을 글로벌 Validator로 등록한다. 이 Validator는 @NotNull 같은 애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에, @Valid , @Validated 만 적용하면 된다.
검증 오류가 발생하면, FieldError , ObjectError 를 생성해서 BindingResult 에 담아준다.

### 에러 코드
Bean Validation 오류 메시지를 좀 더 자세히 변경하기
`NotBlank`라는 오류 코드를 기반으로 MessageCodesResolver를 통해 다양한 메시지 코드가 순서대로 생성된다.

`@NotBlank`
* NotBlank.item.itemName
* NotBlank.itemName
* NotBlank.java.lang.String
* NotBlank

`@Range`
* Range.item.price
* Range.price
* Range.java.lang.Integer
* Range 

#### 메시지 등록
```
//errors.properties
required = 필수 값 입니다.
min= {0} 이상이어야 합니다.
range= {0} ~ {1} 범위를 허용합니다.
max= {0} 까지 허용합니다.
```
{0}은 필드명, {1},{2}는 각 애노테이션 마다 다르다.

`@NotBlank(message = "공백은 입력할 수 없습니다.")` 애노테이션 message 사용

#### 메시지 찾는 순서
생성된 메시지 코드 순서대로 messageSource 에서 메시지 찾기 -> 애노테이션의 message 속성 사용 -> `@NotBlank(message = "공백! {0}")` -> 라이브러리가 제공하는 기본 값 사용 공백일 수 없습니다.

### 오브젝트 오류
FiledError가 아닌 ObjectError는 어떻게 처리 할까?
`@ScriptAssert()`와 자바 코드 작성 방법이 있다.

#### @ScriptAsser()
`@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000")`
실제 사용시 제약이 많고 복잡하며 실무에서는 검증 기능이 해당 객체의 범위를 넘어서는 경우들이 종종 등장하는 데 그럴 경우 대응이 어렵다.

#### 자바 코드 작성
```java
 if (item.getPrice() != null && item.getQuantity() != null) {
 int resultPrice = item.getPrice() * item.getQuantity();
 if (resultPrice < 10000) {
 bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
 }
```
다음과 같이 오브젝트 오류의 경우 @ScriptAssert를 억지로 사용하는 것 보다 오류 관련 부분만 자바 코드로 작성하는 것을 권장한다.

