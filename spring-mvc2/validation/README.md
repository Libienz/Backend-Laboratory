# Spring MVC 2편

<details>
<summary>Section 04 Validation </summary>
<div markdown="1">

## 검증 요구사항 도착
- 상품 관리 시스템에 새로운 요구사항이 추가되었다.
  - 타입 검증
    - 가격, 수량에 문자가 들어가면 검증 오류 처리
  - 필드 검증
    - 상품명: 필수, 공백 X
    - 가격 1000원 이상, 1백만원 이하
    - 수량 최대 9999
  - 특정 필드의 범위를 넘어서는 검증
    - 가격 * 수량의 합은 10000원 이상
- 지금까지 만든 웹 애플리케이션은 폼 입력시 숫자를 문자로 작성하거나 해서 검증 오류가 발생하면 오류 화면으로 바로 이동한다.
- 이렇게 되면 사용자는 처음부터 해당 폼으로 이동해서 다시 입력을 해야 한다
- 유저 경험이 좋지 않다. 웹 서비스는 의례 폼 입력시 오류가 발생하면 고객이 입력한 데이터를 유지한 상태로 어떤 오류가 발생했는지 친절하게 알려주어야 한다.
- 컨트롤러의 중요한 역할 중 하나는 HTTP 요청이 정상인지 검증하는 것이다!
- 그리고 정상 로직보다 이런 검증 로직을 잘 개발하는 것이 더 어려울 수 있다.

### 참고: 클라이언트 검증, 서버 검증
- 클라이언트 검증은 조작할 수 있음으로 보안에 취약하다.
- 서버만으로 검증하면, 즉각적인 고객 사용성이 부족해진다.
- 둘을 적절히 섞어서 사용하되, 최종적으로 서버 검증은 필수
- API 방식을 사용하면 API 스펙을 잘 정의해서 검증 오류를 API 응답 결과에 잘 남겨주어야 함

## V1 검증 직접 처리
- 고객이 상품 등록 폼에서 상품명을 입력하지 않거나 가격, 수량 등이 너무 작거나 커서 검증 범위를 넘어서면 서버 검증 로직이 실패해야 한다.
- v1에서는 컨트롤러에서의 분기를 통해 직접적으로 검증을 처리한다. 

#### ValidationControllerV1 - addItem()

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
        //특정 필드가 아닌 복합 룰 검증
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                errors.put("globalError", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice);
            }
        }
        //검증에 실패하면 다시 입력 폼으로
        if (!errors.isEmpty()) {
            log.info("errors = {} ", errors);
            model.addAttribute("errors", errors);
            return "validation/v1/addForm"; //이렇게 넘어가도 유저가 입력한 값이 남아있다. 왜? th:object item에 들어있으니까
        }

        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v1/items/{itemId}";
    }
```
- 해당 코드를 보면 검증시 오류가 발생할 경우 Map에 오류 정보를 담아두는 것을 확인할 수 있다.
- 이때 어떤 필드에서 오류가 발생했는지 구분하기 위해 오류가 발생한 필드명을 key로 이용한다
- 이후 뷰에서 이 데이터를 사용해서 고객에게 친절한 오류메시지를 출력하는 것이다.

#### 검증에 실패하면 다시 입력 폼으로
```java
if (!errors.isEmpty()) {
 model.addAttribute("errors", errors);
 return "validation/v1/addForm";
}
```
- 만약 검증에서 오류 메시지가 하나라도 있으면 오류 메시지를 출력하기 위해 model에 errors를 담고 입력 폼이 있는 뷰 템플릿으로 보낸다.
- 이렇게 해도 유저가 입력한 데이터는 남아있다. 왜!? th:object Item에 담겨있으니까!

### 타임리프 오류 처리 
```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <link th:href="@{/css/bootstrap.min.css}"
          href="../css/bootstrap.min.css" rel="stylesheet">
    <style>
 .container {
 max-width: 560px;
 }
 .field-error {
 border-color: #dc3545;
 color: #dc3545;
 }
 </style>
</head>
<body>
<div class="container">
    <div class="py-5 text-center">
        <h2 th:text="#{page.addItem}">상품 등록</h2>
    </div>
    <form action="item.html" th:action th:object="${item}" method="post">
        <div th:if="${errors?.containsKey('globalError')}">
            <p class="field-error" th:text="${errors['globalError']}">전체 오류
                메시지</p>
        </div>
        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</
            label>
            <input type="text" id="itemName" th:field="*{itemName}"
                   th:class="${errors?.containsKey('itemName')} ? 'form-control 
field-error' : 'form-control'"
                   class="form-control" placeholder="이름을 입력하세요">
            <div class="field-error" th:if="${errors?.containsKey('itemName')}"
                 th:text="${errors['itemName']}">
                상품명 오류
            </div>
        </div>
        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}"
                   th:class="${errors?.containsKey('price')} ? 'form-control 
field-error' : 'form-control'"
                   class="form-control" placeholder="가격을 입력하세요">
            <div class="field-error" th:if="${errors?.containsKey('price')}"
                 th:text="${errors['price']}">
                가격 오류
            </div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}"
                   th:class="${errors?.containsKey('quantity')} ? 'form-control 
field-error' : 'form-control'"
                   class="form-control" placeholder="수량을 입력하세요">
            <div class="field-error" th:if="${errors?.containsKey('quantity')}"
                 th:text="${errors['quantity']}">
                수량 오류
            </div>
        </div>
        <hr class="my-4">
        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit"
                        th:text="#{button.save}">저장</button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/validation/v1/items}'|"
                        type="button" th:text="#{button.cancel}">취소</button>
            </div>
        </div>
    </form>
</div> <!-- /container -->
</body>
</html>
```
- th:if를 사용하여 조건에 만족할 경우에만 HTML 태그를 보여주는 것을 확인할 수 있다

### 정리
- 만약 검증 오류가 발생하면 입력 폼을 다시 보여준다.
- 검증 오류들을 고객에게 친절하게 안내해서 다시 입력할 수 있게 한다.
- 검증 오류가 발생해도 고객이 입력한 데이터가 유지된다.

### 문제점
- 뷰 템플릿에서 중복 처리가 많다 (뭔가 비슷한 코드의 향연)
- 타입 오류 처리가 안된다.
  - Item의 price, quantity 같은 숫자 필드는 타입이 Integer 임으로 문자 타입으로 설정하는 것이 불가능하다.
  - 숫자 타입에 문자가 들어오면 오류가 발생한다.
  - 그런데 이러한 오류는 스프링 MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에 컨트롤러가 호출되지도 않고 400을 띄우게 된다.
- 결국 고객이 입력한 값도 어딘가에 별도로 관리가 되어야 한다.

## V2 Binding Result 
- 지금부터는 스프링이 제공하는 검증 오류 처리 방법을 알아보자
- 핵심은 BindingResult

#### ValidationControllerV2 - addItemV1
```java

    @PostMapping("/add")
    public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {


        //검증 로직
        if (!StringUtils.hasText(item.getItemName())) {
//            errors.put("itemName", "상품 이름은 필수입니다.");
            bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수 입니다"));
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
//            errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
            bindingResult.addError(new FieldError("item", "price", "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
        }
        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
//            errors.put("quantity", "수량은 최대 9,999 까지 허용합니다.");
            bindingResult.addError(new FieldError("item", "quantity", "수량은 최대 9,999 까지 허용합니다."));
        }
        //특정 필드가 아닌 복합 룰 검증
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
//                errors.put("globalError", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice);
                bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice));
            }
        }
        //검증에 실패하면 다시 입력 폼으로
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }

        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
    }

```
- 이전 Map에서 하던 역할을 BindingResult가 하는 것이다.
- 필드에 오류가 있으면 FieldError 객체를 생성해서 bindingResult에 담아두면 된다.
- 이렇게 bindingResult를 채워서 뷰에 넘겨주면 타임리프는 또 최적화된 문법으로 이를 조회하고 조건 렌더링 할 수 있다.

```html
<form action="item.html" th:action th:object="${item}" method="post">
  <div th:if="${#fields.hasGlobalErrors()}">
    <p class="field-error" th:each="err : ${#fields.globalErrors()}"
       th:text="${err}">글로벌 오류 메시지</p>
  </div>
  <div>
    <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
    <input type="text" id="itemName" th:field="*{itemName}"
           th:errorclass="field-error" class="form-control"
           placeholder="이름을 입력하세요">
    <div class="field-error" th:errors="*{itemName}">
      상품명 오류
    </div>
  </div>
  <div>
    <label for="price" th:text="#{label.item.price}">가격</label>
    <input type="text" id="price" th:field="*{price}"
           th:errorclass="field-error" class="form-control"
           placeholder="가격을 입력하세요">
    <div class="field-error" th:errors="*{price}">
      가격 오류
    </div>
  </div>
  <div>
    <label for="quantity" th:text="#{label.item.quantity}">수량</label>
    <input type="text" id="quantity" th:field="*{quantity}"
           th:errorclass="field-error" class="form-control"
           placeholder="수량을 입력하세요">
    <div class="field-error" th:errors="*{quantity}">
      수량 오류
    </div>
  </div>
```
- 타임리프는 스프링의 BindingResult를 활용해서 편리하게 검증 오류를 표현하는 기능을 제공한다.

## BindingResult
- 스프링이 제공하는 검증 오류를 보관하는 객체이다.
- 검증 오류가 발생하면 여기에 보관하면 된다.
- BindingResult가 있으면 @ModelAttribute에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다! 
- 검증오류를 핸들링하겠다는 의사를 표명하는 것!

## FieldError, ObjectError
- 사용자 입력 오류 메시지가 화면에 남지 않는다.
- 화면에 남도록 수정해보자

```java    
    @PostMapping("/add")
    public String addItemV2(@ModelAttribute Item item, BindingResult bindingResult,
                            RedirectAttributes redirectAttributes) {
        if (!StringUtils.hasText(item.getItemName())) {
            bindingResult.addError(new FieldError("item", "itemName",
                    item.getItemName(), false, null, null, "상품 이름은 필수입니다."));
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >
                1000000) {
            bindingResult.addError(new FieldError("item", "price", item.getPrice(),
                    false, null, null, "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
        }
        if (item.getQuantity() == null || item.getQuantity() > 10000) {
            bindingResult.addError(new FieldError("item", "quantity",
                    item.getQuantity(), false, null, null, "수량은 최대 9,999 까지 허용합니다."));
        }
        //특정 필드 예외가 아닌 전체 예외
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.addError(new ObjectError("item", null, null, "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice));
            }
        }
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
    }

```
- FieldError의 생성자를 다르게 바꾸어 rejectedValue의 값을 남기도록 설정했다.
- 사용자의 입력 데이터가 컨트롤러의 @ModelAttribute에 바인딩되는 시점에 오류가 발생하면 모델 객체에 사용자 입력 값을 유지하기 어렵다
- 예를 들어서 가격에 숫자가 아닌 문자가 입력된다면 가격은 Integer 타입임으로 문자를 보관할 수 있는 방법이 없는 것이다.
- 그래서 오류가 발생한 경우 사용자 입력 값을 보관하는 별도의 방법이 필요하다.
- 그리고 이렇게 보관한 사용자 입력 값을 검증 오류 발생시 화면에 다시 출력하는 것이 위의 코드이다.

## 오류 코드와 메시지 처리1
- 오류 메시지를 하드코딩 하지 않고 체계적으로 다루어 보자

#### errors.properties

```properties
required.item.itemName=상품 이름은 필수입니다.
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
max.item.quantity=수량은 최대 {0} 까지 허용합니다.
totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}
```

#### addItemV3()
```java    
    @PostMapping("/add")
    public String addItemV3(@ModelAttribute Item item, BindingResult bindingResult,
                            RedirectAttributes redirectAttributes) {
        if (!StringUtils.hasText(item.getItemName())) {
            bindingResult.addError(new FieldError("item", "itemName",
                    item.getItemName(), false, new String[]{"required.item.itemName"}, null,
                    null));
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() >
                1000000) {
            bindingResult.addError(new FieldError("item", "price", item.getPrice(),
                    false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));
        }
        if (item.getQuantity() == null || item.getQuantity() > 10000) {
            bindingResult.addError(new FieldError("item", "quantity",
                    item.getQuantity(), false, new String[]{"max.item.quantity"}, new Object[]
                    {9999}, null));
        }
        //특정 필드 예외가 아닌 전체 예외
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.addError(new ObjectError("item", new String[]
                        {"totalPriceMin"}, new Object[]{10000, resultPrice}, null));
            }
        }
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
    }

```
- codes 인자를 통해 메시지 코드를 지정하고 argument를 전달해 치환할 값이 있으면 치환하도록 했다
- 중앙에서 메시지를 관리하고 코드를 이용하여 오류 메시지의 일관성을 유지하도록 설계한 것이다.

## 오류 코드와 메시지 처리2
- FieldError, ObjectError는 파라미터도 많아서 직접 다루기 너무 번거롭다.
- 좀 더 자동화 할 수 있는 여지가 없을까?
- 컨트롤러에서 BindingResult는 검증해야 할 객체인 target 바로 다음에 온다.
- 따라서 BindingResult는 이미 본인이 검증해야 할 객체인 target을 알고 있는 것이다.
- 미리 알고 있다는 점을 이용해서 코드를 줄여보자!
- BindingResult가 제공하는 rejectValue(), reject()를 사용하면 FieldError, ObjectError를 직접 생성하지 않고 깔끔하게 검증 오류를 다룰 수 있다.

#### ValidationItemControllerV2 - addItemV4() 추가
```java
    @PostMapping("/add")
    public String addItemV4(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
        log.info("objectName={}", bindingResult.getObjectName());
        log.info("target={}", bindingResult.getTarget());
        if (!StringUtils.hasText(item.getItemName())) {
            bindingResult.rejectValue("itemName", "required");
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
            bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);
        }
        if (item.getQuantity() == null || item.getQuantity() > 10000) {
            bindingResult.rejectValue("quantity", "max", new Object[]{9999}, null);
        }
        //특정 필드 예외가 아닌 전체 예외
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin", new Object[]{10000,
                        resultPrice}, null);
            }
        }
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
    }
```
- 실행해보니 오류 메시지가 정상 출력된다. 그런데 errors.properties에 있는 코드를 완벽히 직접 입력하지 않았는데 어떻게 출력한 것일까?
- rejectValue의 파라미터로 받는 error코드는 메시지에 등록된 코드가 아니다. (messageResolber를 위한 코드)
- 이는 뒤에서 자세히 알아보자

## 오류 코드와 메시지 처리 3
- 오류 코드를 만들 때 다음과 같이 자세히 만들 수도 있고, 
  - required.item.itemName : 상품 이름은 필수 입니다.
  - range.item.price : 상품의 가격 범위 오류 입니다.
- 또는 다음과 같이 단순하게 만들수도 있다. 
  - required : 필수 값 입니다.
  - range : 범위 오류 입니다.
- 단순하게 만들면 범용성이 좋아서 여러곳에서 사용할 수 있지만, 메시지를 세밀하게 작성하기 어렵다.
- 반대로 너무 자세하게 만들면 범용성이 떨어진다.
- 가장 좋은 방법은 범용성으로 사용하다가 세밀하게 작성해야 하는 경우에는 세밀한 내용이 적용되도록 메시지에 단계를 두는 방법이다.

### 예시
- 예를 들어서 required라는 오류 코드를 사용한다고 가정해보자
- 다음과 같이 required라는 메시지만 있으면 이 메시지를 선택해서 사용하는 것이다.
  - required : 필수 값 입니다.
- 그런데 오류 메시지에 required.item.itemName과 같이 객체명과 필드명을 조합한 세밀한 메시지 코드가 있으면 이 메시지를 높은 우선 순위로 사용하는 것이다.
  - required : 필수 값 입니다.
  - required.item.itemName: 상품 이름은 필수 입니다.
- 물론 이렇게 객체명과 필드명을 조합한 메시지가 있는지 우선 확인하고, 없으면 좀 더 범용적인 메시지를 선택하도록 추가 개발이 필요하다
- 하지만 범용성 있게 잘 개발해두면 메시지의 추가만으로 매우 편리하게 오류 메시지를 관리할 수 있을 것이다.
- 스프링은 MessageCodesResolver라는 것으로 이러한 기능을 지원한다! 


## 오류 코드와 메시지 처리 4
- 우선 테스트 코드로 MessageCodesResolver를 알아보자

```java
package hello.itemservice.validation;
import org.junit.jupiter.api.Test;
import org.springframework.validation.DefaultMessageCodesResolver;
import org.springframework.validation.MessageCodesResolver;
import static org.assertj.core.api.Assertions.assertThat;
public class MessageCodesResolverTest {
  MessageCodesResolver codesResolver = new DefaultMessageCodesResolver();
  @Test
  void messageCodesResolverObject() {
    String[] messageCodes = codesResolver.resolveMessageCodes("required",
            "item");
    assertThat(messageCodes).containsExactly("required.item", "required");
  }
  @Test
  void messageCodesResolverField() {
    String[] messageCodes = codesResolver.resolveMessageCodes("required",
            "item", "itemName", String.class);
    assertThat(messageCodes).containsExactly(
            "required.item.itemName",
            "required.itemName",
            "required.java.lang.String",
            "required"
    );
  }
}
```

### MessageCodesResolver
- 검증 오류 코드로 메시지 코드들을 생성한다.
- MessageCodesResolver는 인터페이스이고 DefaultMessageCodesResolver는 기본 구현체이다.
- DefaultMessageCodesResolver는 기본 메시지 생성 규칙이 있다.
  - 객체 오류
    - code + "." + object name
    - code
    - 예시
      - 오류 코드 : required, object name : item
      - required.item
      - required
  - 필드 오류
    - code + "." + object name + "." + field
    - code + "." + field
    - code + "." + field type
    - code
    - 예시 
      - 오류 코드: typeMismatch, object name "user", field "age", field type: int
      - "typeMismatch.user.age"
      - "typeMismatch.age"
      - "typeMismatch.int"
      - "typeMismatch"

### 동작 방식
- rejectValue(), reject()는 내부에서 MessageCodeResolver를 사용한다.
- 여기에서 메시지 코드들을 생성한다.
- FieldError, ObjectError의 생성자를 보면 오류 코드를 하나가 아니라 여러 오류 코드를 가질 수 있었다.
- MessageCodeResolver를 통해서 생성된 순서대로 오류 코드를 보관하기 위한 것이다.
- 이 부분을 로그를 찍어 확인해보면 코드가 다음과 같이 저장되어 있다
  - codes [range.item.price, range.price, range.java.lang.Integer, range]

#### FieldError rejectValue("itemName", "required")
- 다음 4가지 오류 코드를 자동으로 생성
- required.item.itemName
- required.itemName
- required.java.lang.String
- required
#### ObjectError reject("totalPriceMin")
- 다음 2가지 오류 코드를 자동으로 생성
- totalPriceMin.item
- totalPriceMin

#### 우선순위가 가능한 이유!
- 이렇게 codes를 만들어서 넘기면 우선순위에 맞게 걸리는 값을 꺼내는 것이다.
- 이를 통해서 범용적이면서도 세세한 것이 필요한 부분에는 알맞은 처리를 할 수 있는 메시지 코드 계층화가 이루어진 것이다


## 오류 코드와 메시지 처리 5
- 핵심은 구체적인 것에서 덜 구체적인 것으로!
- MessageCodeResolver는 required.item.itemName 처럼 구체적인 것을 먼저 만들어주고 required 처럼 덜 구체적인 것을 가장 나중에 만든다.
- 이렇게 하면 앞서 말한 것처럼 메시지와 관련된 공통 전략을 편리하게 도입할 수 있다.
- 하지만 왜 이렇게 복잡하게 사용하는가?
- 모든 오류 코드에 대해서 메시지를 각각 다 정의하면 개발자 입장에서 관리하기 너무 힘들다
- 계층화를 시켜 효과적으로 관리하는 것이다.
- 이제 우리 어플리케이션에 이런 오류 코드 전략을 도입해보자
- errors.properties만 수정하면 된다.

## 오류 코드와 메시지 처리
- 검증 오류 코드는 다음과 같이 2가지로 나눌 수 있다.
  - 개발자가 직접 설정한 오류 코드 -> rejectValue를 직접 호출
  - 스프링이 직접 검증 오류에 추가한 경우 (주로 타입 정보가 맞지 않음)
- 지금까지는 컨트롤러가 호출 되고 우리가 검증한 내용에 대해 오류 코드를 생성하고 처리하는 것을 보았다.
- 그런데 binding에 실패해서 스프링이 날려주는 오류 메시지는 어떻게 처리해야 할까?

### 직접 확인해보자
- price 필드에 문자 "A"를 입력해보자
- 로그를 확인해보면 BindingResult에 FieldError가 담겨있고, 다음과 같은 메시지 코드들이 생성되었다.
- codes[typeMismatch.item.price,typeMismatch.price,typeMismatch.java.lang.Integer,typ
  eMismatch]
- 4가지 메시지 코드가 입력되어 있는 것이다.
- 그렇다 스프링은 타입 오류가 발생하면 typeMismatch라는 오류 코드를 사용한다. 
- 이 오류 코드가 MessageCodesResolver를 통하면서 4가지 메시지 코드가 생성된 것이다.

### errors.properties 추가
```properties
#추가
typeMismatch.java.lang.Integer=숫자를 입력해주세요.
typeMismatch=타입 오류입니다.
```
- 이렇게 추가하고 실행하면 오류 메시지를 올바르게 처리하는 것을 확인할 수 있다.
- 메시지 코드 생성 전략은 그냥 만들어진 것이 아니다!
- 조금 뒤에서 Bean Validation을 학습하면 그 진가를 더 확인할 수 있다.

## Validator 분리1
- 목표 : 복잡한 검증 로직을 별도로 분리하자
- 컨트롤러에서 검증 로직이 차지하는 부분은 매우 크다.
- 이런 경우 별도의 클래스로 역할을 분리하는 것이 좋다.
- 그리고 이렇게 분리한 검증 로직을 재사용 할 수도 있다.

#### ItemValidator 생성
```java
package hello.itemservice.web.validation;
import hello.itemservice.domain.item.Item;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.ValidationUtils;
import org.springframework.validation.Validator;
@Component
public class ItemValidator implements Validator {
  @Override
  public boolean supports(Class<?> clazz) {
    return Item.class.isAssignableFrom(clazz);
  }
  @Override
  public void validate(Object target, Errors errors) {
    Item item = (Item) target;
    ValidationUtils.rejectIfEmptyOrWhitespace(errors, "itemName",
            "required");
    if (item.getPrice() == null || item.getPrice() < 1000 ||
            item.getPrice() > 1000000) {
      errors.rejectValue("price", "range", new Object[]{1000, 1000000},
              null);
    }
    if (item.getQuantity() == null || item.getQuantity() > 10000) {
      errors.rejectValue("quantity", "max", new Object[]{9999}, null);
    }
    //특정 필드 예외가 아닌 전체 예외
    if (item.getPrice() != null && item.getQuantity() != null) {
      int resultPrice = item.getPrice() * item.getQuantity();
      if (resultPrice < 10000) {
        errors.reject("totalPriceMin", new Object[]{10000,
                resultPrice}, null);
      }
    }
  }
}
```
- 스프링은 검증을 체계적으로 제공하기 위해 Validator 인터페이스를 제공한다.
- support() : 해당 검증기를 지원하는 여부 확인
- validate(): 검증 대상 객체와 BindingResult

#### ItemValidator 호출

```java
private final ItemValidator itemValidator;
@PostMapping("/add")
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult,
        RedirectAttributes redirectAttributes) {
        itemValidator.validate(item, bindingResult);

        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
}
```
- 실행해보면 기존과 완전히 동일하게 동작하는 것을 확인할 수 있다.
- 검증과 관련된 부분이 깔끔하게 분리 되었다. 

## Validator 분리 2
- 스프링이 Validator 인터페이스를 별도로 제공하는 이유는 체계적으로 검증 기능을 도입하기 위해서다.
- 그런데 앞에서는 검증기를 직접 불러서 사용했고, 이렇게 사용해도 문제는 없다.
- 그런데 Validator 인터페이스를 사용해서 검증기를 만들면 스프링의 추가적인 도움을 받을 수 있다.

### WebDataBinder
- WebDataBinder는 스프링의 파라미터 바인딩의 역할을 해주고 검증 기능도 내부에 포함한다.

#### ValidationItemControllerV2 추가
```java
@InitBinder
public void init(WebDataBinder dataBinder) {
        log.info("init binder {}", dataBinder);
        dataBinder.addValidators(itemValidator);
}
```
- 이렇게 WebDataBinder에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다.
- @InitBinder -> 해당 컨트롤러에만 영향을 준다. 글로벌 설정을 하려면 별도로!

#### addItemV6
```java
@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult
        bindingResult, RedirectAttributes redirectAttributes) {
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
}
```
- validator를 직접 호출하는 부분이 사라지고 대신에 검증 대상 앞에 @Validated가 붙었다.

### 동작 방식
- @Validated는 검증기를 실행하라는 애노테이션이다.
- 이 애노테이션이 붙으면 앞서 WebDataBinder에 등록한 검증기를 찾아서 실행한다.
- 그런데 여러 검증기를 등록한다면 그 중에 어떤 검증기가 실행되어야 할 지 구분이 필요하다.
- 이 때 supports()가 사용된다.
- 여기서는 supports(Item.class)가 호출되고, 결과가 true임으로 ItemValidator의 validate()가 호출된다!

</div>
</details>


<details>
<summary>Section 05 Bean Validation </summary>
<div markdown="1">

## Bean Validation - 소개
- 검증 기능을 지금처럼 매번 코드로 작성하는 것은 상당히 번거롭다.
- 특히 특정 필드에 대한 검증 로직은 대부분 빈 값인지 아닌지, 특정 크기를 넘는지 아닌지와 같이 매우 일반적인 로직이다.
- Bean Validation은 검증 로직을 편리하게 적용할 수 있도록 도와준다.

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

## Bean Validation - 시작
- Bean Validation 기능을 어떻게 사용하는지 코드로 알아보자
- 먼저 스프링과 통합하지 않고 순수한 Bean Validaiton 사용법 부터 테스트 코드로 알아보자
#### Item에 BeanValidation 애노테이션 적용
```java
package hello.itemservice.domain.item;
import lombok.Data;
import org.hibernate.validator.constraints.Range;
import javax.validation.constraints.Max;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
@Data
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
  public Item() {
  }
  public Item(String itemName, Integer price, Integer quantity) {
    this.itemName = itemName;
    this.price = price;
    this.quantity = quantity;
  }
}
```

#### 테스트 코드
```java
public class BeanValidationTest {
  @Test
  void beanValidation() {
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    Validator validator = factory.getValidator();
    Item item = new Item();
    item.setItemName(" "); //공백
    item.setPrice(0);
    item.setQuantity(10000);
    Set<ConstraintViolation<Item>> violations = validator.validate(item);
    for (ConstraintViolation<Item> violation : violations) {
      System.out.println("violation=" + violation);
      System.out.println("violation.message=" + violation.getMessage());
    }
  }
}
```
- 검증기를 명시적으로 생성하고 ConstraintViolation 출력 결과를 보면 다양한 정보를 확인할 수 있다.
- 스프링은 여기서 더 나아가서 이미 개발자를 위해 빈 검증기를 통합해두었는데 이제 그 사용법을 알아보자

## Bean Validation - 스프링 적용
- 기존에 등록한 ItemValidator를 제거해두고 BeanValidation을 사용해보자
- 정상 동작하는 것을 확인할  수 있다.
- 스프링 MVC는 어떻게 Bean Validator를 사용하는가?
- 스프링 부트에 spring-boot-starter-validation 라이브러리를 넣으면 자동으로 Bean Validator를 인지하고 스프링에 통합한다
- 스프링 부트는 자동으로 글로벌 Validator로 등록한다.
- 이 Validator는 @NotNull같은 애노테이션을 보고 검증을 수행한다.
- 이렇게 글로벌 Validator가 적용되어 있기 때문에 애노테이션만 적용하면 되는 것이다.
- 검증 오류가 발생하면 FieldError, ObjectError를 생성해서 BindinResult에 담아준다.

## Bean Validator 검증 순서
-  @ModelAttribute 각각의 필드에 타입 변환 시도
  - 성공하면 다음으로
  - 실패하면 typeMismatch 로 FieldError 추가
- Validator 적용
  - 바인딩에 성공한 필드만 Bean Validation이 적용된다.
  - 생각해보면 당연! 
  - 1000~10000사이여야 되는 건 바인딩이 올바르게 된 후에 검증할 수 있다.

## Bean Validation - 에러 코드 핸들링
- Bean Validation이 기본으로 제공하는 오류 메시지를 좀 더 자세히 변경하고 싶으면 어떻게 해야 할끼?
- 어떤 ErrorCode가 resolve되는지 확인하고 errors.properties에 추가하면 된다.
- 예시를 보자
- NotBlank라는 오류 코드를 기반으로 MessageCodeResolver가 만들어낸 메시지 코드들을 살펴보자
  - @NotBlank
  - NotBlank.item.itemName
  - NotBlank.itemName
  - NotBlank.java.lang.String
  - NotBlank
- 여기에 맞게 메시지를 등록하면 메시지가 잘 처리될 것이다.
- 또 메시지처리는 messageSource말고 다른 곳에서 할 수도 있다. 
- BeanValidation이 메시지를 찾는 순서를 알면 어떤 곳에 메시지를 등록하는지 알 수 있다
- BeanValidation이 메시지를 찾는 순서는 다음과 같다
  - 생성된 메시지 코드 순서대로 messageSource에서 메시지 찾기
  - 애노테이션의 message 속성 사용 (@NotBlank(message = "공백! {0}"))
  - 라이브러리가 제공하는 기본 값 사용
- 즉 메시지 코드를 통해 메시지를 찾는 것이 먼저 일어나고 없으면 애노테이션 순서로 감을 기억하자

## Bean Validation - 오브젝트 오류
- Bean Validation에서 특정 필드가 아닌 오브젝트 관련 오류는 어떻게 처리할 수 있을까?
- 잘 사용하지는 않지만 다음과 같이 @ScriptAssert()를 사용할 수 있다.

```java
@Data
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000")
public class Item {
 //...
}
```
- 실행해보면 오브젝트 오류를 잘 잡아내는 것을 확인할 수 있다.
- 메시지 코드도 다음과 같이 생성된다.
  - ScriptAssert.item
  - ScriptAssert
- 그런데 실제 사용해보면 제약이 많고 제공하는 기능보다 복잡함이 크다.
- 그리고 실무에서는 검증 기능이 해당 객체의 범위를 넘어서는 경우들도 종종 등장하는데 그런 경우 대응이 어렵다.
- 결론은 오브젝트 오류의 경우 @ScriptAssert를 사용하는 것보다 다음과 같이 오브젝트 오류 관련 부분만 직접 자바 코드로 작성하는 것을 권장한다.
```java    
    @PostMapping("/add")
    public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
        //특정 필드 예외가 아닌 전체 예외
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
            }
        }
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v3/addForm";
        }
        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }
```

## Bean Validation - 수정에 적용
- 상품 수정에도 빈 검증을 적용해보자
- 이전에 했던 것과 동일하게 수정을 진행했다.
- 그런데 이렇게 두면 한계가 드러나는데.. 바로 다음에 확인해보자

## Bean Validation - 한계
- 검증 요구사항이 추가되었다.
- 등록할 때의 검증사항과 수정할 때의 검증사항이 달라졌다.
- 등록할 때에는 수량을 9999까지 허용하지만 수정할 때에는 무한까지 가능하게 해달라는 요구사항이 들어온 것이다.
- 그런데 우리는 모두 Item에 애노테이션을 걸어서 필드 검증을 진행하고 있고 수정요청인지 등록 요청인지 알길이 없다.
- 바로 이러한 점이 Bean Validation의 한계이다. 
- 해결방법이 아예 없는 것은 아니다.
- 스프링 프레임워크의 @Validated에서만 제공되는 group기능을 사용할 수도 있고
- form 객체를 분리할 수도 있다.
- 하나씩 알아보자

## Bean Validation - groups
- 동일한 모델 객체를 등록할 때와 수정할 때 각각 다르게 검증하는 방법을 알아보자

#### Groups 적용
```java
package hello.itemservice.domain.item;
public interface SaveCheck {
}
```
```java
package hello.itemservice.domain.item;
public interface UpdateCheck {
}
```
#### Item - groups 적용

```java
package hello.itemservice.domain.item;
import lombok.Data;
import org.hibernate.validator.constraints.Range;
import javax.validation.constraints.Max;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
@Data
public class Item {
  @NotNull(groups = UpdateCheck.class) //수정시에만 적용
  private Long id;
  @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
  private String itemName;
  @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
  @Range(min = 1000, max = 1000000, groups = {SaveCheck.class,
          UpdateCheck.class})
  private Integer price;
  @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
  @Max(value = 9999, groups = SaveCheck.class) //등록시에만 적용
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
#### @Validated에 group 지정
```java
@PostMapping("/add")
public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes){
        //...
}
```

```java
@PostMapping("/{itemId}/edit")
public String editV2(@PathVariable Long itemId, @Validated(UpdateCheck.class)
@ModelAttribute Item item, BindingResult bindingResult) {
 //...
}
```
- groups 기능을 사용해서 등록과 수정시에 각각 다른 검증을 진행할 수 있게 되었다.
- 근데 groups 기능을 사용하니 Item은 물론이고, 전반적으로 복잡도가 올라갔다.
- 사실 groups 기능은 실제 잘 사용되지 않는데, 그 이유는 실무에서는 주로 다음에 등장하는 등록용 객체와 수정용 폼 객체를 분리해서 사용하기 때문이다.


## Form 전송 객체 분리 - 소개
- 실무에서는 groups를 잘 사용하지 않고 폼 객체를 분리하는 방법을 쓴다.
- HTML Form -> ItemSaveForm -> Controller -> Item 생성 -> Repository
- 생각해보면 수정과 등록에 요구되는 데이터의 set은 다르다. 
- 등록시에는 로그인 id, 주민번호 등등을 받을 수 있지만
- 수정 시에는 고칠 수 없는 값들도 있기 때문이다.
- 따라서 form 객체를 별도로 분리하는 것이 더 좋다.

## Form 전송 객체 분리 - 개발

#### 폼 객체 바인딩
```java
@PostMapping("/add")
public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form,
BindingResult bindingResult, RedirectAttributes redirectAttributes) {
 //...
}
```
- Item 대신에 ItemSaveForm을 전달 받는다.
- 그리고 @Validated로 검증도 수행하고 BindingResult로 검증 결과도 받는다.
- @ModelAttribute("item")으로 item 모델의 이름을 정의한 것을 주의하자
  - 이름을 지정하지 않는다면 모델의 이름은 규칙에 의해서 itemSaveForm이 될 것이다.
  - 현재 타임리프에서 모델을 item으로 접근하고 있기에 이런 부분을 생각해야 한다.

#### 폼 객체를 Item으로 변환
```java
//성공 로직
Item item = new Item();
item.setItemName(form.getItemName());
item.setPrice(form.getPrice());
item.setQuantity(form.getQuantity());
Item savedItem = itemRepository.save(item);
```
- 폼 객체의 데이터를 기반으로 Item 객체를 생성한다. 
- 이렇게 폼 객체 처럼 중간에 다른 객체가 추가되면 변환하는 과정이 추가된다.

## Bean Validation - HTTP 메시지 컨버터
- @Valid, @Validated는 HttpMessageConverter(@RequestBody)에도 적용할 수 있다.
- 다만 HttpMessageConverter는 바인딩에 실패하면 예외없이 컨트롤러를 호출할 수 없다.
- 이전에 @ModelAttribute는 바인딩에 실패해도 bindingResult에 값을 넘기고 컨트톨러를 호출 한 후 해결할 수 있었다.
- 이런 차이점이 있는 이유는 @ModelAttribute는 필드 단위로 세세한 주입이 이루어지는 반면 HttpMessageConverter단계에서는 객체로 변경하지 못하면 이후 단계 자체가 진행되지 않고 예외가 발생하기 때문이다.


</div>
</details>