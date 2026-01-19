---
source:
    - metadata.py
---

# Metadata

> [`OPTIONS`] 메서드는 리소스에 대한 동작을 암시하거나 리소스 조회를 시작하지 않고도,  
> 클라이언트가 해당 리소스와 연관된 옵션이나 요구사항, 또는 서버의 기능을 파악할 수 있게 해줍니다.
>
> &mdash; [RFC7231, Section 4.3.7.][cite]

REST framework는 `OPTIONS` 요청에 대해 API가 어떻게 응답할지를 결정할 수 있는 **구성 가능한 메커니즘**을 제공합니다. 이를 통해 API 스키마나 기타 리소스 정보를 반환할 수 있습니다.

현재 HTTP `OPTIONS` 요청에 대해 **어떤 형태의 응답을 반환해야 하는지에 대한 널리 채택된 표준은 존재하지 않기 때문에**, REST framework에서는 유용한 정보를 반환하는 **임의(ad-hoc) 스타일**을 제공합니다.

다음은 기본적으로 반환되는 정보의 예시 응답입니다.

    HTTP 200 OK
    Allow: GET, POST, HEAD, OPTIONS
    Content-Type: application/json

    {
        "name": "To Do List",
        "description": "List existing 'To Do' items, or create a new item.",
        "renders": [
            "application/json",
            "text/html"
        ],
        "parses": [
            "application/json",
            "application/x-www-form-urlencoded",
            "multipart/form-data"
        ],
        "actions": {
            "POST": {
                "note": {
                    "type": "string",
                    "required": false,
                    "read_only": false,
                    "label": "title",
                    "max_length": 100
                }
            }
        }
    }

## Setting the metadata scheme

메타데이터 클래스는 `'DEFAULT_METADATA_CLASS'` 설정 키를 사용해 전역으로 지정할 수 있습니다.

    REST_FRAMEWORK = {
        'DEFAULT_METADATA_CLASS': 'rest_framework.metadata.SimpleMetadata'
    }

또는 개별 뷰 단위로 메타데이터 클래스를 지정할 수도 있습니다.

    class APIRoot(APIView):
        metadata_class = APIRootMetadata

        def get(self, request, format=None):
            return Response({
                ...
            })

REST framework 패키지에는 `SimpleMetadata` 라는 **단 하나의 메타데이터 클래스 구현**만 포함되어 있습니다.  
다른 스타일을 사용하고 싶다면, **커스텀 메타데이터 클래스**를 직접 구현해야 합니다.

## Creating schema endpoints

일반적인 `GET` 요청으로 접근 가능한 **스키마 엔드포인트**를 만들고 싶다면, 메타데이터 API를 재사용하는 방법을 고려할 수 있습니다.

예를 들어, 다음과 같은 추가 라우트를 viewset에 정의하여 링크 가능한 스키마 엔드포인트를 제공할 수 있습니다.

    @action(methods=['GET'], detail=False)
    def api_schema(self, request):
        meta = self.metadata_class()
        data = meta.determine_metadata(request, self)
        return Response(data)

이 방식을 선택할 수 있는 이유는 여러 가지가 있는데, 그중 하나는 `OPTIONS` 응답이 [캐시되지 않기 때문][no-options]입니다.

---

# Custom metadata classes

커스텀 메타데이터 클래스를 제공하고 싶다면 `BaseMetadata` 를 상속하고  
`determine_metadata(self, request, view)` 메서드를 구현해야 합니다.

이를 통해 다음과 같은 작업을 할 수 있습니다.

- [JSON Schema][json-schema] 와 같은 포맷으로 스키마 정보를 반환
- 관리자(admin) 사용자에게만 디버그 정보 반환

## Example

다음 클래스는 `OPTIONS` 요청에 대해 반환되는 정보를 제한하는 예시입니다.

    class MinimalMetadata(BaseMetadata):
        """
        `OPTIONS` 요청에 대해 필드 및 기타 정보는 포함하지 않고,
        name과 description만 반환한다.
        """
        def determine_metadata(self, request, view):
            return {
                'name': view.get_view_name(),
                'description': view.get_view_description()
            }

이후 설정에서 해당 커스텀 클래스를 사용하도록 구성합니다.

    REST_FRAMEWORK = {
        'DEFAULT_METADATA_CLASS': 'myproject.apps.core.MinimalMetadata'
    }

# Third party packages

다음은 추가적인 메타데이터 구현을 제공하는 서드파티 패키지들입니다.

## DRF-schema-adapter

[drf-schema-adapter][drf-schema-adapter] 는 프론트엔드 프레임워크나 라이브러리에 스키마 정보를 제공하는 작업을 쉽게 만들어주는 도구 모음입니다.  
메타데이터 믹인(mixin), 두 개의 메타데이터 클래스, 그리고 [json-schema][json-schema] 및 다양한 라이브러리에서 읽을 수 있는 스키마 정보를 생성하기 위한 여러 어댑터를 제공합니다.

또한 특정 프론트엔드에 맞게 **자체 어댑터를 작성**할 수도 있으며,  
스키마 정보를 JSON 파일로 내보낼 수 있는 exporter도 제공합니다.

[cite]: https://tools.ietf.org/html/rfc7231#section-4.3.7
[no-options]: https://www.mnot.net/blog/2012/10/29/NO_OPTIONS
[json-schema]: https://json-schema.org/
[drf-schema-adapter]: https://github.com/drf-forms/drf-schema-adapter
