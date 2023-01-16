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