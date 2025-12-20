# Tutorial 2: Requests and Responses

이제부터는 REST framework의 핵심 내용을 본격적으로 다루기 시작합니다.  
여기서는 몇 가지 필수적인 구성 요소를 소개하겠습니다.

## Request objects

REST framework는 기존의 `HttpRequest`를 확장한 `Request` 객체를 제공합니다.  
이 객체는 더 유연한 요청 파싱 기능을 제공하며, 핵심 기능은 `request.data` 속성입니다.

`request.data`는 `request.POST`와 유사하지만, Web API 작업에 훨씬 유용합니다.

```python
request.POST  # 폼 데이터만 처리합니다. 'POST' 메서드에서만 동작합니다.
request.data  # 임의의 데이터를 처리합니다. 'POST', 'PUT', 'PATCH' 메서드에서 동작합니다.
```

## Response objects

REST framework는 또한 `Response` 객체를 제공합니다.  
이 객체는 렌더링되지 않은 콘텐츠를 받아, 콘텐츠 네고시에이션을 통해 클라이언트에 반환할 적절한 콘텐츠 타입을 결정하는 `TemplateResponse`의 한 형태입니다.

```python
return Response(data)  # 클라이언트가 요청한 콘텐츠 타입으로 렌더링됩니다.
```

## Status codes

뷰에서 숫자 기반의 HTTP 상태 코드를 직접 사용하는 것은 가독성이 떨어질 수 있고,  
잘못된 상태 코드를 사용해도 눈치채기 어려운 문제가 있습니다.

REST framework는 `status` 모듈에 `HTTP_400_BAD_REQUEST`와 같은 명확한 식별자를 제공합니다.  
숫자 대신 이러한 상수를 사용하는 것이 좋습니다.

## Wrapping API views

REST framework는 API view를 작성할 때 사용할 수 있는 두 가지 래퍼(wrapper)를 제공합니다.

1. 함수 기반 view에서 사용하는 `@api_view` 데코레이터
2. 클래스 기반 view에서 사용하는 `APIView` 클래스

이 래퍼들은 다음과 같은 기능을 제공합니다.

- view에서 `Request` 인스턴스를 전달받도록 보장
- 콘텐츠 네고시에이션을 수행할 수 있도록 `Response` 객체에 컨텍스트 추가
- 허용되지 않은 메서드에 대해 `405 Method Not Allowed` 응답 반환
- 잘못된 입력으로 `request.data` 접근 시 발생하는 `ParseError` 예외 처리

## Pulling it all together

이제 새로운 컴포넌트들을 사용해 기존 view를 약간 리팩터링해 보겠습니다.

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(["GET", "POST"])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == "GET":
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == "POST":
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

이 view는 이전 예제보다 더 개선되었습니다.  
코드는 더 간결해졌고, Django Forms API를 사용할 때와 매우 유사한 느낌을 줍니다.  
또한 이름이 지정된 상태 코드를 사용함으로써 응답의 의미도 더 명확해졌습니다.

다음은 개별 스니펫을 처리하는 view입니다 (`views.py`).

```python
@api_view(["GET", "PUT", "DELETE"])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == "GET":
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == "PUT":
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == "DELETE":
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

이 구조 역시 일반적인 Django view를 작성하는 방식과 크게 다르지 않습니다.

이제 요청이나 응답을 특정 콘텐츠 타입에 직접 묶지 않는다는 점에 주목하세요.  
`request.data`는 `json` 요청뿐만 아니라 다양한 형식을 처리할 수 있습니다.  
마찬가지로 우리는 데이터만 담은 `Response` 객체를 반환하고, 실제 렌더링은 REST framework가 적절한 콘텐츠 타입으로 처리합니다.

## Adding optional format suffixes to our URLs

이제 응답이 단일 콘텐츠 타입에 고정되지 않으므로,  
API 엔드포인트에 포맷 접미사(format suffix)를 지원하도록 해보겠습니다.

포맷 접미사를 사용하면 [<http://example.com/api/items/4.json>][json-url]과 같이  
특정 포맷을 명시적으로 나타내는 URL을 사용할 수 있습니다.

먼저 두 view 모두에 `format` 키워드 인자를 추가합니다.

`def snippet_list(request, format=None):`  
`def snippet_detail(request, pk, format=None):`

그 다음 `snippets/urls.py` 파일을 수정하여 기존 URL 패턴에 `format_suffix_patterns`를 추가합니다.

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path("snippets/", views.snippet_list),
    path("snippets/<int:pk>/", views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

이 설정은 필수는 아니지만, 특정 포맷을 명확하게 지정할 수 있는 간단하고 깔끔한 방법을 제공합니다.

## How's it looking?

[tutorial part 1][tut-1]에서 했던 것처럼 커맨드라인에서 API를 테스트해 보세요.  
동작 방식은 거의 동일하지만, 잘못된 요청을 보냈을 때 에러 처리가 더 깔끔해졌습니다.

이전과 마찬가지로 모든 스니펫 목록을 조회할 수 있습니다.

```bash
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
    {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
    },
    {
    "id": 2,
    "title": "",
    "code": "print(\"hello, world\")\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
    }
]
```

`Accept` 헤더를 사용해 응답 포맷을 제어할 수도 있습니다.

```bash
http http://127.0.0.1:8000/snippets/ Accept:application/json  # JSON 요청
http http://127.0.0.1:8000/snippets/ Accept:text/html         # HTML 요청
```

또는 포맷 접미사를 사용할 수도 있습니다.

```bash
http http://127.0.0.1:8000/snippets.json  # JSON 접미사
http http://127.0.0.1:8000/snippets.api   # Browsable API 접미사
```

요청에 사용하는 포맷 역시 `Content-Type` 헤더로 제어할 수 있습니다.

```bash
# 폼 데이터로 POST
http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
    "id": 3,
    "title": "",
    "code": "print(123)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}

# JSON으로 POST
http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

{
    "id": 4,
    "title": "",
    "code": "print(456)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

위의 `http` 요청에 `--debug` 옵션을 추가하면, 요청 헤더에서 실제 요청 타입을 확인할 수 있습니다.

이제 브라우저를 열고 [<http://127.0.0.1:8000/snippets/>][devserver]로 접속해 보세요.

### Browsability

API는 클라이언트 요청에 따라 응답의 콘텐츠 타입을 결정하므로,  
웹 브라우저에서 접근하면 기본적으로 HTML 형식의 리소스 표현을 반환합니다.

이로 인해 API는 완전히 웹에서 탐색 가능한 HTML 인터페이스를 제공하게 됩니다.

웹에서 탐색 가능한 API는 사용성과 개발 경험을 크게 향상시키며,  
다른 개발자들이 API를 이해하고 활용하는 진입 장벽도 크게 낮춰 줍니다.

자세한 내용과 커스터마이징 방법은 [browsable api][browsable-api] 문서를 참고하세요.

## What's next?

다음 단계인 [tutorial part 3][tut-3]에서는 클래스 기반 view를 사용하고,  
제네릭 view를 통해 작성해야 할 코드 양을 어떻게 줄일 수 있는지 살펴봅니다.

[json-url]: http://example.com/api/items/4.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization.md
[tut-3]: 3-class-based-views.md
