---
source:
    - parsers.py
---

# 파서 (Parsers)

> 머신과 상호작용하는 웹 서비스는 단순한 폼보다 훨씬 복잡한 데이터를 주고받기 때문에,
> form-encoded 방식보다는 더 구조화된 포맷을 사용하는 경향이 있다.
>
> &mdash; Malcom Tredinnick, [Django developers group][cite]

REST framework는 다양한 미디어 타입의 요청을 받을 수 있도록 여러 **내장 Parser 클래스**를 제공한다.  
또한 **커스텀 파서**를 직접 정의할 수도 있어, API가 허용하는 미디어 타입을 자유롭게 설계할 수 있다.

## 파서가 결정되는 방식

각 뷰에서 사용할 수 있는 파서 집합은 항상 **클래스 리스트**로 정의된다.  
`request.data`에 접근하면, REST framework는 들어온 요청의 `Content-Type` 헤더를 확인한 뒤, 해당 콘텐츠를 파싱할 파서를 결정한다.

---

**참고**: 클라이언트 애플리케이션을 개발할 때는 HTTP 요청을 보낼 때 반드시 `Content-Type` 헤더를 설정해야 한다.

Content-Type을 설정하지 않으면, 대부분의 클라이언트는 기본값으로  
`'application/x-www-form-urlencoded'`를 사용하게 되며, 이는 의도와 다를 수 있다.

예를 들어, jQuery의 [.ajax() 메서드][jquery-ajax]를 사용해 `json` 데이터를 전송한다면,  
`contentType: 'application/json'` 설정을 꼭 포함해야 한다.

---

## 파서 설정하기

기본 파서 목록은 전역 설정인 `DEFAULT_PARSER_CLASSES`를 통해 지정할 수 있다.  
예를 들어, 아래 설정은 기본값(JSON + form 데이터) 대신 **JSON 요청만 허용**하도록 한다.

    REST_FRAMEWORK = {
        'DEFAULT_PARSER_CLASSES': [
            'rest_framework.parsers.JSONParser',
        ]
    }

또는 개별 뷰나 뷰셋 단위로 파서를 지정할 수도 있다.  
`APIView` 기반 클래스 뷰를 사용하는 예시는 다음과 같다.

    from rest_framework.parsers import JSONParser
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        """
        JSON 콘텐츠를 포함한 POST 요청을 받을 수 있는 뷰
        """
        parser_classes = [JSONParser]

        def post(self, request, format=None):
            return Response({'received data': request.data})

함수 기반 뷰에서 `@api_view` 데코레이터를 사용하는 경우에는 다음과 같이 설정할 수 있다.

    from rest_framework.decorators import api_view
    from rest_framework.decorators import parser_classes
    from rest_framework.parsers import JSONParser

    @api_view(['POST'])
    @parser_classes([JSONParser])
    def example_view(request, format=None):
        """
        JSON 콘텐츠를 포함한 POST 요청을 받을 수 있는 뷰
        """
        return Response({'received data': request.data})

---

# API 레퍼런스

## JSONParser

`JSON` 요청 콘텐츠를 파싱한다.  
`request.data`는 딕셔너리 형태로 채워진다.

**.media_type**: `application/json`

## FormParser

HTML 폼 콘텐츠를 파싱한다.  
`request.data`는 `QueryDict` 형태로 채워진다.

일반적으로 HTML 폼 데이터를 완전히 지원하려면 `FormParser`와 `MultiPartParser`를 함께 사용하는 것이 좋다.

**.media_type**: `application/x-www-form-urlencoded`

## MultiPartParser

파일 업로드를 지원하는 multipart HTML 폼 콘텐츠를 파싱한다.  
`request.data`는 `QueryDict`, `request.FILES`는 `MultiValueDict`로 채워진다.

HTML 폼 데이터를 완전히 지원하려면 `FormParser`와 `MultiPartParser`를 함께 사용하는 것이 일반적이다.

**.media_type**: `multipart/form-data`

## FileUploadParser

원시(raw) 파일 업로드 요청을 파싱한다.  
`request.data`는 `'file'` 키 하나만 가진 딕셔너리가 되며, 업로드된 파일이 그 값으로 들어간다.

뷰가 `filename` URL 키워드 인자와 함께 호출되면, 해당 값이 파일명으로 사용된다.

`filename` 인자가 없다면, 클라이언트가 반드시 `Content-Disposition` HTTP 헤더에 파일명을 설정해야 한다.  
예: `Content-Disposition: attachment; filename=upload.jpg`

**.media_type**: `*/*`

##### 참고 사항

* `FileUploadParser`는 원시 데이터 형태로 파일을 업로드할 수 있는 **네이티브 클라이언트**를 위한 것이다.  
  웹 기반 업로드나 multipart 업로드를 지원하는 클라이언트라면 `MultiPartParser`를 사용하는 것이 적절하다.
* 이 파서는 모든 콘텐츠 타입과 매칭되므로, 일반적으로 API 뷰에 **단독으로** 설정해야 한다.
* `FileUploadParser`는 Django의 표준 `FILE_UPLOAD_HANDLERS` 설정과 `request.upload_handlers` 속성을 따른다.  
  자세한 내용은 [Django 문서][upload-handlers]를 참고하자.

##### 기본 사용 예시

    # views.py
    class FileUploadView(views.APIView):
        parser_classes = [FileUploadParser]

        def put(self, request, filename, format=None):
            file_obj = request.data['file']
            # ...
            # 업로드된 파일 처리
            # ...
            return Response(status=204)

    # urls.py
    urlpatterns = [
        # ...
        re_path(r'^upload/(?P<filename>[^/]+)$', FileUploadView.as_view())
    ]

---

# 커스텀 파서

커스텀 파서를 구현하려면 `BaseParser`를 상속하고,  
`.media_type` 속성을 설정한 뒤 `.parse(self, stream, media_type, parser_context)` 메서드를 구현하면 된다.

이 메서드는 `request.data`를 채우는 데 사용될 데이터를 반환해야 한다.

`.parse()` 메서드에 전달되는 인자는 다음과 같다.

### stream

요청 본문(body)을 나타내는 stream-like 객체.

### media_type

선택 사항.  
요청 콘텐츠의 미디어 타입이다.

요청의 `Content-Type` 헤더에 따라 파서의 `media_type`보다 더 구체적일 수 있으며,  
예를 들어 `"text/plain; charset=utf-8"` 같은 파라미터를 포함할 수도 있다.

### parser_context

선택 사항.  
요청 콘텐츠를 파싱하는 데 필요한 추가 컨텍스트 정보를 담은 딕셔너리다.

기본적으로 다음 키들을 포함한다: `view`, `request`, `args`, `kwargs`.

## 예시

아래는 요청 본문을 문자열로 읽어 `request.data`에 그대로 담는 **플레인 텍스트 파서** 예시이다.

    class PlainTextParser(BaseParser):
        """
        플레인 텍스트 파서
        """
        media_type = 'text/plain'

        def parse(self, stream, media_type=None, parser_context=None):
            """
            요청 본문을 문자열로 반환
            """
            return stream.read()

---

# 서드파티 패키지

다음과 같은 서드파티 패키지들도 사용할 수 있다.

## YAML

[REST framework YAML][rest-framework-yaml]은 [YAML][yaml] 파싱 및 렌더링을 지원한다.  
과거에는 REST framework에 포함되어 있었지만, 현재는 서드파티 패키지로 분리되었다.

#### 설치 및 설정

pip로 설치한다.

    pip install djangorestframework-yaml

REST framework 설정을 수정한다.

    REST_FRAMEWORK = {
        'DEFAULT_PARSER_CLASSES': [
            'rest_framework_yaml.parsers.YAMLParser',
        ],
        'DEFAULT_RENDERER_CLASSES': [
            'rest_framework_yaml.renderers.YAMLRenderer',
        ],
    }

## XML

[REST Framework XML][rest-framework-xml]은 간단한 비공식 XML 포맷을 제공한다.  
이 역시 과거에는 REST framework에 포함되어 있었으나, 현재는 서드파티 패키지로 제공된다.

#### 설치 및 설정

pip로 설치한다.

    pip install djangorestframework-xml

REST framework 설정을 수정한다.

    REST_FRAMEWORK = {
        'DEFAULT_PARSER_CLASSES': [
            'rest_framework_xml.parsers.XMLParser',
        ],
        'DEFAULT_RENDERER_CLASSES': [
            'rest_framework_xml.renderers.XMLRenderer',
        ],
    }

## MessagePack

[MessagePack][messagepack]은 빠르고 효율적인 바이너리 직렬화 포맷이다.  
[Juan Riaza][juanriaza]가 관리하는 [djangorestframework-msgpack][djangorestframework-msgpack] 패키지를 통해  
REST framework에서 MessagePack 파서와 렌더러를 사용할 수 있다.

## CamelCase JSON

[djangorestframework-camel-case]는 REST framework용 camelCase JSON 파서 및 렌더러를 제공한다.  
이를 통해 serializer에서는 Python 스타일의 snake_case 필드명을 사용하면서,  
API 응답에서는 JavaScript 스타일의 camelCase 필드명을 노출할 수 있다.  
이 패키지는 [Vitaly Babiy][vbabiy]가 관리한다.

[jquery-ajax]: https://api.jquery.com/jQuery.ajax/
[cite]: https://groups.google.com/d/topic/django-developers/dxI4qVzrBY4/discussion
[upload-handlers]: https://docs.djangoproject.com/en/stable/topics/http/file-uploads/#upload-handlers
[rest-framework-yaml]: https://jpadilla.github.io/django-rest-framework-yaml/
[rest-framework-xml]: https://jpadilla.github.io/django-rest-framework-xml/
[yaml]: http://www.yaml.org/
[messagepack]: https://github.com/juanriaza/django-rest-framework-msgpack
[juanriaza]: https://github.com/juanriaza
[vbabiy]: https://github.com/vbabiy
[djangorestframework-msgpack]: https://github.com/juanriaza/django-rest-framework-msgpack
[djangorestframework-camel-case]: https://github.com/vbabiy/djangorestframework-camel-case
