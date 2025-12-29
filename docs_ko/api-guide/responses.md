---
source:
    - response.py
---

# Responses

> 기본적인 HttpResponse 객체와 달리, TemplateResponse 객체는  
> 뷰가 응답을 계산하기 위해 제공한 컨텍스트의 세부 정보를 유지합니다.  
> 응답의 최종 출력은 응답 처리 과정의 이후 단계에서, 실제로 필요할 때까지 계산되지 않습니다.
>
> &mdash; [Django 문서][cite]

REST framework는 `Response` 클래스를 제공하여 HTTP 콘텐츠 네고시에이션(content negotiation)을 지원합니다.  
이를 통해 클라이언트 요청에 따라 여러 콘텐츠 타입으로 렌더링될 수 있는 응답을 반환할 수 있습니다.

`Response` 클래스는 Django의 `SimpleTemplateResponse`를 상속합니다.  
`Response` 객체는 데이터와 함께 초기화되며, 이 데이터는 **네이티브 Python 기본 타입**으로 구성되어야 합니다.  
REST framework는 이후 표준 HTTP 콘텐츠 네고시에이션을 사용해 최종 응답 콘텐츠를 어떻게 렌더링할지 결정합니다.

`Response` 클래스를 반드시 사용해야 하는 것은 아닙니다.  
필요하다면 뷰에서 일반적인 `HttpResponse`나 `StreamingHttpResponse` 객체를 반환할 수도 있습니다.  
`Response` 클래스를 사용하면 여러 포맷으로 렌더링 가능한, 콘텐츠 네고시에이션 기반 Web API 응답을 보다 깔끔한 인터페이스로 반환할 수 있습니다.

특별한 이유로 REST framework를 크게 커스터마이징하지 않는 한,  
`Response` 객체를 반환하는 뷰에서는 항상 `APIView` 클래스나 `@api_view` 함수를 사용하는 것이 좋습니다.  
이렇게 하면 응답이 뷰에서 반환되기 전에 콘텐츠 네고시에이션을 수행하고,  
해당 응답에 적합한 renderer를 선택할 수 있습니다.

---

# 응답 생성하기

## Response()

**시그니처:** `Response(data, status=None, template_name=None, headers=None, content_type=None)`

일반적인 `HttpResponse` 객체와 달리,  
`Response` 객체는 이미 렌더링된 콘텐츠로 생성되지 않습니다.  
대신 렌더링되지 않은 데이터(unrendered data)를 전달하며,  
이 데이터는 어떤 Python 기본 타입이든 될 수 있습니다.

`Response` 클래스에서 사용하는 renderer는  
Django 모델 인스턴스와 같은 복잡한 데이터 타입을 기본적으로 처리할 수 없으므로,  
`Response` 객체를 생성하기 전에 데이터를 **기본 타입으로 직렬화**해야 합니다.

이 직렬화 작업에는 REST framework의 `Serializer` 클래스를 사용할 수도 있고,  
직접 구현한 커스텀 직렬화 로직을 사용할 수도 있습니다.

Arguments:

* `data`: 응답에 사용할 직렬화된 데이터
* `status`: 응답의 상태 코드. 기본값은 200입니다. 자세한 내용은 [상태 코드][statuscodes]를 참고하세요.
* `template_name`: `HTMLRenderer`가 선택된 경우 사용할 템플릿 이름
* `headers`: 응답에 사용할 HTTP 헤더 딕셔너리
* `content_type`: 응답의 콘텐츠 타입  
  일반적으로는 콘텐츠 네고시에이션에 따라 renderer가 자동으로 설정되지만,  
  특정 경우에는 명시적으로 지정해야 할 수도 있습니다.

---

# Attributes

## .data

렌더링되지 않은(unrendered) 상태의 직렬화된 응답 데이터입니다.

## .status_code

HTTP 응답의 숫자형 상태 코드입니다.

## .content

렌더링된 응답 콘텐츠입니다.  
`.content`에 접근하기 전에 반드시 `.render()` 메서드가 호출되어야 합니다.

## .template_name

지정된 `template_name` 값입니다.  
응답에 대해 `HTMLRenderer` 또는 다른 커스텀 템플릿 renderer가 선택된 경우에만 필요합니다.

## .accepted_renderer

응답을 렌더링하는 데 사용될 renderer 인스턴스입니다.

뷰에서 응답이 반환되기 직전에  
`APIView` 또는 `@api_view`에 의해 자동으로 설정됩니다.

## .accepted_media_type

콘텐츠 네고시에이션 단계에서 선택된 미디어 타입입니다.

뷰에서 응답이 반환되기 직전에  
`APIView` 또는 `@api_view`에 의해 자동으로 설정됩니다.

## .renderer_context

renderer의 `.render()` 메서드로 전달될  
추가 컨텍스트 정보를 담고 있는 딕셔너리입니다.

뷰에서 응답이 반환되기 직전에  
`APIView` 또는 `@api_view`에 의해 자동으로 설정됩니다.

---

# 표준 HttpResponse 속성

`Response` 클래스는 `SimpleTemplateResponse`를 확장하므로,  
일반적인 HttpResponse에서 제공하는 모든 속성과 메서드를 그대로 사용할 수 있습니다.

예를 들어, 다음과 같이 표준 방식으로 응답 헤더를 설정할 수 있습니다.

    response = Response()
    response['Cache-Control'] = 'no-cache'

## .render()

**시그니처:** `.render()`

다른 `TemplateResponse`와 마찬가지로,  
이 메서드는 직렬화된 응답 데이터를 최종 응답 콘텐츠로 렌더링하기 위해 호출됩니다.

`.render()`가 호출되면, 응답 콘텐츠는  
`accepted_renderer` 인스턴스의  
`.render(data, accepted_media_type, renderer_context)`  
메서드 호출 결과로 설정됩니다.

일반적으로는 Django의 표준 응답 처리 과정에서 자동으로 호출되므로,  
직접 `.render()`를 호출할 필요는 거의 없습니다.

[cite]: https://docs.djangoproject.com/en/stable/ref/template-response/
[statuscodes]: status-codes.md
