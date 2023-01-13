## Validation

* 클라이언트 검증 : 조작이 가능하며 보안에 취약하다.
* 서버 검증 : 즉각적인 고객 사용성이 부족해진다.
두가지 방식을 적절히 섞어서 활용하되 최종적으로 서버 검증이 필요하다. API 방식을 사용할시 aPi 스펙을 잘 정의 하고 검증 오류를 API 응답 결과에 남겨 주어야 한다.

### 상품 등록 검증 V1
```java
@PostMapping("/add")
public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {
 
 //검증 오류 결과를 보관
 Map<String, String> errors = new HashMap<>();
 
 //검증 로직
 if (!StringUtils.hasText(item.getItemName())) {
   errors.put("itemName", "상품 이름은 필수입니다.");
 }
 
 if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
   errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
 }
 
 if (item.getQuantity() == null || item.getQuantity() >= 9999) {
 errors.put("quantity", "수량은 최대 9,999 까지 허용합니다.");
 }
 // ....
 ```
검증시 오류가 발생하면 어떤 검증에서 오류가 발생했는지 정보를 담아준다. 검증시 오류가 발생하면errors에 담아 둔다. 이때 어떤 필드에서 오류가 발생했는지 구분하기 위해 오류가 발생한 필드명을 key로 사용한다. 이후 뷰에서 이 데이터를 활용해서 고객에게 오류 메시지를 출력한다.

```java
 //특정 필드가 아닌 복합 룰 검증
 if (item.getPrice() != null && item.getQuantity() != null) {
  int resultPrice = item.getPrice() * item.getQuantity();
 if (resultPrice < 10000) {
  errors.put("globalError", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " +    resultPrice);
 }
 }
 ```
 특정 필드를 넘어서는 오류 처리를 위해 globalError라는 key 사용
 
 ```java
 //검증에 실패하면 다시 입력 폼으로
 if (!errors.isEmpty()) {
   model.addAttribute("errors", errors);
  return "validation/v1/addForm";
 }
 ```
검증에서 오류 메시지가 하나라도 있을시 오류메시지 출력을 위해 model에 erorrs를 담고 입력폼이 있는 뷰템플릿으로 보낸다.

```
<form action="item.html" th:action th:object="${item}" method="post">
 <div th:if="${errors?.containsKey('globalError')}">
 <p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>
 </div>
 <div>
 <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
 <input type="text" id="itemName" th:field="*{itemName}" th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'" class="form-control" placeholder="이름을 입력하세요">
 <div class="field-error" th:if="${errors?.containsKey('itemName')}"th:text="${errors['itemName']}">상품명 오류</div>
 ```
`th:if`를 사용해서 조건에 만족할때만 글로벌 오류 메시지 출력 할 수 있게 할 수 있다.

등록폼에 진입한 시점에는 errors 가 없다. 따라서 errors.containsKey() 를 호출하는 순간 NullPointerException 이 발생한다.

`errors?.`는 errors 가 null 일때 NullPointerException 이 발생하는 대신, null 을 반환하는 문법이다. `th:i`f 에서 null 은 실패로 처리되므로 오류 메시지가 출력되지 않는다.


#### 문제점
* 뷰 템플릿에서 중복처리가 많고 타입 오류 처리가 안된다. Item 의 price , quantity 같은 숫자 필드는 타입이 Integer 이므로 문자 타입으로 설정하는 것이 불가능하다. 숫자 타입에 문자가 들어오면 오류가 발생한다. 그런데 이러한 오류는 스프링MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에, 컨트롤러가 호출되지도 않고, 400 예외가 발생하면서 오류 페이지를 띄워준다.
*  만약 컨트롤러가 호출된다고 가정해도 Item 의 price 는 Integer 이므로 문자를 보관할 수가 없다. 결국 문자는 바인딩이 불가능하므로 고객이 입력한 문자가 사라지게 되고, 고객은 본인이 어떤 내용을 입력해서 오류가 발생했는지 이해하기 어렵다.

### 상품 등록 검증 V2
#### BindingResult
스프링이 제공하는 검증오류 처리 방법의 핵심은 BindingResult이다.
* 스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관하면 된다.
* BindingResult 가 있으면 @ModelAttribute 에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다!

@ModelAttribute에 바인딩 타입시 오류발생 할 경우
* bidingResult가 없을시 -> 400오류가 발생 컨트롤러가 호출되지 않고, 오류 페이지로 이동한다.
* BindingResult가 있으면 -> 오류정보(FieldError)를 BidingResult에 담아서 컨트롤러를 정상 호출한다.

#### BindingResult에 검증 오류를 적용하는 3가지 방법
* @ModelAttribute 의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 FieldError 생성해서BindingResult 에 넣어준다.
* 개발자가 직접 넣어준다.
* Validator 사용

```java
@PostMapping("/add")
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult,
RedirectAttributes redirectAttributes) {
 if (!StringUtils.hasText(item.getItemName())) {
 bindingResult.addError(new FieldError("item", "itemName", "상품 이름은
필수입니다."));
 }
 if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >
1000000) {
 bindingResult.addError(new FieldError("item", "price", "가격은 1,000 ~ 
1,000,000 까지 허용합니다."));
 }
 if (item.getQuantity() == null || item.getQuantity() > 10000) {
 bindingResult.addError(new FieldError("item", "quantity", "수량은 최대
9,999 까지 허용합니다."));
 }
 //특정 필드 예외가 아닌 전체 예외
 if (item.getPrice() != null && item.getQuantity() != null) {
 int resultPrice = item.getPrice() * item.getQuantity();
 if (resultPrice < 10000) {
 bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은
10,000원 이상이어야 합니다. 현재 값 = " + resultPrice));
 }
 }
 if (bindingResult.hasErrors()) {
 log.info("errors={}", bindingResult);
 return "validation/v2/addForm";
 }
 ```
BindingResult 는 검증할 대상 바로 다음에 와야한다. 순서가 중요!!
* 예를 들어서 @ModelAttribute Item item , 바로 다음에 BindingResult 가 와야 한다. BindingResult 는 Model에 자동으로 포함된다.

`public ObjectError(String objectName, String defaultMessage) {}` 특정 필드를 넘어서는 오류가 있으면 ObjectError객체를 생성해서 bindingResult에 담아 두면 된다.
* ObjectName : @ModelAttribute의 이름
* defaultMessage : 오류의 기본 메시지

#### 타임리프 스프링 검증 오류 통합 기능 
타임리프는 스프링의 BindingResult 를 활용해서 편리하게 검증 오류를 표현하는 기능을 제공한다.
* #fields : #fields 로 BindingResult 가 제공하는 검증 오류에 접근할 수 있다.
* th:errors : 해당 필드에 오류가 있는 경우에 태그를 출력한다. th:if 의 편의 버전이다.
* th:errorclass : th:field 에서 지정한 필드에 오류가 있으면 class 정보를 추가한다.

#### BidingResult와 Erorrs 
BidingResult는 Errors 인테페으스를 상속 받고 있다. 실제 넘어오는 구현체는 BeanPropertyBindingResult 라는 것인데, 둘다 구현하고 있으므로 둘다 사용 가능하다 
* Errors는 단순한 오류 저장과 조회 기능을 제공한다. 
* BindingResult 추가적인 기능(`addError()`)을 제공한다. 

#### FieldError, ObjectError
FieldError는 두 가지 생성자를 제공한다.
```java
public FieldError(String objectName, String field, String defaultMessage);
public FieldError(String objectName, String field, @Nullable Object 
rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable
Object[] arguments, @Nullable String defaultMessage)
```
* `objectName` : 오류가 발생한 객체 이름
* `field` : 오류 필드
* `rejectedValue` : 사용자가 입력한 값(거절된 값)
* `bindingFailure` : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
* `codes` : 메시지 코드
* `arguments` : 메시지에서 사용하는 인자
* `defaultMessage` : 기본 오류 메시지

FieldError , ObjectError 의 생성자는 `codes` , `arguments` 를 제공한다. 이것은 오류 발생시 오류 코드로 메시지를 찾기 위해 사용된다.


사용자의 입력 데이터가 컨트롤러의 @ModelAttribute 에 바인딩되는 시점에 오류가 발생하면 모델 객체에 사용자 입력 값을 유지하기 어렵다. 예를 들어서 가격에 숫자가 아닌 문자가 입력된다면 가격은 Integer 타입이므로 문자를 보관할 수 있는 방법이 없다. 그래서 오류가 발생한 경우 사용자 입력 값을 보관하는 별도의 방법이 필요하다. 그리고 이렇게 보관한 사용자 입력 값을 검증 오류 발생시 화면에 다시 출력하면 된다. FieldError 는 오류 발생시 사용자 입력 값을 저장하는 기능을 제공한다.

#### 타임리프의 사용자 입력 값 유지
`th:field="*{price}"`
타임리프의 `th:field` 는 매우 똑똑하게 동작하는데, 정상 상황에는 모델 객체의 값을 사용하지만, 오류가 발생하면 FieldError 에서 보관한 값을 사용해서 값을 출력한다.

#### 스프링의 바인딩 오류 처리
타입 오류로 바인딩에 실패하면 스프링은 FieldError 를 생성하면서 사용자가 입력한 값을 넣어둔다. 
그리고 해당 오류를 BindingResult 에 담아서 컨트롤러를 호출한다. 따라서 타입 오류 같은 바인딩
실패시에도 사용자의 오류 메시지를 정상 출력할 수 있다.

`new FieldError("item", "price", item.getPrice(), false, new String[] {"range.item.price"}, new Object[]{1000, 1000000}`
* `codes` : `required.item.itemName`를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라 배열로 여러 값을 전달할 수 있는데, 순서대로 매칭을 시도하는데 처음 매칭되는 메시지가 적용된다.
* `arguments` : `Object[]{1000,1000000}`를 사용해서 코드의 {0},{1}로 치환할 값을 전달한다.

FieldError , ObjectError 는 다루기 너무 번거롭다. 

#### rejectValue() , reject()
```java
void rejectValue(@Nullable String field, String errorCode,
@Nullable Object[] errorArgs, @Nullable String defaultMessage);
```

`field` : 오류 필드명
`errorCode` : 오류 코드(이 오류 코드는 메시지에 등록된 코드가 아니다. 뒤에서 설명할messageResolver를 위한 오류 코드이다.)
`errorArgs` : 오류 메시지에서 {0} 을 치환하기 위한 값
`defaultMessage` : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

`bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);`
`bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);`
BindingResult 가 제공하는 rejectValue() , reject() 를 사용하면 FieldError , ObjectError 를 직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다.

* BindingResult 는 어떤 객체를 대상으로 검증하는지 target을 이미 알고 있다고 했다. 따라서 target(item)에 대한 정보는 없어도 된다. 오류 필드명은 동일하게 price 를 사용했다.
* FieldError() 를 직접 다룰 때는 오류 코드를 range.item.price 와 같이 모두 입력했다. 그런데 rejectValue() 를 사용하고 부터는 오류 코드를 range 로 간단하게 입력했다. 그래도 오류 메시지를 잘 찾아서 출력한다.

#### 오류 코드와 메시지 처리
스프링은 MessageCodesResolver 라는 것으로 이러한 기능을 지원한다

자세히 
* `requireed.item.itemName` : 상품이름은 필수입니다. 
* `reange.item.price` : 상품의 가격 범위 오류입니다.
단순하게
* `required` : 필수 값
* `reange` : 범위오류 입니다.
단순하게 만들면 범용성이 좋아서 여러곳에서 사용할 수 있지만, 메시지를 세밀하게 작성하기 어렵다. 
반대로 너무 자세하게 만들면 범용성이 떨어진다. 가장 좋은 방법은 범용성으로 사용하다가, 세밀하게
작성해야 하는 경우에는 세밀한 내용이 적용되도록 메시지에 단계를 두는 방법이다.

객체명과 필드명을 조합한 메시지(`required.item.itemName`)가 있는지 우선 확인하고, 없으면 좀 더 범용적인 메시지(`required`)를 선택하도록 추가 개발을 해야겠지만, 범용성 있게 잘 개발해두면, 메시지의 추가 만으로 매우 편리하게 오류 메시지를 관리할 수 있을 것이다.

#### 오류 코드 관리
핵심은 구체적인 것에서! 덜 구체적인 것으로!
`MessageCodesResolver` 는 `required.item.itemName` 처럼 구체적인 것을 먼저 만들어주고, required 처럼 덜 구체적인 것을 가장 나중에 만든다.

모든 오류 코드에 대해서 메시지를 각각 다 정의하면 개발자 입장에서 관리가 너무 힘들기 때문에 크게 중요하지 않은 메시지는 범용성이 있는 required 같은 메시지로 끝내고 중요한 메시지는 꼭 필요할 때 구체적으로 적어서 사용하는 방식이 효과적이다. 

#### Validator 분리
컨트롤러에서 검증 로직이 차지하는 부분은 매우 크다. 이런 경우 별도의 클래스로 역할을 분리하는 것이 좋다. 그리고 이렇게 분리한 검증 로직을 재사용 할 수도 있다.
```java
public interface Validator {
boolean supports(Class<?> clazz);
void validate(Object target, Errors errors);
}
```
스프링은 검증을 체계적으로 제공하기 위해 해당 인터페이스를 제공한다.
* `supports() {}` : 해당 검증기를 지원하는 여부 확인
* `validate(Object target, Errors errors)` : 검증 대상 객체와 BindingResult
```java
@InitBinder
public void init(WebDataBinder dataBinder) {
 log.info("init binder {}", dataBinder);
 dataBinder.addValidators(itemValidator);
}
```
이렇게 `WebDataBinder` 에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다. `@InitBinder` 해당 컨트롤러에만 영향을 준다.

```java
@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult 
bindingResult, RedirectAttributes redirectAttributes) {
 if (bindingResult.hasErrors()) {
 log.info("errors={}", bindingResult);
 return "validation/v2/addForm";
 }
 //....
 ```
@Validated 애노테이션이 붙으면 앞서 WebDataBinder 에 등록한 검증기를 찾아서 실행한다. 그런데 여러 검증기를 등록한다면 그 중에 어떤 검증기가 실행되어야 할지 구분이 필요하다. 이때 supports() 가 사용된다. 여기서는 supports(Item.class) 호출되고, 결과가 true 이므로 ItemValidator 의 validate() 가호출된다.
