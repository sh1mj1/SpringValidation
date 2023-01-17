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