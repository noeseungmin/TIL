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
