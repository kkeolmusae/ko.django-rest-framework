---
source:
    - decorators.py
    - views.py
---

# 클래스 기반 뷰 (Class-based Views)

> Django의 클래스 기반 뷰는 기존 방식의 뷰에서 벗어난 반가운 변화입니다.
>
> &mdash; [Reinout van Rees][cite]

REST framework는 Django의 `View` 클래스를 상속한 `APIView` 클래스를 제공합니다.

`APIView` 클래스는 일반적인 `View` 클래스와 다음과 같은 차이점이 있습니다.

* 핸들러 메서드에 전달되는 요청은 Django의 `HttpRequest`가 아니라, REST framework의 `Request` 인스턴스입니다.
* 핸들러 메서드는 Django의 `HttpResponse` 대신 REST framework의 `Response`를 반환할 수 있습니다.  
  이 경우 뷰가 콘텐츠 네고시에이션을 관리하고, 응답에 올바른 renderer를 설정합니다.
* 모든 `APIException` 예외는 자동으로 처리되어 적절한 응답으로 변환됩니다.
* 요청이 핸들러 메서드로 전달되기 전에 인증(authentication), 권한(permission), 스로틀(throttle) 검사가 수행됩니다.

`APIView` 클래스의 사용 방식은 일반적인 `View` 클래스와 거의 동일합니다.  
들어오는 요청은 `.get()`, `.post()`와 같은 적절한 핸들러 메서드로 디스패치됩니다.  
또한 클래스에 여러 속성을 설정하여 API 정책의 다양한 측면을 제어할 수 있습니다.

예시:

    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import authentication, permissions
    from django.contrib.auth.models import User

    class ListUsers(APIView):
        """
        시스템의 모든 사용자를 나열하는 뷰.

        * 토큰 인증이 필요합니다.
        * 관리자 사용자만 접근할 수 있습니다.
        """
        authentication_classes = [authentication.TokenAuthentication]
        permission_classes = [permissions.IsAdminUser]

        def get(self, request, format=None):
            """
            모든 사용자 목록을 반환합니다.
            """
            usernames = [user.username for user in User.objects.all()]
            return Response(usernames)

---

**참고**:  
Django REST Framework의 `APIView`, `GenericAPIView`, 다양한 `Mixin`, `ViewSet` 간의  
전체 메서드와 속성, 그리고 이들 간의 관계는 처음 접할 때 다소 복잡하게 느껴질 수 있습니다.

여기 문서 외에도, 각 클래스 기반 뷰의 모든 메서드와 속성을 탐색할 수 있는  
[browsable reference][classy-drf]를 제공하는  
[Classy Django REST Framework][classy-drf] 리소스를 참고하면 도움이 됩니다.

---

## API 정책 속성 (API policy attributes)

다음 속성들은 API 뷰의 플러그인 가능한 동작들을 제어합니다.

### .renderer_classes

### .parser_classes

### .authentication_classes

### .throttle_classes

### .permission_classes

### .content_negotiation_class

## API 정책 인스턴스화 메서드

다음 메서드들은 REST framework가 다양한 플러그인 API 정책을 인스턴스화할 때 사용됩니다.  
일반적으로 이 메서드들을 오버라이드할 필요는 없습니다.

### .get_renderers(self)

### .get_parsers(self)

### .get_authenticators(self)

### .get_throttles(self)

### .get_permissions(self)

### .get_content_negotiator(self)

### .get_exception_handler(self)

## API 정책 구현 메서드

다음 메서드들은 핸들러 메서드로 디스패치되기 전에 호출됩니다.

### .check_permissions(self, request)

### .check_throttles(self, request)

### .perform_content_negotiation(self, request, force=False)

## 디스패치 메서드 (Dispatch methods)

다음 메서드들은 뷰의 `.dispatch()` 메서드에 의해 직접 호출됩니다.  
이들은 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`와 같은  
핸들러 메서드가 호출되기 전후에 필요한 작업을 수행합니다.

### .initial(self, request, *args, **kwargs)

핸들러 메서드가 호출되기 전에 수행되어야 할 작업을 처리합니다.  
이 메서드는 권한 검사, 스로틀링, 콘텐츠 네고시에이션을 수행하는 데 사용됩니다.

일반적으로 이 메서드를 오버라이드할 필요는 없습니다.

### .handle_exception(self, exc)

핸들러 메서드에서 발생한 모든 예외는 이 메서드로 전달되며,  
이 메서드는 `Response` 인스턴스를 반환하거나 예외를 다시 발생시킵니다.

기본 구현은 `rest_framework.exceptions.APIException`의 모든 하위 클래스와  
Django의 `Http404`, `PermissionDenied` 예외를 처리하여  
적절한 오류 응답을 반환합니다.

API가 반환하는 오류 응답을 커스터마이징해야 한다면,  
이 메서드를 상속하여 구현해야 합니다.

### .initialize_request(self, request, *args, **kwargs)

핸들러 메서드에 전달되는 요청 객체가  
일반적인 Django `HttpRequest`가 아니라 `Request` 인스턴스임을 보장합니다.

일반적으로 이 메서드를 오버라이드할 필요는 없습니다.

### .finalize_response(self, request, response, *args, **kwargs)

핸들러 메서드에서 반환된 `Response` 객체가  
콘텐츠 네고시에이션 결과에 따라 올바른 콘텐츠 타입으로 렌더링되도록 보장합니다.

일반적으로 이 메서드를 오버라이드할 필요는 없습니다.

---

# 함수 기반 뷰 (Function Based Views)

> [클래스 기반 뷰가] 항상 더 우월한 해결책이라고 말하는 것은 실수입니다.
>
> &mdash; [Nick Coghlan][cite2]

REST framework는 일반적인 함수 기반 뷰 또한 사용할 수 있도록 지원합니다.  
이를 위해 함수 기반 뷰를 감싸는 간단한 데코레이터 집합을 제공합니다.

이 데코레이터들은 다음을 보장합니다.

* 뷰 함수가 Django의 `HttpRequest` 대신 `Request` 인스턴스를 전달받도록 합니다.
* Django의 `HttpResponse` 대신 `Response`를 반환할 수 있도록 합니다.
* 요청 처리 방식을 설정할 수 있도록 합니다.

## @api_view()

**시그니처:** `@api_view(http_method_names=['GET'])`

이 기능의 핵심은 `api_view` 데코레이터입니다.  
이 데코레이터는 뷰가 응답할 HTTP 메서드 목록을 인자로 받습니다.

예를 들어, 단순히 데이터를 반환하는 매우 간단한 뷰는 다음과 같이 작성할 수 있습니다.

    from rest_framework.decorators import api_view
    from rest_framework.response import Response

    @api_view()
    def hello_world(request):
        return Response({"message": "Hello, world!"})

이 뷰는 [settings]에 정의된 기본 renderer, parser, 인증 클래스 등을 사용합니다.

기본적으로는 `GET` 메서드만 허용되며,  
다른 메서드는 `"405 Method Not Allowed"` 응답을 반환합니다.

이 동작을 변경하려면 허용할 메서드를 다음과 같이 명시하면 됩니다.

    @api_view(['GET', 'POST'])
    def hello_world(request):
        if request.method == 'POST':
            return Response({"message": "Got some data!", "data": request.data})
        return Response({"message": "Hello, world!"})

## API 정책 데코레이터 (API policy decorators)

기본 설정을 오버라이드하려면,  
REST framework에서 제공하는 추가 데코레이터들을 사용할 수 있습니다.  

이 데코레이터들은 반드시 `@api_view` 데코레이터 *아래*에 위치해야 합니다.

예를 들어, 특정 사용자가 하루에 한 번만 호출할 수 있도록  
[스로틀][throttling]을 적용한 뷰는 다음과 같이 작성할 수 있습니다.

    from rest_framework.decorators import api_view, throttle_classes
    from rest_framework.throttling import UserRateThrottle

    class OncePerDayUserThrottle(UserRateThrottle):
        rate = '1/day'

    @api_view(['GET'])
    @throttle_classes([OncePerDayUserThrottle])
    def view(request):
        return Response({"message": "Hello for today! See you tomorrow!"})

이러한 데코레이터들은 앞서 설명한 `APIView` 서브클래스의 속성들과 1:1로 대응됩니다.

사용 가능한 데코레이터 목록은 다음과 같습니다.

* `@renderer_classes(...)`
* `@parser_classes(...)`
* `@authentication_classes(...)`
* `@throttle_classes(...)`
* `@permission_classes(...)`
* `@content_negotiation_class(...)`
* `@metadata_class(...)`
* `@versioning_class(...)`

각 데코레이터는 해당하는 [API 정책 속성][api-policy-attributes]을 설정하는 것과 동일한 효과를 가집니다.

모든 데코레이터는 단일 인자를 받습니다.  
이름이 `_class`로 끝나는 데코레이터는 하나의 클래스를 기대하고,  
`_classes`로 끝나는 데코레이터는 클래스의 리스트 또는 튜플을 기대합니다.

## 뷰 스키마 데코레이터 (View schema decorator)

함수 기반 뷰에 대해 기본 스키마 생성 방식을 오버라이드하려면  
`@schema` 데코레이터를 사용할 수 있습니다.

이 데코레이터는 반드시 `@api_view` 데코레이터 *아래*에 위치해야 합니다.

예시:

    from rest_framework.decorators import api_view, schema
    from rest_framework.schemas import AutoSchema

    class CustomAutoSchema(AutoSchema):
        def get_link(self, path, method, base_url):
            # 여기에서 뷰 인트로스펙션을 오버라이드합니다...

    @api_view(['GET'])
    @schema(CustomAutoSchema())
    def view(request):
        return Response({"message": "Hello for today! See you tomorrow!"})

이 데코레이터는  
[Schema 문서][schemas]에 설명된 `AutoSchema` 인스턴스,  
`AutoSchema` 서브클래스 인스턴스, 또는 `ManualSchema` 인스턴스 하나를 인자로 받습니다.

스키마 생성에서 해당 뷰를 제외하고 싶다면 `None`을 전달할 수 있습니다.

    @api_view(['GET'])
    @schema(None)
    def view(request):
        return Response({"message": "스키마에 표시되지 않습니다!"})

[cite]: https://reinout.vanrees.org/weblog/2011/08/24/class-based-views-usage.html
[cite2]: http://www.boredomandlaziness.org/2012/05/djangos-cbvs-are-not-mistake-but.html
[settings]: settings.md
[throttling]: throttling.md
[schemas]: schemas.md
[classy-drf]: http://www.cdrf.co
[api-policy-attributes]: views.md#api-policy-attributes
