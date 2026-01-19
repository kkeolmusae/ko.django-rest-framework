---
source:
    - versioning.py
---

# Versioning

> 인터페이스에 버전을 붙인다는 건, 배포된 클라이언트를 “정중하게” 죽이는 방법일 뿐이다.
>
> &mdash; [Roy Fielding][cite].

API 버저닝은 서로 다른 클라이언트 간에 동작을 변경할 수 있게 해줍니다. REST framework는 여러 가지 버저닝 스킴을 제공합니다.

버저닝은 들어오는 클라이언트 요청에 의해 결정되며, 요청 URL 또는 요청 헤더를 기반으로 할 수 있습니다.

버저닝에 접근하는 방법은 여러 가지가 있습니다. 특히, 여러분이 통제할 수 없는 외부의 여러 클라이언트와 함께 매우 장기적으로 운영해야 하는 시스템을 설계하는 경우에는, [버전이 없는(non-versioned) 시스템도 적절할 수 있습니다][roy-fielding-on-versioning].

## Versioning with REST framework

API 버저닝이 활성화되어 있으면 `request.version` 속성에, 들어온 요청이 요구한 버전에 해당하는 문자열이 들어갑니다.

기본적으로는 버저닝이 비활성화되어 있으며, 이 경우 `request.version` 은 항상 `None` 을 반환합니다.

#### Varying behavior based on the version

API 동작을 어떻게 버전에 따라 바꿀지는 전적으로 여러분에게 달려 있습니다. 다만 흔한 예로는, 새로운 버전에서 다른 직렬화(serialize) 방식을 사용하도록 바꾸는 경우가 있습니다. 예:

    def get_serializer_class(self):
        if self.request.version == 'v1':
            return AccountSerializerVersion1
        return AccountSerializer

#### Reversing URLs for versioned APIs

REST framework가 포함한 `reverse` 함수는 버저닝 스킴과 연동됩니다. 아래처럼 현재 `request` 를 키워드 인자로 반드시 넘겨야 합니다.

    from rest_framework.reverse import reverse

    reverse('bookings-list', request=request)

위 함수는 요청 버전에 맞게 필요한 URL 변환을 적용합니다. 예:

* `NamespaceVersioning` 을 사용 중이고 API 버전이 `'v1'` 이라면, URL lookup은 `'v1:bookings-list'` 를 사용하게 되며, 이는 `http://example.org/v1/bookings/` 같은 URL로 resolve될 수 있습니다.
* `QueryParameterVersioning` 을 사용 중이고 API 버전이 `1.0` 이라면, 반환되는 URL은 `http://example.org/bookings/?version=1.0` 같은 형태가 될 수 있습니다.

#### Versioned APIs and hyperlinked serializers

URL 기반 버저닝 스킴과 함께 하이퍼링크 직렬화 스타일(hyperlinked serialization styles)을 사용할 때는, serializer에 request를 context로 포함시키도록 주의해야 합니다.

    def get(self, request):
        queryset = Booking.objects.all()
        serializer = BookingsSerializer(queryset, many=True, context={'request': request})
        return Response({'all_bookings': serializer.data})

이렇게 하면 반환되는 URL에 올바른 버전 정보가 포함될 수 있습니다.

## Configuring the versioning scheme

버저닝 스킴은 `DEFAULT_VERSIONING_CLASS` 설정 키로 정의합니다.

    REST_FRAMEWORK = {
        'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning'
    }

명시적으로 설정하지 않으면 `DEFAULT_VERSIONING_CLASS` 값은 `None` 이며, 이 경우 `request.version` 은 항상 `None` 을 반환합니다.

개별 뷰에서 버저닝 스킴을 지정할 수도 있습니다. 보통은 전역으로 하나의 스킴을 사용하는 편이 더 합리적이므로 필요하지 않은 경우가 많습니다. 그래도 필요하다면 `versioning_class` 속성을 사용하세요.

    class ProfileList(APIView):
        versioning_class = versioning.QueryParameterVersioning

#### Other versioning settings

다음 설정 키들도 버저닝 제어에 사용됩니다.

* `DEFAULT_VERSION`. 버저닝 정보가 없을 때 `request.version` 에 사용할 값입니다. 기본값은 `None`.
* `ALLOWED_VERSIONS`. 설정하면 버저닝 스킴이 반환할 수 있는 버전 집합을 제한하며, 제공된 버전이 이 집합에 없으면 오류를 발생시킵니다. `DEFAULT_VERSION` 값은 (`None` 이 아닌 한) 항상 `ALLOWED_VERSIONS` 집합의 일부로 간주됩니다. 기본값은 `None`.
* `VERSION_PARAM`. 미디어 타입 또는 URL query parameter 등에서 사용할 버전 파라미터 이름(문자열)입니다. 기본값은 `'version'`.

또한, 커스텀 버저닝 스킴을 정의하여 `default_version`, `allowed_versions`, `version_param` 클래스 변수를 사용하면, 버저닝 클래스와 위 3개 값을 뷰/뷰셋 단위로 설정할 수도 있습니다. 예를 들어 `URLPathVersioning` 을 사용하고 싶다면:

    from rest_framework.versioning import URLPathVersioning
    from rest_framework.views import APIView

    class ExampleVersioning(URLPathVersioning):
        default_version = ...
        allowed_versions = ...
        version_param = ...

    class ExampleView(APIVIew):
        versioning_class = ExampleVersioning

---

# API Reference

## AcceptHeaderVersioning

이 스킴은 클라이언트가 `Accept` 헤더의 미디어 타입 일부로 버전을 지정하도록 요구합니다. 버전은 메인 미디어 타입을 보완하는 미디어 타입 파라미터로 포함됩니다.

아래는 accept header 버저닝 스타일을 사용하는 HTTP 요청 예시입니다.

    GET /bookings/ HTTP/1.1
    Host: example.com
    Accept: application/json; version=1.0

위 요청에서는 `request.version` 속성이 문자열 `'1.0'` 을 반환합니다.

Accept 헤더 기반 버저닝은 [일반적으로][klabnik-guidelines] [베스트 프랙티스][heroku-guidelines]로 여겨지지만, 클라이언트 요구사항에 따라 다른 스타일이 더 적합할 수도 있습니다.

#### Using accept headers with vendor media types

엄밀히 말하면 `json` 미디어 타입은 [추가 파라미터를 포함하는 것으로 명시되어 있지 않습니다][json-parameters]. 잘 정의된(public) API를 만든다면 [vendor media type][vendor-media-type] 사용을 고려할 수 있습니다. 이를 위해 JSON 기반 renderer를 커스텀 미디어 타입으로 설정하도록 renderer를 구성합니다.

    class BookingsAPIRenderer(JSONRenderer):
        media_type = 'application/vnd.megacorp.bookings+json'

그러면 클라이언트 요청은 다음처럼 됩니다.

    GET /bookings/ HTTP/1.1
    Host: example.com
    Accept: application/vnd.megacorp.bookings+json; version=1.0

## URLPathVersioning

이 스킴은 클라이언트가 URL 경로(path)의 일부로 버전을 지정하도록 요구합니다.

    GET /v1/bookings/ HTTP/1.1
    Host: example.com
    Accept: application/json

URL conf에는 `'version'` 키워드 인자로 버전을 매칭하는 패턴이 포함되어야 하며, 그래야 이 정보가 버저닝 스킴에서 사용 가능합니다.

    urlpatterns = [
        re_path(
            r'^(?P<version>(v1|v2))/bookings/$',
            bookings_list,
            name='bookings-list'
        ),
        re_path(
            r'^(?P<version>(v1|v2))/bookings/(?P<pk>[0-9]+)/$',
            bookings_detail,
            name='bookings-detail'
        )
    ]

## NamespaceVersioning

클라이언트 입장에서는 이 스킴은 `URLPathVersioning` 과 동일합니다. 차이는 Django 애플리케이션에서의 설정 방식인데, URL keyword argument 대신 URL namespacing을 사용합니다.

    GET /v1/something/ HTTP/1.1
    Host: example.com
    Accept: application/json

이 스킴에서는 `request.version` 이 들어오는 요청 경로와 매칭되는 `namespace` 를 기준으로 결정됩니다.

아래 예제에서는 같은 뷰 세트에 대해 서로 다른 namespace 아래 두 가지 URL prefix를 제공합니다.

    # bookings/urls.py
    urlpatterns = [
        re_path(r'^$', bookings_list, name='bookings-list'),
        re_path(r'^(?P<pk>[0-9]+)/$', bookings_detail, name='bookings-detail')
    ]

    # urls.py
    urlpatterns = [
        re_path(r'^v1/bookings/', include('bookings.urls', namespace='v1')),
        re_path(r'^v2/bookings/', include('bookings.urls', namespace='v2'))
    ]

`URLPathVersioning` 과 `NamespaceVersioning` 은 단순한 버저닝 스킴이 필요할 때 둘 다 합리적인 선택입니다. `URLPathVersioning` 은 소규모 애드혹 프로젝트에 더 적합할 수 있고, `NamespaceVersioning` 은 큰 프로젝트에서 관리하기 더 쉬운 편입니다.

## HostNameVersioning

호스트네임 버저닝 스킴은 클라이언트가 URL의 호스트네임 일부로 요청 버전을 지정하도록 요구합니다.

예를 들어 다음은 `http://v1.example.com/bookings/` 로 보내는 HTTP 요청입니다.

    GET /bookings/ HTTP/1.1
    Host: v1.example.com
    Accept: application/json

기본 구현은 호스트네임이 다음 단순 정규식을 만족하기를 기대합니다.

    ^([a-zA-Z0-9]+)\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+$

첫 번째 그룹이 괄호로 묶여 있는데, 이는 호스트네임에서 매칭되는 부분을 나타낸다는 점에 유의하세요.

`HostNameVersioning` 은 보통 디버그 모드에서 사용하기 불편할 수 있는데, `127.0.0.1` 같은 raw IP로 접근하는 경우가 많기 때문입니다. 이 경우 도움이 될 수 있는 [localhost를 커스텀 서브도메인으로 접근하는 방법][lvh]에 대한 온라인 튜토리얼들이 있습니다.

호스트네임 기반 버저닝은 버전에 따라 서로 다른 서버로 라우팅해야 하는 요구사항이 있을 때 특히 유용할 수 있습니다. API 버전별로 서로 다른 DNS 레코드를 구성할 수 있기 때문입니다.

## QueryParameterVersioning

이 스킴은 URL의 query parameter로 버전을 포함하는 단순한 스타일입니다. 예:

    GET /something/?version=0.1 HTTP/1.1
    Host: example.com
    Accept: application/json

---

# Custom versioning schemes

커스텀 버저닝 스킴을 구현하려면 `BaseVersioning` 을 상속하고 `.determine_version` 메서드를 오버라이드합니다.

## Example

다음 예제는 커스텀 `X-API-Version` 헤더를 사용해 요청된 버전을 결정합니다.

    class XAPIVersionScheme(versioning.BaseVersioning):
        def determine_version(self, request, *args, **kwargs):
            return request.META.get('HTTP_X_API_VERSION', None)

버저닝 스킴이 요청 URL 기반이라면, 버전이 포함된 URL을 어떻게 reverse할지도 바꿔야 합니다. 이를 위해 클래스의 `.reverse()` 메서드를 오버라이드하면 됩니다. 예시는 소스 코드를 참고하세요.

[cite]: https://www.slideshare.net/evolve_conference/201308-fielding-evolve/31
[roy-fielding-on-versioning]: https://www.infoq.com/articles/roy-fielding-on-versioning
[klabnik-guidelines]: http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned
[heroku-guidelines]: https://github.com/interagent/http-api-design/blob/master/en/foundations/require-versioning-in-the-accepts-header.md
[json-parameters]: https://tools.ietf.org/html/rfc4627#section-6
[vendor-media-type]: https://en.wikipedia.org/wiki/Internet_media_type#Vendor_tree
[lvh]: https://reinteractive.net/posts/199-developing-and-testing-rails-applications-with-subdomains
