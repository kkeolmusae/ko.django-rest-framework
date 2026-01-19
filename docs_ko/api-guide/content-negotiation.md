---
source:
    - negotiation.py
---

# Content negotiation

> HTTP에는 “콘텐츠 협상(content negotiation)”을 위한 여러 메커니즘이 마련되어 있습니다. 이는 여러 표현(representation)이 가능한 경우, 주어진 응답에 대해 가장 적절한 표현을 선택하는 과정입니다.
>
> &mdash; [RFC 2616][cite], Fielding et al.

[cite]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html

Content negotiation은 클라이언트 또는 서버의 선호도에 따라, 클라이언트에게 반환할 수 있는 여러 표현(representation) 중 하나를 선택하는 과정입니다.

## Determining the accepted renderer

REST framework는 단순한 방식의 content negotiation을 사용하여, 사용 가능한 renderer들, 각 renderer의 우선순위, 그리고 클라이언트의 `Accept:` 헤더를 기반으로 클라이언트에 반환할 미디어 타입을 결정합니다. 이 방식은 **부분적으로는 클라이언트 주도**, **부분적으로는 서버 주도**입니다.

1. 더 구체적인(media type이 더 specific한) 미디어 타입이, 덜 구체적인 미디어 타입보다 우선합니다.
2. 여러 미디어 타입이 동일한 구체성(specificity)을 가진다면, 해당 뷰에 설정된 renderer들의 순서(우선순위)에 따라 결정됩니다.

예를 들어, 다음과 같은 `Accept` 헤더가 주어졌다고 합시다.

    application/json; indent=4, application/json, application/yaml, text/html, */*

각 미디어 타입의 우선순위는 다음과 같습니다.

* `application/json; indent=4`
* `application/json`, `application/yaml` 그리고 `text/html`
* `*/*`

만약 요청된 뷰가 `YAML` 과 `HTML` renderer만 설정되어 있다면, REST framework는 `renderer_classes` 리스트(또는 `DEFAULT_RENDERER_CLASSES` 설정)에서 **더 먼저 나열된 renderer**를 선택합니다.

`HTTP Accept` 헤더에 대한 자세한 정보는 [RFC 2616][accept-header]를 참고하세요.

---

**Note**: REST framework는 선호도를 결정할 때 `"q"` 값을 고려하지 않습니다. `"q"` 값의 사용은 캐싱에 부정적인 영향을 주며, 저자의 의견으로는 content negotiation에 있어 불필요하고 과도하게 복잡한 접근입니다.

이는 HTTP 스펙이 서버 기반 선호도와 클라이언트 기반 선호도를 서버가 어떻게 가중치로 반영해야 하는지에 대해 의도적으로 구체적으로 규정하지(underspecify) 않기 때문에 가능한 합리적인 접근입니다.

---

# Custom content negotiation

대부분의 경우 REST framework에 커스텀 content negotiation 스킴을 제공할 일은 거의 없겠지만, 필요하다면 가능합니다. 커스텀 스킴을 구현하려면 `BaseContentNegotiation` 을 오버라이드하세요.

REST framework의 content negotiation 클래스는 요청에 대해 적절한 parser를 선택하는 것과, 응답에 대해 적절한 renderer를 선택하는 것을 모두 처리하므로, `.select_parser(request, parsers)` 와 `.select_renderer(request, renderers, format_suffix)` 메서드 둘 다를 구현해야 합니다.

`select_parser()` 메서드는 사용 가능한 parser 리스트 중에서 하나의 parser 인스턴스를 반환해야 하며, 들어온 요청을 처리할 수 있는 parser가 없다면 `None` 을 반환해야 합니다.

`select_renderer()` 메서드는 `(renderer instance, media type)` 형태의 2-튜플을 반환하거나, `NotAcceptable` 예외를 발생시켜야 합니다.

## Example

다음은 parser나 renderer를 선택할 때 클라이언트 요청을 무시하는 커스텀 content negotiation 클래스의 예시입니다.

    from rest_framework.negotiation import BaseContentNegotiation

    class IgnoreClientContentNegotiation(BaseContentNegotiation):
        def select_parser(self, request, parsers):
            """
            `.parser_classes` 리스트의 첫 번째 parser를 선택한다.
            """
            return parsers[0]

        def select_renderer(self, request, renderers, format_suffix):
            """
            `.renderer_classes` 리스트의 첫 번째 renderer를 선택한다.
            """
            return (renderers[0], renderers[0].media_type)

## Setting the content negotiation

기본 content negotiation 클래스는 `DEFAULT_CONTENT_NEGOTIATION_CLASS` 설정으로 전역 지정할 수 있습니다. 예를 들어, 아래 설정은 위에서 예시로 든 `IgnoreClientContentNegotiation` 클래스를 사용합니다.

    REST_FRAMEWORK = {
        'DEFAULT_CONTENT_NEGOTIATION_CLASS': 'myapp.negotiation.IgnoreClientContentNegotiation',
    }

또한 `APIView` 기반 클래스 뷰를 사용하여 개별 뷰(또는 뷰셋) 단위로 content negotiation을 설정할 수도 있습니다.

    from myapp.negotiation import IgnoreClientContentNegotiation
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class NoNegotiationView(APIView):
        """
        content negotiation을 수행하지 않는 예시 뷰.
        """
        content_negotiation_class = IgnoreClientContentNegotiation

        def get(self, request, format=None):
            return Response({
                'accepted media type': request.accepted_renderer.media_type
            })

[accept-header]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
