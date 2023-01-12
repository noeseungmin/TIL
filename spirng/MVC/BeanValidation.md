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

* `@NotBlank` : 빈값 + 공백만 있는 경우를 허락하지 않는다.
* `@NotNull` : null을 허용 X
* `@Range(min = 1000, max = 10000)` : 범위 안의 값
* `@Max(9999)` : 최대 9999까지만 허용한다.

### 의존관계 추가
Bean Validation을 사용하려면 다음 의존관계를 추가해야 한다.
`implementation 'org.springframework.boot:spring-boot-starter-validation'`

#### 스프링 부트 BeanValidator
스프링 부트는 위의 의존관계 추가시 자동으로 Bean Validator를 인지하고 스프링에 통합한다.
* `@Validated`는 스프링 전용 애노테이션이고 내부에 group 기능을 포함한다. 
* `@Valid`는 자바 표준 검증 애노테이션이다. 둘중 아무거나 사용해도 동일하게 작동한다.

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

### 한계
등록과 수정의 요구사항이 다를 수 있다.

등록시 요구사항
* 타입 검증 : 가격 수량에 문자가 들어갈시 검증 오류
* 필드 검증 : 상품명(필수, 공백X), 가격(1000이상, 백만원 이하), 수량(최대 9999)
* 특정 필드 범위 넘어서는 검증 : 가격 * 수량 = 10000원 이상

수정시 요구사항 
* id 값
* quantity 수량 제한 해제

```java
@NotNull //수정 요구사항 추가
private Long id;

@NotBlank
private String itemName;

@NotNull
@Range(min = 1000, max = 1000000)
private Integer price;

@NotNull
//@Max(9999) //수정 요구사항 추가
private Integer quantity;
```
HTTP 요청은 언제든지 악의적으로 변경해서 요청할 수 있으므로 서버에서 항상 검증해야 한다.

이러한 방법으로 작성시 수정은 정상적으로 작동하지만 등록에서 문제가 발생한다. 등록시에는 id 값이 존재하지 않아 `@NotNull` id를 적용한 것 때문에 검증에 실패하고 다시 폼 화면으로 넘어온다. quantity 수량 제한 최대값인 9999도 적용 할 수 없다.

#### 해결방법
방법 1 : BeanValidation의 groups 기능
등록시에 검증할 기능과 수정시에 검증할 기능을 각각 그룹으로 나누어 적용할 수 있다.
저장용 interface SaveCheck, 수정용 interface UpdateCheck를 생성 한다.
```java
 @NotNull(groups = UpdateCheck.class) //수정시에만 적용
 private Long id;

 @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
 private String itemName;
 
 @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
 @Range(min = 1000, max = 1000000, groups = {SaveCheck.class,UpdateCheck.class})
 private Integer price;
 
 @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
 @Max(value = 9999, groups = SaveCheck.class) //등록시에만 적용
 private Integer quantity;
 ```
 
 `ValidationItemController`
 
 저장 로직에 `@Validated(SaveCheck.class)` SaveCheck Groups 적용
 수정 로직에 `@Validated(UpdateCheck.class)` UpdateCheck.class 적용
 
* @Valid에는 groups 적용 불가 @Validated를 사용해야한다.
groups 기능을 사용해서 등록과 수정시에 각각 다르게 검증을 할 수 있었다. 그런데 groups 기능을 사용하니 Item 은 물론이고, 전반적으로 복잡도가 올라갔다. 실무에서는 회원 등록시 회원과 관련된 데이터만 전달받는 것이 아니라, 약관 정보도 추가로 받는 등 Item과 관계없는 수 많은 부가 데이터가 넘어온다. 그래서 보통 Item 을 직접 전달받는 것이 아니라, 복잡한 폼의 데이터를 컨트롤러까지 전달할 별도의 객체를 만들어서 전달한다.

방법 2 : ItemSaveForm ,ItemUpdateForm과 같은 폼 전송을 위한 별도의 모델 객체 생성
```java
@Data
public class Item {
 private Long id;
 private String itemName;
 private Integer price;
 private Integer quantity;
}
```
Item의 검증 코드를 모두 제거한다.

#### 등록
```java
@Data
public class ItemSaveForm {
 @NotBlank
 private String itemName;
 
 @NotNull
 @Range(min = 1000, max = 1000000)
 private Integer price;
 
 @NotNull
 @Max(value = 9999)
 private Integer quantity;
}
```
등록용 폼 객체를 사용하도록 컨트롤러 수정

기존 : `public String addItem(@PathVariable Long itemId, @Validated @ModelAttribute Item item, BindingResult bindingResult)` 

변경 : `public String addItem(@Validated  @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes)`

Item 대신에 ItemSaveform 을 전달 받는다. 그리고 @Validated 로 검증도 수행하고, BindingResult 로 검증 결과도 받는다.

#### 수정
```java
@Data
public class ItemUpdateForm {
 @NotNull
 private Long id;
 
 @NotBlank
 private String itemName;

 @NotNull
 @Range(min = 1000, max = 1000000)
 private Integer price;
 
 //수정에서는 수량은 자유롭게 변경할 수 있다.
 private Integer quantity;
}
```
수정용 폼 객체를 사용하도록 컨트롤러 수정

기존 : `public String edit(@PathVariable Long itemId, @Validated @ModelAttribute Item item, BindingResult bindingResult)`

변경 : `public String edit(@PathVariable Long itemId, @Validated @ModelAttribute("item") ItemUpdateForm form, BindingResult bindingResult)`

Item 대신에 ItemUpdateForm 을 전달 받는다. 그리고 @Validated 로 검증도 수행하고, BindingResult로 검증 결과도 받는다.

@ModelAttribute("item")에 item이름을 넣어준 부부은 뷰템플릿에서 접근 하는 `th:object` 이름을 item 으로 지정했기 때문에 ItemSaveForm의 규칙에 의해 MVC모델에 itemSaveForm이라는 이름으로 담기는걸 방지하기 위함이다.  

### HTTP 메시지 컨버터
* @ModelAttribute 는 HTTP 요청 파라미터(URL 쿼리 스트링, POST Form)를 다룰 때 사용한다.
* @RequestBody 는 HTTP Body의 데이터를 객체로 변환할 때 사용한다. 주로 API JSON 요청을 다룰 때 사용한다.
```java
@PostMapping("/add")
 public Object addItem(@RequestBody @Validated ItemSaveForm form, BindingResult bindingResult) {
 
 log.info("API 컨트롤러 호출");
 
 if (bindingResult.hasErrors()) {
 log.info("검증 오류 발생 errors={}", bindingResult);
 return bindingResult.getAllErrors();
}
 
 log.info("성공 로직 실행");
 return form;
}
 ```
 
#### @ModelAttribute vs @RequestBody
* HTTP 요청 파리미터를 처리하는 @ModelAttribute 는 각각의 필드 단위로 세밀하게 적용된다. 그래서 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리할 수 있었다.
* HttpMessageConverter 는 @ModelAttribute 와 다르게 각각의 필드 단위로 적용되는 것이 아니라, 전체 객체 단위로 적용된다. 따라서 메시지 컨버터의 작동이 성공해서 ItemSaveForm 객체를 만들어야 @Valid , @Validated 가 적용된다.
* @ModelAttribute 는 필드 단위로 정교하게 바인딩이 적용된다. 특정 필드가 바인딩 되지 않아도 나머지 필드는 정상 바인딩 되고, Validator를 사용한 검증도 적용할 수 있다.
* @RequestBody 는 HttpMessageConverter 단계에서 JSON 데이터를 객체로 변경하지 못하면 이후 단계 자체가 진행되지 않고 예외가 발생한다. 컨트롤러도 호출되지 않고, Validator도 적용할 수 없다.

