# SpringValidation


# 1. 검증 요구사항

상품 관리 시스템에 새로운 요구사항이 추가되었다.

### 요구사항: 검증 로직 추가

- 타입 검증
    - 가격, 수량에 문자가 들어가면 검증 오류 처리
- 필드 검증
    - 상품명: 필수, 공백X
    - 가격: 1000원 이상, 1백만원 이하
    - 수량: 최대 9999
- 특정 필드의 범위를 넘어서는 검증
    - 가격 * 수량의 합은 10,000원 이상

지금까지 만든 웹 애플리케이션은 폼 입력시 숫자를 문자로 작성하는 등의 검증 오류가 발생하면 오류 화면으로 바로 이동합니다. 이렇게 되면 사용자는 처음부터 해당 폼으로 다시 이동해서 입력을 해야 한다.

아마도 이런 서비스라면 사용자는 금방 떠나버릴 것이다. 웹 서비스는 폼 입력시 오류가 발생하면, 고객이 입력한 데이터를 유지한 상태로 어떤 오류가 발생했는지 친절하게 알려주어야 한다.

컨트롤러의 중요한 역할중 하나는 HTTP 요청이 정상인지 검증하는 것이다. 그리고 정상 로직보다 이런 검증 로직을 잘 개발하는 것이 어쩌면 더 어려울 수 있다.

### 참고: 클라이언트 검증, 서버 검증

- 클라이언트 검증은 조작할 수 있으므로 보안에 취약하다.
- 서버만으로 검증하면, 즉각적인 고객 사용성이 부족해진다.
- 둘을 적절히 섞어서 사용하되, 최종적으로 서버 검증은 필수이다!!!
- API 방식을 사용하면 API 스펙을 잘 정의해서 검증 오류를 API 응답 결과에 잘 남겨주어야 합니다.

먼저 검증을 직접 구현해보고, 뒤에서 스프링과 타임리프가 제공하는 검증 기능을 활용해봅시다.

# 2. 프로젝트 설정 V1

이전 프로젝트에 이어서 검증(Validation) 기능을 학습해보자.

이번에도 역시 자바 스프링 MVC 1편에서의 프로젝트을 사용합니다. 패키지 이름과 url 이름만 바꾸었습니다.

# 3. 검증 직접 처리 - 소개

### 상품 저장 성공

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4359ba5d-0ce4-40f4-a609-f7823f9e144b/Untitled.png)

- 사용자가 상품 등록 폼에서 정상적인 데이터를 입력하면
    - 서버에서는 검증 로직이 통과하고
    - 상품을 저장하고
    - 상품 상세 화면으로 redirect 합니다.

### 상품 저장 검증 실패

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/edcfcc30-3d24-4d95-bb42-7483bc175289/Untitled.png)

- 고객이 상품 등록 폼에서 상품명을 입력하지 않거나, 가격, 수량 등이 너무 작거나 커서 검증 범위를 넘어서면,
    - 서버 검증 로직이 실패해야 한다.
    - 이렇게 검증에 실패한 경우 고객에게 다시 상품 등록 폼을 보여주고,
    - 어떤 값을 잘못 입력했는지 친절하게 알려주어야 한다.

너무나 당연한 말이죠???!!! 이제 요구사항에 맞추어 검증 로직을 직접 개발해봅시다.

# 4. 검증 직접 처리 - 개발

### 상품 등록 검증

먼저 상품 등록 검증 코드를 작성해보자.

`ValidationItemControllerV1` - `addItem()` 수정

```java
@PostMapping("/add")
public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes , Model model) {

    // 검증 오류 결과 보관
    Map<String, String> errors = new HashMap<>();

    // 검증 로직
    if (!StringUtils.hasText(item.getItemName())) {
        errors.put("itemName", "상품 이름은 필수입니다.");
    }
    if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
    }
    if (item.getQuantity() == null || item.getQuantity() >= 9999) {
        errors.put("quantity", "수량은 최대 9,999 까지 허용합니다.");
    }

    // 특정 필드가 아닌 복합 룰 검증
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            errors.put("globalError", "가격 * 수량의 합은 10,000 원 이상이어야 합니다. 현재 값 = " + resultPrice);
        }
    }

    // 검증에 실패하면 다시 입력 폼으로
    if (!errors.isEmpty()) {
				log.info("errors={}", errors);
        model.addAttribute("errors", errors);
        return "validation/v1/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v1/items/{itemId}";
}
```

위 코드를 보면 그렇게 어려운 부분은 없다. 일일이 모든 코드를 보지 않고 조금씩만 봅시다.

검증 오류 보관

```java
Map<String, String> errors = new HashMap<>();
```

만약 검증시 오류가 발생하면 어떤 검증에서 오류가 발생했는지 정보를 담아둔다.

`import org.springframework.util.StringUtils;` 을 추가해야 합니다.

검증시 오류가 발생하면 errors 에 담아둔다. 

이 때 어떤 필드에서 오류가 발생했는지 구분하기 위해 오류가 발생한 필드명을 key 로 사용한다. 

이 후 뷰에서 이 데이터를 사용해서 고객에게 친절한 오류 메시지를 출력할 수 있다

검증에 실패하면 다시 입력 폼으로 이동하도록 한다.

```java
if (!errors.isEmpty()) {
		log.info("errors={}", errors);
    model.addAttribute("errors", errors);
    return "validation/v1/addForm";
}
```

만약 검증에서 오류 메시지가 하나라도 있으면 오류 메시지를 출력하기 위해 `model` 에 `errors` 를 담고, 입력 폼이 있는 뷰 템플릿으로 보낸다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1c123d89-8e65-442d-9fc0-bc1b75a06909/Untitled.png)

위와 같이 이름을 입력하지 않고 다른 조건은 만족시켰을 때 저장을 해보면 아래 오류가 로그로 찍힙니다.

```java
h.i.w.v.ValidationItemControllerV1 : errors={itemName=상품 이름은 필수입니다.}
```

만약 모든 조건을 만족시키지 않았을 때는 아래처럼 오류 로그가 찍힐 것입니다.

```java
h.i.w.v.ValidationItemControllerV1 : errors={itemName=상품 이름은 필수입니다.
, quantity=수량은 최대 9,999 까지 허용합니다.
, price=가격은 1,000 ~ 1,000,000 까지 허용합니다.
, globalError=가격 * 수량의 합은 10,000 원 이상이어야 합니다. 현재 값 = 0}
```

걸국 우리는 이 오류 표시를 친절하게 사용자에게 표시해야 합니다!

그렇다면 아래 뷰 템플릿을 봅시다.

`addForm.html`

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
            <p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>
        </div>
        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}"
                   th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"
                   class="form-control" placeholder="이름을 입력하세요">
            <div class="field-error" th:if="${errors?.containsKey('itemName')}"
                 th:text="${errors['itemName']}">
                상품명 오류
            </div>
        </div>
        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}"
                   th:class="${errors?.containsKey('price')} ? 'form-control field-error' : 'form-control'"
                   class="form-control" placeholder="가격을 입력하세요">
            <div class="field-error" th:if="${errors?.containsKey('price')}"
                 th:text="${errors['price']}">
                가격 오류
            </div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}"
                   th:class="${errors?.containsKey('quantity')} ? 'form-control field-error' : 'form-control'"
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
                        th:text="#{button.save}">저장
                </button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/validation/v1/items}'|"
                        type="button" th:text="#{button.cancel}">취소
                </button>
            </div>
        </div>
    </form>
</div> <!-- /container -->
</body>
</html>
```

먼저 오류 메시지를 빨간색으로 출력하여 사용자의 인식을 높입니다.

css 추가 - 빨간색

```html
.field-error {
    border-color: #dc3545;
    color: #dc3545;
}
```

글로벌 오류 메시지

```html
<div th:if="${errors?.containsKey('globalError')}">
	<p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>
</div>
```

오류 메시지는 `errors` 에 내용이 있을 때만 출력하면 된다. 

타임리프의 `th:if` 를 사용하면 조건에 만족할 때만 해당 HTML 태그를 출력할 수 있다.

참고 Safe Navigation Operator

만약 여기에서 `errors` 가 `null` 이라면 어떻게 될까요?

생각해보면 등록폼에 진입한 시점에는 `errors` 가 없다. 따라서 `errors.containsKey()` 를 호출하는 순간 `NullPointerException` 이 발생합니다.

`errors?.` 은 `errors` 가 `null` 일때 `NullPointerException` 이 발생하는 대신에, `null` 을 반환하는 문법입니다. 안드로이드 코틀린에서도 비슷한 문법이 있죠!

즉, `th:if` 에서 `null` 은 실패로 처리되므로 오류 메시지가 출력되지 않습니다.

이것은 스프링의 SpringEL이 제공하는 문법입니다. 자세한 내용은 다음을 참고하면 됩니다.

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions-operator-safe-navigation

대표적으로 아래 코드를 봅시다.

```html
<div>
    <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
    <input type="text" id="itemName" th:field="*{itemName}"
           th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"
           class="form-control" placeholder="이름을 입력하세요">
    <div class="field-error" th:if="${errors?.containsKey('itemName')}"
         th:text="${errors['itemName']}">
        상품명 오류
    </div>
</div>
```

`key` 가 `itemName` 인 `model` 의 데이터에서 에러가 있다면 클래스(상자 모습)을 `form-control field-error` 로 하고 에러를 출력합니다. 만약 에러가 없다면 클래스를 `form-control` 로 하고 오류를 출력하지 않습니다.

위 코드에서 `classappend` 을 사용하여 조금 더 깔끔하게 코드를 사용할 수도 있다.

```html
<input type="text" 
		th:classappend="${errors?.containsKey('itemName')} ? 'field-error' : _"
		class="form-control">
```

그러면  상품 등록을 실행하고 검증이 잘 동작 하는지 확인해봅시다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d6818db0-203f-461e-bddf-efa3417dc743/Untitled.png)

정상적으로 동작하는 것을 확인할 수 있습니다!

### 정리

- 만약 검증 오류가 발생하면 입력 폼을 다시 보여준다.
- 검증 오류들을 고객에게 친절하게 안내해서 다시 입력할 수 있게 한다.
- 검증 오류가 발생해도 고객이 입력한 데이터가 유지된다.

### 남은 문제점

- 뷰 템플릿에서 중복 처리가 많습니다. 뭔가 비슷한 코드가 많지요
- 타입 오류 처리가 안된다.
    - `Item` 의 `price` , `quantity` 같은 숫자 필드는 타입이 `Integer` 이므로 문자 타입으로 설정하는 것이 불가능합니다.
    - 즉, 숫자 타입에 문자가 들어오면 오류가 발생하는데 이 오류는 스프링MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에, 컨트롤러가 호출되지도 않고, 400 예외가 발생하면서 오류 페이지를 띄워줍니다.
    - `Item` 의 `price` 에 문자를 입력하는 것 처럼 타입 오류가 발생해도 고객이 입력한 문자를 화면에 남겨야 할 수도 있습니다. 만약 컨트롤러가 호출된다고 가정해도 `Item` 의 `price` 는 `Integer` 이므로 문자를 보관할 수가 없습니다.
    - 결국 문자는 바인딩이 불가능하므로 고객이 입력한 문자가 사라지게 되고, 고객은 본인이 어떤 내용을
    입력해서 오류가 발생했는지 이해하기 어려워지지요….
    - 결국에는! 고객이 입력한 값도 어딘가에 별도로 관리가 되어야 합니다!
    

다음부터 스프링이 제공하는 검증 방법을 하나씩 알아봅니다.

# 5. 프로젝트 준비 V2

앞서 만든 기능을 유지하기 위해, 컨트롤러와 템플릿 파일을 복사합니다.

`ValidationItemControllerV2` 컨트롤러 생성

`hello.itemservice.web.validation.ValidationItemControllerV1` 복사
`hello.itemservice.web.validation.ValidationItemControllerV2` 붙여넣기

URL 경로 변경: `validation/v1/` →  `validation/v2/`

### 템플릿 파일 복사

- `validation/v1` 디렉토리의 모든 템플릿 파일을 `validation/v2` 디렉토리로 복사
    - `/resources/templates/validation/v1/` →  `/resources/templates/validation/v2/`
        - `addForm.html`
        - `editForm.html`
        - `item.html`
        - `items.html`

- `/resources/templates/validation/v2/` 하위 4개 파일 모두 URL 경로 변경
    - `validation/v1/` → `validation/v2/`
        - `addForm.html`
        - `editForm.html`
        - `item.html`
        - `items.html`

참고: 이 때 cmd + R 과 cmd + shift + R 을 사용하여 더 쉽게 바꿀 수 있습니다!! (윈도우는 ctrl + R & ctrl + shift + R)


# 6. BindingResult1

지금부터 스프링이 제공하는 검증 오류 처리 방법을 알아보자. 여기서 핵심은 BindingResult이다. 우선 코드로 확인해보자.

`ValidationItemControllerV2` - `addItemV1`

```java
@PostMapping("/add")
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes , Model model) {

    if (!StringUtils.hasText(item.getItemName())) {
        bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수 입니다."));
    }
    if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.addError(new FieldError("item", "price", "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
    }
    if (item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.addError(new FieldError("item", "quantity", "수량은 최대 9,999 까지 허용합니다."));
    }

    // 특정 필드가 아닌 복합 룰 검증
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.addError(new ObjectError("item", "가격 + 수량의 합은 10,000 원 이상이어야 합니다. 현재 값= " + resultPrice));
        }
    }

    // 검증에 실패하면 다시 입력 폼으로
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

코드 변경

메서드 이름 변경: `addItem()` → `addItemV1()`

주의

`BindingResult bindingResult` 파라미터의 위치는 `@ModelAttribute Item item` 다음에 와야 한다!!!

`FieldError` - 필드 오류

```java
if (!StringUtils.hasText(item.getItemName())) {
    bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수 입니다."));
}
```

필드에 오류가 있으면 `FieldError` 객체를 생성해서 `bindingResult` 에 담아두면 된다.

`FieldError` 생성자 요약

```java
public FieldError(String objectName, String field, String defaultMessage) {}
```

- `objectName` : `@ModelAttribute` 이름
- `field` : 오류가 발생한 필드 이름
- `defaultMessage` : 오류 기본 메시지

`ObjectError` - 글로벌 오류

```java
bindingResult.addError(new ObjectError("item", "가격 + 수량의 합은 10,000 원 이상이어야 합니다. 현재 값= " + resultPrice));
```

특정 필드를 넘어서는 오류가 있으면 `ObjectError` 객체를 생성해서 `bindingResult` 에 담아두면 된다.

`ObjectError` 생성자 요약

```java
public ObjectError(String objectName, String defaultMessage) {}
```

- `objectName` : `@ModelAttribute` 의 이름
- `defaultMessage` : 오류 기본 메시지

`validation/v2/addForm.html` 수정

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

        <div th:if="${#fields.hasGlobalErrors()}">
            <p class="field-error" th:each="err: ${#fields.globalErrors()}" th:text="${err}">글로벌 오류 메시지</p>
        </div>

        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}"
                   th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
            <div class="field-error" th:errors="*{itemName}">상품명 오류</div>
        </div>

        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}"
                   th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
            <div class="field-error" th:errors="*{price}">가격 오류</div>
        </div>
        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}"
                   th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
            <div class="field-error" th:errors="*{quantity}">수량 오류</div>
        </div>
        <hr class="my-4">
        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit"
                        th:text="#{button.save}">저장
                </button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='items.html'"
                        th:onclick="|location.href='@{/validation/v2/items}'|"
                        type="button" th:text="#{button.cancel}">취소
                </button>
            </div>
        </div>
    </form>
</div> <!-- /container -->
</body>
</html>
```

이전 버전에 비해서 타임리프도 굉장히 간단해 진 부분들이 보입니다!!!

### 타임리프 스프링 검증 오류 통합 기능

- 타임리프는 스프링의 `BindingResult` 를 활용해서 편리하게 검증 오류를 표현하는 기능을 제공한다.
- `#fields` : `#fields` 로 `BindingResult` 가 제공하는 검증 오류에 접근할 수 있다.
- `th:errors` : 해당 필드에 오류가 있는 경우에 태그를 출력한다. `th:if` 의 편의 버전이다.
- `th:errorclass` : `th:field` 에서 지정한 필드에 오류가 있으면 class 정보를 추가한다.

- 검증과 오류 메시지 공식 메뉴얼
    - https://www.thymeleaf.org/doc/tutorials/3.0/thymeleafspring.html#validation-and-error-messages

글로벌 오류 처리

```html
<div th:if="${#fields.hasGlobalErrors()}">
    <p class="field-error" th:each="err: ${#fields.globalErrors()}" th:text="${err}">글로벌 오류 메시지</p>
</div>
```

필드 오류 처리

```html
<input type="text" id="itemName" th:field="*{itemName}"
       th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
<div class="field-error" th:errors="*{itemName}">상품명 오류</div>
```

이전 검증 V1 의 `addForm` 에 비해 꽤 간단하고 편리해진 것을 확인할 수 있습니다!!

# 7. BindingResult2

스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관하면 된다.

`BindingResult` 가 있으면 `@ModelAttribute` 에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다!

예) `@ModelAttribute` 에 바인딩 시 타입 오류가 발생하면?

`BindingResult` 가 없으면 400 오류가 발생하면서 컨트롤러가 호출되지 않고, 오류 페이지로 이동한다.

`BindingResult` 가 있으면 오류 정보( `FieldError` )를 `BindingResult` 에 담아서 컨트롤러를 정상 호출한다

### BindingResult 에 검증 오류를 적용하는 3가지 방법

- `@ModelAttribute` 의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 `FieldError` 생성해서 `BindingResult` 에 넣어준다.
- 개발자가 직접 넣어준다.
- `Validator` 사용 이것은 뒤에서 설명

### 타입 오류 확인

숫자가 입력되어야 할 곳에 문자를 입력해서 타입을 다르게 해서 `BindingResult` 를 호출하고 `bindingResult` 의 값을 확인해보자.

아래 그림치럼 숫자가 입력되어야 할 가격에 qqq 을 입력했다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cfaca4a0-b882-40dd-836b-609eee58103b/Untitled.png)

아래와 같은 오류를 로그로 확인할 수 있습니다!

```java
Field error in object 'item' on field 'price': rejected value [qqq]; codes [typeMismatch.item.price,typeMismatch……중략……[Failed to convert property value of type 'java.lang.String' to required type 'java.lang.Integer' … 생략
```

주의

- `BindingResult` 는 검증할 대상 바로 다음에 와야한다. 순서가 중요하다.
    - 예를 들어서 `@ModelAttribute Item item` , 바로 다음에 `BindingResult` 가 와야 한다.
    - `BindingResult` 는 `Model` 에 자동으로 포함된다

### BindingResult와 Errors

`org.springframework.validation.Errors`

`org.springframework.validation.BindingResult`

`BindingResult` 는 인터페이스이고, `Errors` 인터페이스를 상속받고 있다.

실제 넘어오는 구현체는 `BeanPropertyBindingResult` 라는 것인데, 둘다 구현하고 있으므로 `BindingResult` 대신에 `Errors` 를 사용해도 된다. 

`Errors` 인터페이스는 단순한 오류 저장과 조회 기능을 제공한다. `BindingResult` 는 여기에 더해서 추가적인 기능들을 제공한다. 

`addError()` 도 `BindingResult` 가 제공하므로 여기서는 `BindingResult` 를 사용하자. 주로 관례상 `BindingResult` 를 많이 사용한다.

### 정리

- `BindingResult` , `FieldError` , `ObjectError` 를 사용해서 오류 메시지를 처리하는 방법을 알아보았습니다.
- 그런데 오류가 발생하는 경우 고객이 입력한 내용이 모두 사라집니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/98ba1d0e-64c5-438d-bfbc-a9d0e00ccb2e/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b155d35f-65bf-4a2c-b751-7d4e97310f53/Untitled.png)

위 그림처럼 price 에 100 을 입력하고 저장을 하였는데 오류가 발생하면서 기존에 입력한 price 값이 지워지고 있습니다. 

이것이 지워지지 않도록 하기 위해서 어떻게 해야 할까요?  이것을 해결해 봅시다.


# 8. FieldError, ObjectError

우리는 사용자 입력 오류 메시지가 화면에 남도록 할 것입니다.

예) 가격을 1000원 미만으로 설정시 입력한 값이 남아있어야 한다.

그리고 `FieldError` , `ObjectError` 에 대해서 더 자세히 알아볼 것입니다.

`ValidationItemControllerV2` - `addItemV2`

```java
@PostMapping("/add")
public String addItemV2(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    if (!StringUtils.hasText(item.getItemName())) {
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, null, null, "상품 이름은 필수 입니다."));
    }
    if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, null, null, "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
    }
    if (item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, null, null, "수량은 최대 9,999 까지 허용합니다."));
    }

    // 특정 필드가 아닌 복합 룰 검증
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.addError(new ObjectError("item", null, null, "가격 + 수량의 합은 10,000 원 이상이어야 합니다. 현재 값= " + resultPrice));
        }
    }

    // 검증에 실패하면 다시 입력 폼으로
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

코드 변경

`addItemV1()` 의 `@PostMapping("/add")` 을 주석 처리하자.

### FieldError 생성자

`FieldError` 는 두 가지 생성자를 제공한다.

```java
public FieldError(String objectName, String field, String defaultMessage);

public FieldError(String objectName, String field, 
									@Nullable Object rejectedValue, 
									boolean bindingFailure, @Nullable String[] codes, 
									@Nullable Object[] arguments, @Nullable String defaultMessage)
```

파라미터 목록

- objectName : 오류가 발생한 객체 이름
- field : 오류 필드
- rejectedValue : 사용자가 입력한 값(거절된 값)
- bindingFailure : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- codes : 메시지 코드
- arguments : 메시지에서 사용하는 인자
- defaultMessage : 기본 오류 메시지

ObjectError 도 유사하게 두 가지 생성자를 제공한다. 코드를 참고하자.

### 오류 발생시 사용자 입력 값 유지

```java
new FieldError("item", "price", item.getPrice(), false, null, null, 
							"가격은 1,000 ~1,000,000 까지 허용합니다.")
```

사용자의 입력 데이터가 컨트롤러의 `@ModelAttribute` 에 바인딩되는 시점에 오류가 발생하면 모델 객체에 사용자 입력 값을 유지하기 어렵다. 

예를 들어서 가격에 숫자가 아닌 문자가 입력된다면 가격은 `Integer` 타입이므로 문자를 보관할 수 있는 방법이 없다. 

그래서 오류가 발생한 경우 사용자 입력 값을 보관하는 별도의 방법이 필요하다. 

그리고 이렇게 보관한 사용자 입력 값을 검증 오류 발생시 화면에 다시 출력하면 된다. 

`FieldError` 는 오류 발생시 사용자 입력 값을 저장하는 기능을 제공한다 

여기서 `rejectedValue` 가 바로 오류 발생시 사용자 입력 값을 저장하는 필드다. 

`bindingFailure` 는 타입 오류 같은 바인딩이 실패했는지 여부를 적어주면 된다. 

여기서는 바인딩이 실패한 것은 아니기 때문에 `false` 를 사용한다.

### 타임리프의 사용자 입력 값 유지

- `th:field="*{price}"`
    - 타임리프의 `th:field` 는 매우 똑똑하게 동작합니다.
    - 정상 상황에는 모델 객체의 값을 사용하지만,
    - 오류가 발생하면 `FieldError` 에서 보관한 값을 사용해서 값을 출력한다.

### 스프링의 바인딩 오류 처리

- 타입 오류로 바인딩에 실패하면 스프링은 `FieldError` 를 생성하면서 사용자가 입력한 값을 넣어둔다.
- 그리고 해당 오류를 `BindingResult` 에 담아서 컨트롤러를 호출한다.
- 따라서 타입 오류 같은 바인딩 실패시에도 사용자의 오류 메시지를 정상 출력할 수 있다.

우리가 위에서 price 에 문자를 넣었을 때 결과 화면을 다시 볼까요?

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cfaca4a0-b882-40dd-836b-609eee58103b/Untitled.png)

메시지가 사용자 입장에서는 굉장히 불친절합니다! 이러한 오류 메시지를 어떻게 처리해야 할지 스프링에서 좋은 기능을 제공합니다.

다음 글에서는 오류 코드와 메시지 처리 에 대해서 알아봅니다



# 9. 오류 코드와 메시지 처리 1

우리는  오류 메시지를 체계적으로 다루는 법을 익힐 것입니다.

`FieldError` 생성자

FieldError 는 두 가지 생성자를 제공한다.

```java

public FieldError(String objectName, String field, String defaultMessage);
public FieldError(String objectName, String field, 
									@Nullable Object rejectedValue, boolean bindingFailure, 
									@Nullable String[] codes, @Nullable Object[] arguments, 
									@Nullable String defaultMessage)
```

파라미터 목록

- `objectName` : 오류가 발생한 객체 이름
- `field` : 오류 필드
- `rejectedValue` : 사용자가 입력한 값(거절된 값)
- `bindingFailure` : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- `codes` : 메시지 코드
- `arguments` : 메시지에서 사용하는 인자
- `defaultMessage` : 기본 오류 메시지

`FieldError` , `ObjectError` 의 생성자는 `codes` , `arguments` 를 제공한다. 이것은 오류 발생시 오류 코드로 메시지를 찾기 위해 사용된다.

errors 메시지 파일 생성

- `messages.properties` 를 사용해도 되지만, 오류 메시지를 구분하기 쉽게 `errors.properties` 라는
별도의 파일로 관리해보자.

먼저 스프링 부트가 해당 메시지 파일을 인식할 수 있게 다음 설정을 추가한다. 

이렇게 하면 `messages.properties` , `errors.properties` 두 파일을 모두 인식한다. 

(생략하면 `messages.properties` 를 기본으로 인식한다.)

스프링 부트 메시지 설정 추가 - `application.properties`

```java
spring.messages.basename=messages,errors
```

`errors.properties` 추가 - `src/main/resources/errors.properties`

```java
required.item.itemName=상품 이름은 필수입니다.
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
max.item.quantity=수량은 최대 {0} 까지 허용합니다.
totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}
```

참고: errors_en.properties 파일을 생성하면 오류 메시지도 국제화 처리를 할 수 있다.

이제 errors 에 등록한 메시지를 사용하도록 코드를 변경해보자.

`ValidationItemControllerV2` - `addItemV3()` 추가

```java
@PostMapping("/add")
public String addItemV3(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

    if (!StringUtils.hasText(item.getItemName())) {
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));
    }
    if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));
    }
    if (item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, new String[]{"max.item.quantity"}, new Object[]{9999}, null));
    }

    // 특정 필드가 아닌 복합 룰 검증
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.addError(new ObjectError("item", new String[]{"totalPriceMin"}, new Object[]{10000, resultPrice}, null));
        }
    }

    // 검증에 실패하면 다시 입력 폼으로
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

핵심 부분을 뜯어 봅시다.

```java
//range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
new FieldError("item", "price", item.getPrice(), false, 
								new String[] {"range.item.price"}, new Object[]{1000, 1000000}
```

- `codes` : `required.item.itemName` 를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라
배열로 여러 값을 전달할 수 있는데, 순서대로 매칭해서 처음 매칭되는 메시지가 사용된다.
- `arguments` : `Object[]{1000, 1000000}` 를 사용해서 코드의 `{0}` , `{1}` 로 치환할 값을 전달한다

실행

실행해보면 메시지, 국제화에서 학습한 `MessageSource` 를 찾아서 메시지를 조회하는 것을 확인할 수 있습니다.

# 10. 오류 코드와 메시지 처리 2

그런데 `FieldError` , `ObjectError` 는 다루기 너무 번거롭습니다.

혹시 오류 코드도 좀 더 자동화 할 수 있지 않을까요? 예) `item.itemName` 과 같은 형식으로 말이죠!

컨트롤러에서 `BindingResult` 는 검증해야 할 객체인 `target` 바로 다음에 와야 합니다. 우리의 경우 model 이었죠.

즉, `BindingResult` 는 이미 본인이 검증해야 할 객체인 `target` 을 알고 있다는 뜻이 됩니다.

다음을 컨트롤러에서 실행해보자.

```java
log.info("objectName={}", bindingResult.getObjectName());
log.info("target={}", bindingResult.getTarget()
```

결과

```java
objectName=item // @ModelAttribute name
target=Item(id=null, itemName=상품, price=100, quantity=1234)
```

### **rejectValue(), reject()**

- `BindingResult`가 제공하는 `rejectValue()`, `reject()`를 사용하면 `FieldError`, `ObjectError`를 직접 생성하지 않고 깔끔하게 검증 오류를 다룰 수 있다.

`ValidationItemControllerV2` - `addItemV4()` 추가

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
    if (item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.rejectValue("quantity", "max", new Object[]{9999}, null);
    }

    // 특정 필드가 아닌 복합 룰 검증
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
        }
    }

    // 검증에 실패하면 다시 입력 폼으로
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

이렇게 변경해도 오류 메시지가 정상 출력됩니다.

이상한 점은 우리는 `errors.properties` 에 있는 코드를 직접 입력하지 않았다는 점입니다.

**rejectValue()**

```java
void rejectValue(@Nullable String field, String errorCode, 
								@Nullable Object[] errorArgs, @Nullable String defaultMessage);
```

**reject()**

```java
void reject(String errorCode, @Nullable Object[] errorArgs, @Nullable String defaultMessage);
```

- `field`: 오류 필드명
- `errorCode`: 오류 코드(메시지에 등록된 오류 코드가 아닌 `messageResolver`를 위한 오류 코드)
- `errorArgs`: 오류 메시지에서 `{0}`을 치환하기 위한 값
- `defaultMessage`: 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

```java
bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null)
```

`BindingResult`는 어떤 객체를 대상으로 검증하는지 `target`을 이미 알고 있으므로 `target(item)`에 대한 정보는 없어도 됩니다.

우리가 `FieldError()` 를 직접 다룰 때는 오류 코드를 `range.item.price` 처럼 `errors.properties` 에 있는 key 값을 접누 입력해서 사용했습니다.

그런데  `rejectValue()` 을 사용하고 부터는 오류 코드를 `range` 로 간단하게 입력했습니다. 그런데도 오류 메시지를 찾아서 출력하는 모습입니다.

어떤 규칙으로 이를 가능하게 하는 것일까요?  이를 알기 위해서는 `MessageCodesResolver`  을 이해해야 합니다.

# 11. 오류 코드와 메시지 처리 3

오류 코드를 만들 때 자세히 만들 수도 있고 간단히 만들 수도 있습니다.

- 자세히
    - required.item.itemName=상품 이름은 필수입니다.
- 간단히
    - required=필수 값입니다.

예를 들어서 required 라고 오류 코드를 사용한다고 가정해보자. 

그런데 오류 메시지에 required.item.itemName 와 같이 객체명과 필드명을 조합한 세밀한 메시지 코드가 있다면 이 메시지를 높은 우선 순위로 사용합니다.

```java
#Level 1
required.item.itemName=상품 이름은 필수입니다.

#Level 2
required=필수 값입니다.
```

물론 이렇게 객체명과 필드명을 조합한 메시지가 있는지 우선 확인하고, 없으면 좀 더 범용적인 메시지를 선택하도록 추가 개발을 해야겠지만, 범용성 있게 잘 개발해두면, 메시지의 추가 만으로 매우 편리하게 오류 메시지를 관리할 수 있을 것입니다.

스프링의 `MessageCodeResolver` 이 이러한 기능을 지원하는 데 이를 테스트 코드로 알아봅시다.

# 12. 오류 코드와 메시지 처리 4

**`MessageCodesResolver`**

```java
package hello.itemservice.message.validation;

import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.validation.DefaultMessageCodesResolver;
import org.springframework.validation.MessageCodesResolver;

import static org.assertj.core.api.Assertions.*;

public class MessageCodesResolverTest {

    MessageCodesResolver codesResolver = new DefaultMessageCodesResolver();

    @Test
    void messageCodesResolverObject() {
        String[] messageCodes = codesResolver.resolveMessageCodes("required", "item");
        assertThat(messageCodes).containsExactly("required.item", "required");
    }

    @Test
    void messageCodeResolverField() {
        String[] messageCodes = codesResolver.resolveMessageCodes("required", "item", "itemName", String.class);
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

- 검증 오류 코드로 메시지 코드들을 생성
- `MessageCodesResolver`는 인터페이스이고 `DefaultMessageCodesResolver`가 기본 구현체
- 주로 `ObjectError`, `FieldError`와 함께 사용

### **DefaultMessageCodesResolver의 기본 메시지 생성 규칙**

**객체 오류**

```java
객체 오류의 경우 다음 순서로 2가지 생성
1. code + "." + object name
2. code

예) code: "required", object name: item
1. required.item
2. required
```

**필드 오류**

```java
필드 오류의 경우 다음 순서로 4가지 메시지 코드 생성
1. code + "." + object name + "." + field
2. code + "." + field
3. code + "." + field type
4. code

예) code: typeMismatch, object name: "user", field: "age", field type: int
1. "typeMismatch.user.age"
2. "typeMismatch.age"
3. "typeMismatch.int"
4. "typeMismatch"
```

**동작 방식**

- `rejectValue()`, `reject()` 는 내부에서 `MessageCodesResolver`로 메시지 코드들을 생성
- `FieldError`, `ObjectError` 의 생성자를 보면, 오류 코드를 하나가 아니라 여러 오류 코드를 가질 수 있다. `MessageCodesResolver`를 통해서 생성된 순서대로 오류 코드를 보관합니다.

이 부분을 `BindingResult` 의 로그로 확인해보면 아래와 같이 결과를 얻습니다.

```java
log.info("errors={}', bindingResult);
```

결과

```java
Field error in object 'item' on field 'itemName': rejected value []; 
codes [required.item.itemName,
			required.itemName, 
			required.java.lang.String,
			required]; arguments []; default message [null]
```

```java
Error in object 'item': 
codes [totalPriceMin.item, 
				totalPriceMin]; arguments [10000,0]; default message [null]
```

위와 같이 `FieldError` (`rejectValue(”itemName”, “required”)`) 와 
`ObjectError` (`reject(”totalPriceMin”)` ) 을 확인할 수 있습니다.

### **오류 메시지 출력**

- 타임리프 화면을 렌더링 할 때 `th:errors`가 실행
- 만약 오류가 있다면 생성된 오류 메시지 코드를 순서대로 돌아가면서 메시지를 검색, 없으면 디폴트 메시지를 출력

# 13. 오류 코드와 메시지 처리 5

### **오류 코드 관리 전략**

- `MessageCodesResolver`는 `required.item.itemName`처럼 구체적인 것을 먼저 만들어주고 나서, `required`처럼 덜 구체적인 것을 가장 나중에 생성합니다.
- 이렇게 크게 중요하지 않은 메시지는 범용성 있는 `requried` 같은 메시지로 끝내고, 정말 중요한 메시지는 꼭 필요할 때 구체적으로 적어서 사용하는 방식이 더 효과적입니다.

그렇다면 실제 실습을 통해 오류 코드 전략을 도입해봅시다.

`errors.properties`

```java
#required.item.itemName=상품 이름은 필수입니다.
#range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
#max.item.quantity=수량은 최대 {0} 까지 허용합니다.
#totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}

#==ObjectError==
#Level1
totalPriceMin.item=상품의 가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}

#Level2 - 생략
totalPriceMin=전체 가격은 {0}원 이상이어야 합니다. 현재 값 = {1}

#==FieldError==
#Level1
required.item.itemName=상품 이름은 필수입니다.
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
max.item.quantity=수량은 최대 {0} 까지 허용합니다.

#Level2 - 생략

#Level3
required.java.lang.String = 필수 문자입니다.
required.java.lang.Integer = 필수 숫자입니다.
min.java.lang.String = {0} 이상의 문자를 입력해주세요.
min.java.lang.Integer = {0} 이상의 숫자를 입력해주세요.
range.java.lang.String = {0} ~ {1} 까지의 문자를 입력해주세요.
range.java.lang.Integer = {0} ~ {1} 까지의 숫자를 입력해주세요.
max.java.lang.String = {0} 까지의 문자를 허용합니다.
max.java.lang.Integer = {0} 까지의 숫자를 허용합니다.

#Level4
required = 필수 값 입니다.
min= {0} 이상이어야 합니다.
range= {0} ~ {1} 범위를 허용합니다.
max= {0} 까지 허용합니다.
```

크게 객체 오류와 필드 오류를 나누었다. 그리고 범용성에 따라 레벨을 나누어두었다. `itemName` 의 경우 `required` 검증 오류 메시지가 발생하면 다음 코드 순서대로 메시지가 생성된다.

1. `required.item.itemName`
2. `required.itemName`
3. `required.java.lang.String`
4. `required`

그리고 이렇게 생성된 메시지 코드를 기반으로 순서대로 `MessageSource` 에서 메시지에서 찾는다. 구체적인 것에서 덜 구체적인 순서대로 찾는다. 메시지에 1번이 없으면 2번을 찾고, 2번이 없으면 3번을 찾는다.

이렇게 되면 만약에 크게 중요하지 않은 오류 메시지는 기존에 정의된 것을 그냥 재활용 하면 된다!

실행

- Level 1 전부 주석해보기
- Level 2,3 전부 주석해보기
- Level 4 전부 주석해보기 - 못 찾으면 코드에 작성한 디폴트 메시지를 사용한다.

Object 오류도 Level1, Level2로 재활용 가능하다.

우리가 의도한 대로 적절한 결과가 나오는 것을 확인할 수 있습니다!!

### **ValidationUtils**

`ValidationUtils` 사용 전

```java
if (!StringUtils.hasText(item.getItemName())) {
		bindingResult.rejectValue("itemName", "required", "기본: 상품 이름은 필수입니다.");
}
```

`ValidationUtils` 사용 후

`ValidationUtils` 을 사용하면 아래처럼 한줄로 가능해집니다. 제공하는 기능은 `Empty` 혹은 공백 같은 단순한 기능만 제공합니다.

```java
ValidationUtils.rejectIfEmptyOrWhitespace(bindingResult, "itemName", "required");
```

### 정리

1. `rejectValue()` 호출
2. `MessageCodesResolver` 를 사용해서 검증 오류 코드로 메시지 코드들을 생성
3. `new FieldError()` 를 생성하면서 메시지 코드들을 보관
4. `th:erros` 에서 메시지 코드들로 메시지를 순서대로 메시지에서 찾고, 보여준다.


# 14. 오류 코드와 메시지 처리 6

### **스프링이 직접 만든 오류 메시지 처리**

- 검증 오류 코드는 다음과 같이 2가지로 나눌 수 있습니다.
    1. 개발자가 직접 설정한 오류 코드 → `rejectValue()`를 직접 호출
    2. 스프링이 직접 검증 오류에 추가한 경우(주로 타입 정보가 맞지 않음)

이제부터 지금까지 배운 메시지 코드 전략의 강점을 확인해봅시다.

`price` 필드에 문자 “A” 을 입력했을 때

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e61ee99e-7aec-4276-8830-46a73bf7f271/Untitled.png)

로그를 확인해보면 `BindingResult` 에 `FieldError` 가 담겨있고, 아래 메시지 코드들이 생성된 것을 확인할 수 있습니다.

```java
codes[typeMismatch.item.price,
			typeMismatch.price,
			typeMismatch.java.lang.Integer,
			typeMismatch]
```

다음과 같이 4가지 메시지 코드가 입력되어 있다.

- `typeMismatch.item.price`
- `typeMismatch.price`
- `typeMismatch.java.lang.Integer`
- `typeMismatch`

스프링은 타입 오류가 발생하면 알아서 `typeMismatch` 라는 오류 코드를 사용합니다! 이 오류 코드가 `MessageCodeResolver` 을 통과하면서 4자기 메시지 코드가 실행됩니다.!

우리는 아직 `errors.properties` 에 메시지 코드가 없기 때문에 스프링이 생성한 기본 메시지가 출력되는 것이지요.

그렇다면 `errors.properties` 에 아래 내용을 추가해봅시다.

```java
typeMismatch.java.lang.Integer=숫자를 입력해주세요.
typeMismatch=타입 오류입니다.
```

이후 다시 실행하면 아래와 같은 결과가 됩니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/93412351-1d0b-4958-bf55-a6f269538ef0/Untitled.png)

결과적으로 우리는 소스코드를 하나도 건드리지 않고 원하는 메시지를 단계별로 설정할 수 있는 것입니다.

# 15. Validator 분리 1

우리는 복잡한 검증 로직을 별도로 분리할 필요가 있습니다.

- 컨트롤러에서 검증 로직이 차지하는 부분은 매우 큽니다.
- 이런 경우 별도의 클래스로 역할을 분리하는 것이 좋습니다.
- 그리고 이렇게 분리한 검증 로직을 재사용할 수도 있습니다.

`ItemValidator` 만들기

```java
package hello.itemservice.web.validation;

@Component
public class ItemValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
        // itme == clazz
        // itme == subItem
    }

    @Override
    public void validate(Object target, Errors errors) {
        Item item = (Item) target;

        if (!StringUtils.hasText(item.getItemName())) {
            errors.rejectValue("itemName", "required");
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
            errors.rejectValue("price", "range", new Object[]{1000, 1000000}, null);
        }
        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
            errors.rejectValue("quantity", "max", new Object[]{9999}, null);
        }

        // 특정 필드가 아닌 복합 룰 검증
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                errors.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
            }
        }

    }
}
```

스프링은 검증을 체계적으로 제공하기 위해 `Validator` 인터페이스를 제공합니다.

### **Validator 인터페이스**

```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```

- `supports()`: 해당 검증기를 지원하는 여부 확인
- `validate(Object target, Errors errors)`: 검증 대상 객체와 `BindingResult`

`ValidationItemControllerV2` - `addItemV5()` - `ItemValidator` 직접 호출

```java
@Controller
@RequestMapping("/validation/v2/items")
@RequiredArgsConstructor
@Slf4j
public class ValidationItemControllerV2 {

    private final ItemRepository itemRepository;
    private final ItemValidator itemValidator;

...

		@PostMapping("/add")
    public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

        itemValidator.validate(item, bindingResult);

        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v2/addForm";
        }

        // 성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
    }

		...
}
```

`ItemValidator` 를 스프링 빈으로 주입 받아서 직접 호출했습니다.

실행

실행해보면 기존과 완전히 동일하게 동작하는 것을 확인할 수 있다. 검증과 관련된 부분이 깔끔하게 분리되었습니다!!!!!

# 16. Validator 분리 2

스프링이 `Validator` 인터페이스를 별도로 제공하는 이유는 체계적으로 검증 기능을 도입하기 위함입니다.

그런데 앞에서는 검증기를 직접 불러서 사용했고, 이렇게 사용해도 됩니다.

그런데 `Validator` 인터페이스를 사용해서 검증기를 만들면 스프링의 추가적인 도움을 받을 수 있습니다.

`ValidationItemControllerV2` 에  추가 - `WebDataBinder`

```java
@InitBinder
public void init(WebDataBinder dataBinder) {
    log.info("init binder {}", dataBinder);
    dataBinder.addValidators(itemValidator);
}
```

- `WebDataBinder`는 스프링의 파라미터 바인딩의 역할을 해주고 검증 기능도 내부에 포함
- 이렇게 `WebDataBinder`에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용
- `@InitBinder`: 해당 컨트롤러에만 영향. 글로벌 설정은 별도

`ValidationItemControllerV2` - `addItemV6()`  (`@Validated` 적용 )

```java
@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

validator를 직접 호출하는 부분이 사라지고, 대신에 검증 대상 앞에 `@Validated` 가 붙었다.

- 이 애노테이션이 붙으면 앞서 `WebDataBinder`에 등록한 검증기를 찾아서 실행
- 그런데 여러 검증기를 등록한다면 각 검증기의 `supports()` 사용하여 구분
- 아래와 같은 검증기에서는 `supports(Item.class)`가 호출되고, 결과가 `true`이므로 `ItemValidator`의 `validate()`가 호출

실행

기존과 동일하게 잘 동작하는 것을 확인할 수 있다.

동작 방식

`@Validated` 는 검증기를 실행하라는 애노테이션이다.

이 애노테이션이 붙으면 앞서 `WebDataBinder` 에 등록한 검증기를 찾아서 실행한다. 

그런데 여러 검증기를 등록한다면 그 중에 어떤 검증기가 실행되어야 할지 구분이 필요하다. 

우리가 만든 `init(WebDataBinder dataBinder)` 이 함수가 검증기가 실행되는 것을 결정했다.

그런데 이 검증기가 실제로 실행될 수 있는지, 적절한지도 판단해야 한다.

이때 `supports()` 가 사용된다.

여기서는 `supports(Item.class)` 호출되고, 결과가 `true` 이므로 `ItemValidator` 의 `validate()` 가 호출된다 

```java
@Component
public class ItemValidator implements Validator {
  
    @Override
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
    }
  
    @Override
    public void validate(Object target, Errors errors) {...}
}
```

### **글로벌 설정 - 모든 컨트롤러에 다 적용**

```java
@SpringBootApplication
public class ItemServiceApplication implements WebMvcConfigurer {
  
    public static void main(String[] args) {
        SpringApplication.run(ItemServiceApplication.class, args);
    }
  
    @Override
    public Validator getValidator() {
        return new ItemValidator();
    }
}
```

주의

- 글로벌 설정을 하면 다음에 설명할 `BeanValidator` 가 자동 등록되지 않음
- 참고로 글로벌 설정을 직접 사용하는 경우는 드묾

참고

검증시 `@Validated` `@Valid` 둘다 사용가능하다.

`javax.validation.@Valid` 를 사용하려면 `build.gradle` 의존관계 추가가 필요하다.

`implementation 'org.springframework.boot:spring-boot-starter-validation'`
 는 `@Validated` 는 스프링 전용 검증 애노테이션이고, `@Valid` 는 자바 표준 검증 애노테이션이다.

자세한 내용은 다음 Bean Validation에서 설명하겠다.


#
#
#


# 1. Bean Validation 소개,

### Bean Validation - 소개

검증 기능을 지금처럼 매번 코드로 작성하는 것은 상당히 번거롭습니다. 

특히 특정 필드에 대한 검증 로직은 대부분 빈 값인지 아닌지, 특정 크기를 넘는지 아닌지와 같이 매우 일반적인 로직입니다.. 다음 코드를 봅시다.

`Item`

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
    @Range(min=1000, max = 1000000)
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

이런 검증 로직을 모든 프로젝트에 적용할 수 있게 공통화하고, 표준화 한 것이 바로 Bean Validation
이다.

Bean Validation을 잘 활용하면, 애노테이션 하나로 검증 로직을 매우 편리하게 적용할 수 있다.

### Bean Validation 이란?

먼저 Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다.

쉽게 이야기해서 **검증 애노테이션과 여러 인터페이스의 모음**이다. 

마치 JPA가 표준 기술이고 그 구현체로 하이버네이트가 있는 것과 같다.

Bean Validation을 구현한 기술중에 일반적으로 사용하는 구현체는 하이버네이트 Validator이다. 

이름이 하이버네이트가 붙어서 그렇지 ORM과는 관련이 없다.

### 하이버네이트 Validator 관련 링크

공식 사이트: [http://hibernate.org/validator/](http://hibernate.org/validator/)

공식 메뉴얼: [https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/)

검증 애노테이션 모음:  [https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/)
html_single/#validator-defineconstraints-spec

### Bean Validation - 시작

Bean Validation 기능을 어떻게 사용하는지 코드로 알아보자. 먼저 스프링과 통합하지 않고, 순수한 Bean
Validation 사용법 부터 테스트 코드로 알아보자.

### Bean Validation 의존관계 추가

`build.gradle`

```java
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

spring-boot-starter-validation 의존관계를 추가하면 라이브러리가 추가 된다.

**Jakarta Bean Validation**

`jakarta.validation-api` : Bean Validation 인터페이스

`hibernate-validator` 구현체

### 테스트 코드 작성

`Item` - Bean Validation 애노테이션 적용

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
    @Range(min=1000, max = 1000000)
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

**검증 애노테이션**

`@NotBlank` : 빈값 + 공백만 있는 경우를 허용하지 않는다.

`@NotNull` : null 을 허용하지 않는다.

`@Range(min = 1000, max = 1000000)` : 범위 안의 값이어야 한다.

`@Max(9999)` : 최대 9999까지만 허용한다.

**참고**

`javax.validation.constraints.NotNull`

`org.hibernate.validator.constraints.Range`

`javax.validation` 으로 시작하면 특정 구현에 관계없이 제공되는 표준 인터페이스이고, `org.hibernate.validator` 로 시작하면 하이버네이트 validator 구현체를 사용할 때만 제공되는 검증 기능이다. 

**실무에서 대부분 하이버네이트 validator를 사용하므로 자유롭게 사용해도 된다**

`BeanValidationTest` - Bean Validation 테스트 코드 작성

```java
package hello.itemservice.message.validation;

import hello.itemservice.domain.item.Item;
import org.junit.jupiter.api.Test;

import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;
import java.util.Set;

public class BeanValidationTest {

    @Test
    void beanValidation(){
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        Item item = new Item();
        item.setItemName(" ");
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

**검증기 생성**

다음 코드와 같이 검증기를 생성한다. 이후 스프링과 통합하면 우리가 직접 이런 코드를 작성하지는 않으므로, 이렇게 사용하는구나 정도만 참고하자.

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
Validator validator = factory.getValidator();
```

**검증 실행**

검증 대상( item )을 직접 검증기에 넣고 그 결과를 받는다. Set 에는 ConstraintViolation 이라는 검증 오류가 담긴다. 따라서 결과가 비어있으면 검증 오류가 없는 것이다.

```java
Set<ConstraintViolation<Item>> violations = validator.validate(item);
```

**실행결과** 

```java
violation={
		interpolatedMessage='공백일 수 없습니다', 
		propertyPath=itemName,
		rootBeanClass=class hello.itemservice.domain.item.Item,
		messageTemplate='{javax.validation.constraints.NotBlank.message}'}

violation.message=공백일 수 없습니다

violation={
		interpolatedMessage='9999 이하여야 합니다', 
		propertyPath=quantity,
		rootBeanClass=class hello.itemservice.domain.item.Item,
		messageTemplate='{javax.validation.constraints.Max.message}'}

violation.message=9999 이하여야 합니다

violation={
		interpolatedMessage='1000에서 1000000 사이여야 합니다',
		propertyPath=price, rootBeanClass=class hello.itemservice.domain.item.Item,
		messageTemplate='{org.hibernate.validator.constraints.Range.message}'}

violation.message=1000에서 1000000 사이여야 합니다
```

`ConstraintViolation` 출력 결과를 보면, 검증 오류가 발생한 객체, 필드, 메시지 정보등 다양한 정보를 확인할 수 있다.

**정리**

이렇게 빈 검증기(Bean Validation)를 직접 사용하는 방법을 알아보았다. 아마 지금까지 배웠던 스프링 MVC 검증 방법에 빈 검증기를 어떻게 적용하면 좋을지 여러가지 생각이 들 것이다. 

**스프링은 이미 개발자를 위해 빈 검증기를 스프링에 완전히 통합해두었다.**

# 2. 프로젝트 준비 V3

앞서 **4. 검증1**  시리즈에서 만든 기능을 유지하기 위해, 컨트롤러와 템플릿 파일을 복사하자.

`ValidationItemControllerV3` 컨트롤러 생성

`hello.itemservice.web.validation.ValidationItemControllerV2` 복사 후에  `hello.itemservice.web.validation.ValidationItemControllerV3` 붙여넣기

URL 경로도 `validation/v2/` 에서 `validation/v3/` 으로 바꾼다.

**템플릿 파일 복사**

`validation/v2` 디렉토리의 모든 템플릿 파일을 `validation/v3` 디렉토리로 복사

`/resources/templates/validation/v2/` 을 `/resources/templates/validation/v3/` 로

`addForm.html` , `editForm.html` , `item.html` , `items.html`

`/resources/templates/validation/v3/` 하위 4개 파일 모두 URL 경로 변경: 

`validation/v2/` 에서 `validation/v3/` 으로

`addForm.html` , `editForm.html` , `item.html` , `items.html`

이렇게 변경한 후 바로 Bean Validation 스프링을 적용해봅시다.


# 3. Bean Validation 스프링 적용

`ValidationItemControllerV3` 코드 수정

```java
package hello.itemservice.web.validation;

import hello.itemservice.domain.item.Item;
import hello.itemservice.domain.item.ItemRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.util.StringUtils;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.validation.ObjectError;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.util.List;

@Controller
@RequestMapping("/validation/v3/items")
@RequiredArgsConstructor
@Slf4j
public class ValidationItemControllerV3 {

    private final ItemRepository itemRepository;

    @GetMapping
    public String items(Model model) {
        List<Item> items = itemRepository.findAll();
        model.addAttribute("items", items);
        return "validation/v3/items";
    }

    @GetMapping("/{itemId}")
    public String item(@PathVariable long itemId, Model model) {
        Item item = itemRepository.findById(itemId);
        model.addAttribute("item", item);
        return "validation/v3/item";
    }

    @GetMapping("/add")
    public String addForm(Model model) {
        model.addAttribute("item", new Item());
        return "validation/v3/addForm";
    }

    @PostMapping("/add")
    public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v3/addForm";
        }

        // 성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }

    @GetMapping("/{itemId}/edit")
    public String editForm(@PathVariable Long itemId, Model model) {
        Item item = itemRepository.findById(itemId);
        model.addAttribute("item", item);
        return "validation/v3/editForm";
    }

    @PostMapping("/{itemId}/edit")
    public String edit(@PathVariable Long itemId, @ModelAttribute Item item) {
        itemRepository.update(itemId, item);
        return "redirect:/validation/v3/items/{itemId}";
    }

}
```

기존에 등록한 `ItemValidator`를 제거했습니다.

`init(WebDataBinder dataBinder)` 메서드도 제거했습니다.

실행

[http://localhost:8080/validation/v3/items](http://localhost:8080/validation/v3/items) 

실행해보면 애노테이션 기반의 Bean Validation이 정상 동작하는 것을 확인할 수 있다.

참고

특정 필드의 범위를 넘어서는 검증(가격 * 수량의 합은 10,000원 이상) 기능이 빠졌는데, 이 부분은 조금 뒤에 설명한다

### 스프링 MVC는 어떻게 Bean Validator를 사용?

스프링 부트가 `spring-boot-starter-validation` 라이브러리를 넣으면 자동으로 **Bean Validator**를
인지하고 스프링에 통합한다.

### 스프링 부트는 자동으로 글로벌 Validator로 등록한다.

`LocalValidatorFactoryBean` 을 글로벌 Validator로 등록한다. 이 Validator는 `@NotNull` 같은 애노테이션을 보고 검증을 수행한다. 

이렇게 글로벌 Validator가 적용되어 있기 때문에, `@Valid` , `@Validated` 만 적용하면 된다.

검증 오류가 발생하면, `FieldError` , `ObjectError` 를 생성해서 `BindingResult` 에 담아준다.

주의!
다음과 같이 직접 글로벌 Validator를 직접 등록하면 스프링 부트는 Bean Validator를 글로벌 `Validator` 로 등록하지 않는다. 따라서 애노테이션 기반의 빈 검증기가 동작하지 않는다. 

참고

검증시 `@Validated` `@Valid` 둘다 사용가능하다. `javax.validation.@Valid` 를 사용하려면 `build.gradle` 의존관계 추가가 필요하다. (이전에 추가했다.)

`implementation 'org.springframework.boot:spring-boot-starter-validation'`

`@Validated` 는 스프링 전용 검증 애노테이션이고, `@Valid` 는 자바 표준 검증 애노테이션이다. 

둘 중 아무거나 사용해도 동일하게 작동하지만, `@Validated` 는 내부에 `groups` 라는 기능을 포함하고 있다. 이 부분은 조금 뒤에 다시 설명하겠다.

### 검증 순서

1. `@ModelAttribute` 각각의 필드에 타입 변환 시도
    1. 성공하면 다음으로
    2. 실패하면 `typeMismatch` 로 `FieldError` 추가
2. Validator 적용

### 바인딩에 성공한 필드만 Bean Validation 적용

BeanValidator는 바인딩에 실패한 필드는 BeanValidation을 적용하지 않는다.

생각해보면 타입 변환에 성공해서 바인딩에 성공한 필드여야 BeanValidation 적용이 의미 있다. 

(일단 모델 객체에 바인딩 받는 값이 정상으로 들어와야 검증도 의미가 있다.)

`@ModelAttribute` → 각각의 필드 타입 변환시도 →  변환에 성공한 필드만 BeanValidation 적용

예)

- `itemName` 에 문자 "A" 입력 → 타입 변환 성공  → `itemName` 필드에 BeanValidation 적용
- `price` 에 문자 "A" 입력  → "A"를 숫자 타입 변환 시도 실패 →  typeMismatch FieldError 추가 → `price` 필드는 BeanValidation 적용 X


# 4. Bean Validation 에러 코드

Bean Validation이 기본으로 제공하는 오류 메시지를 좀 더 자세히 변경하고 싶다면?

Bean Validation을 적용하고 `bindingResult` 에 등록된 검증 오류 코드를 보자.

오류 코드가 애노테이션 이름으로 등록된다. 마치 `typeMismatch` 와 유사하다.

`NotBlank` 라는 오류 코드를 기반으로 `MessageCodesResolver` 를 통해 다양한 메시지 코드가 순서대로 생성된다.

**@NotBlank**

- NotBlank.item.itemName
- NotBlank.itemName
- NotBlank.java.lang.String
- NotBlank

**@Range**

- Range.item.price
- Range.price
- Range.java.lang.Integer
- Range

### 메시지 등록

`errors.properties` - 이제 메시지를 등록해보자.

```java
#Bean Validation 추가
NotBlank={0} 공백X
Range={0}, {2} ~ {1} 허용
Max={0}, 최대 {1}
```

`{0}` 은 필드명이고, `{1}` , `{2}` ...은 각 애노테이션 마다 다르다.

**실행**
실행해보면 방금 등록한 메시지가 정상 적용되는 것을 확인할 수 있다.

### BeanValidation 메시지 찾는 순서

1. 생성된 메시지 코드 순서대로 `messageSource` 에서 메시지 찾기
2. 애노테이션의 `message` 속성 사용 (예 -  `@NotBlank(message = "공백! {0}")` )
3. 라이브러리가 제공하는 기본 값을 사용한다. ( 예 -  공백일 수 없습니다.)

### 애노테이션의 message 사용 예

```java
@NotBlank(message = "공백은 입력할 수 없습니다.")
private String itemName;
```

# 5. Bean Validation 오브젝트 오류

Bean Validation에서 특정 필드( `FieldError` )가 아닌 해당 오브젝트 관련 오류( `ObjectError` )는
어떻게 처리할 수 있을까?

다음과 같이 `@ScriptAssert()` 를 사용하면 된다

```java
@Data
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >=10000")
public class Item {
 //...
}
```

실행해보면 정상 수행되는 것을 확인할 수 있다. 메시지 코드도 다음과 같이 생성된다.

### 메시지 코드

`ScriptAssert.item`

`ScriptAssert`

그런데 실제 사용해보면 제약이 많고 복잡하다. 그리고 실무에서는 검증 기능이 해당 객체의 범위를 넘어서는 경우들도 종종 등장하는데, 그런 경우 대응이 어렵다.

따라서 오브젝트 오류(글로벌 오류)의 경우 `@ScriptAssert` 을 억지로 사용하는 것 보다는 다음과 같이 오브젝트 오류 관련 부분만 직접 자바 코드로 작성하는 것을 권장한다

`ValidationItemControllerV3` - 글로벌 오류 추가

```java
@PostMapping("/add")
public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {

    // 특정 필드 예외가 아닌 전체 예외
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

    // 성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v3/items/{itemId}";
}
```

`@ScriptAssert` 부분은 제거한다.


# 6. Bean Validation 수정에 적용

상품 수정에도 빈 검증(Bean Validation)을 적용해보자.

수정에도 검증 기능을 추가하자

`ValidationItemControllerV3` - `edit()` 변경

```java
@PostMapping("/{itemId}/edit")
public String edit(@PathVariable Long itemId, @Validated @ModelAttribute Item item, BindingResult bindingResult) {

    // 특정 필드 예외가 아닌 전체 예외 (object 예외)
    if (item.getPrice() != null && item.getQuantity() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();

        if (resultPrice < 10000) {
            bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
        }
    }

    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v3/editForm";
    }

    itemRepository.update(itemId, item);
    return "redirect:/validation/v3/items/{itemId}";
}
```

`edit()` : `Item` 모델 객체에 `@Validated` 를 추가

검증 오류가 발생하면 `editForm` 으로 이동하는 코드 추가

`validation/v3/editForm.html` 변경

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
        <h2 th:text="#{page.updateItem}">상품 수정</h2>
    </div>

    <form action="item.html" th:action th:object="${item}" method="post">

        <div th:if="${#fields.hasGlobalErrors()}">
            <p class="field-error" th:each="err: ${#fields.globalErrors()}" th:text="${err}">글로벌 오류 메시지</p>
        </div>

        <div>
            <label for="id" th:text="#{label.item.id}">상품 ID</label>
            <input type="text" id="id" th:field="*{id}" class="form-control" readonly>
        </div>

        <div>
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
            <input type="text" id="itemName" th:field="*{itemName}" th:errorclass="field-error" class="form-control"
                   placeholder="이름을 입력하세요">
            <div class="field-error" th:errors="*{itemName}">상품명 오류</div>
        </div>

        <div>
            <label for="price" th:text="#{label.item.price}">가격</label>
            <input type="text" id="price" th:field="*{price}" th:errorclass="field-error" class="form-control"
                   placeholder="가격을 입력하세요">
            <div class="field-error" th:errors="*{price}">가격 오류</div>
        </div>

        <div>
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>
            <input type="text" id="quantity" th:field="*{quantity}" th:errorclass="field-error" class="form-control"
                   placeholder="수량을 입력하세요">
            <div class="field-error" th:errors="*{quantity}">수량 오류</div>
        </div>

        <hr class="my-4">

        <div class="row">
            <div class="col">
                <button class="w-100 btn btn-primary btn-lg" type="submit" th:text="#{button.save}">저장</button>
            </div>
            <div class="col">
                <button class="w-100 btn btn-secondary btn-lg"
                        onclick="location.href='item.html'"
                        th:onclick="|location.href='@{/validation/v3/items/{itemId}(itemId=${item.id})}'|"
                        type="button" th:text="#{button.cancel}">취소
                </button>
            </div>
        </div>

    </form>

</div> <!-- /container -->
</body>
</html>
```

`.field-error` css 추가

글로벌 오류 메시지

상품명, 가격, 수량 필드에 검증 기능 추가

[http://localhost:8080/validation/v3/items/1/edit](http://localhost:8080/validation/v3/items/1/edit) 실행 결과

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9ff3489b-7918-4f91-86bb-a5da8a8f4a48/Untitled.png)

잘 수정이 된 모습입니다.

# 7. Bean Validation 한계

### 수정시 검증 요구사항

데이터를 등록할 때와 수정할 때는 요구사항이 다를 수 있다.

**등록시 기존 요구사항**

- 타입 검증
    - 가격, 수량에 문자가 들어가면 검증 오류 처리
- 필드 검증
    - 상품명: 필수, 공백X
    - 가격: 1000원 이상, 1백만원 이하
    - 수량: 최대 9999
- 특정 필드의 범위를 넘어서는 검증
    - 가격 * 수량의 합은 10,000원 이상

**수정시 요구사항**

등록 시에는 `quantity` 수량을 최대 9999까지 등록할 수 있지만 수정시에는 수량을 무제한으로 변경할 수 있다.

등록 시에는 `id` 에 값이 없어도 되지만, 수정시에는 `id` 값이 필수이다.

### **수정 요구사항 적용**

수정시에는 `Item` 에서 `id` 값이 필수이고, `quantity` 도 무제한으로 적용할 수 있다.

`Item`

```java
package hello.itemservice.domain.item;

import lombok.Data;
import org.hibernate.validator.constraints.Range;
import org.hibernate.validator.constraints.ScriptAssert;

import javax.validation.constraints.Max;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;

@Data
public class Item {

    @NotNull
    private Long id;

    @NotBlank
    private String itemName;

    @NotNull
    @Range(min=1000, max = 1000000)
    private Integer price;

    @NotNull
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

수정 요구사항을 적용하기 위해 다음을 적용했다.

`id` : `@NotNull` 추가

`quantity` : `@Max(9999)` 제거

**참고**

현재 구조에서는 수정시 `item` 의 `id` 값은 항상 들어있도록 로직이 구성되어 있다. 그래서 검증하지 않아도 된다고 생각할 수 있다. 그런데 HTTP 요청은 언제든지 악의적으로 변경해서 요청할 수 있으므로 서버에서 항상 검증해야 한다. 

예를 들어서 HTTP 요청을 변경해서 `item` 의 `id` 값을 삭제하고 요청할 수도 있다. 따라서 최종 검증은 서버에서 진행하는 것이 안전한다.

### 수정을 실행

정상 동작을 확인할 수 있다.

**그런데 수정은 잘 동작하지만 등록에서 문제가 발생한다.**

등록시에는 `id` 에 값도 없고, `quantity` 수량 제한 최대 값인 9999도 적용되지 않는 문제가 발생한다.

**등록시 화면이 넘어가지 않으면서 다음과 같은 오류를 볼 수 있다.**

`'id': rejected value [null];`

왜냐하면 등록시에는 `id` 에 값이 없다. 따라서 `@NotNull id` 를 적용한 것 때문에 검증에 실패하고 다시 폼 화면으로 넘어온다. 결국 등록 자체도 불가능하고, 수량 제한도 걸지 못한다.

결과적으로 item 은 등록과 수정에서 검증 조건의 충돌이 발생하고, 등록과 수정은 같은 BeanValidation 을 적용할 수 없다. 

이 문제를 어떻게 해결할 수 있을까요?

# 8. Bean Validation - groups

동일한 모델 객체를 등록할 때와 수정할 때 각각 다르게 검증하는 방법을 알아보자.

**방법 2가지**

- BeanValidation의 groups 기능을 사용한다.
- Item을 직접 사용하지 않고, ItemSaveForm, ItemUpdateForm 같은 폼 전송을 위한 별도의 모델
객체를 만들어서 사용한다.

**BeanValidation groups 기능 사용**

이런 문제를 해결하기 위해 Bean Validation 은 groups 라는 기능을 제공한다.

예를 들어서 등록시에 검증할 기능과 수정시에 검증할 기능을 각각 그룹으로 나누어 적용할 수 있다. 

코드로 확인해보자.

### groups 적용

`SaveCheck` - 저장용 groups 생성

```java
package hello.itemservice.domain.item;

public interface SaveCheck {
}
```

`UpdateCheck` - 수정용 groups 생성

```java
package hello.itemservice.domain.item;

public interface UpdateCheck {
}
```

`Item` - groups 적용

```java
package hello.itemservice.domain.item;

import lombok.Data;
import org.hibernate.validator.constraints.Range;

import javax.validation.constraints.Max;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;

@Data
public class Item {

    @NotNull(groups = UpdateCheck.class) // 수정 시에만 적용
    private Long id;

    @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
    private String itemName;

    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
    @Range(min = 1000, max = 1000000, groups = {SaveCheck.class, UpdateCheck.class})
    private Integer price;

    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
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

`ValidationItemControllerV3` - 저장 로직에 SaveCheck Groups 적용

```java
@PostMapping("/add")
public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
		...
}
```

`addItem()` 를 복사해서 `addItemV2()` 생성, `SaveCheck.class` 적용

기존 `addItem()`  의 `@PostMapping("/add")` 주석처리

`ValidationItemControllerV3` - 수정 로직에 UpdateCheck Groups 적용

```java
@PostMapping("/{itemId}/edit")
public String editV2(@PathVariable Long itemId, @Validated(UpdateCheck.class)
											@ModelAttribute Item item, BindingResult bindingResult) {
											 //...
}
```

`edit()` 를 복사해서 `editV2()` 생성, `UpdateCheck.class` 적용

기존 `edit()` 의  `@PostMapping("/{itemId}/edit")` 주석처리

이렇게 적용하면 잘 실행되는 것을 확인 할 수 있습니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/06286d6d-8b39-4fcc-9441-e6c199cf52b4/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8489a2e4-0e15-4c28-9e02-07ec61d6181b/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b4a34333-877e-4495-a5c0-943ab3beddda/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3f54eaf2-0b3a-40f5-81ab-ce1f164ecb03/Untitled.png)

처음 Item 을 Add 할 때는 수량을 9999 개 보다 더 많이 하지 못했지만 

이 후 Item 을 Update 할 때는 수량을 9999 개 보다 더 많이 할 수 있는 모습입니다.

### 정리

groups 기능을 사용해서 등록과 수정시에 각각 다르게 검증을 할 수 있었다. 

그런데 groups 기능을 사용하니 Item 은 물론이고, 전반적으로 복잡도가 올라갔다.

사실 groups 기능은 실제 잘 사용되지는 않습니다!! 

그 이유는 실무에서는 주로 다음에 등장하는 등록용 폼 객체와 수정용 폼 객체를 분리해서 사용하기 때문이다.