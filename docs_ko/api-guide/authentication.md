---
source:
    - authentication.py
---

# 인증 (Authentication)

> 인증(Auth)은 플러그인 방식으로 교체 가능해야 한다.
>
> &mdash; Jacob Kaplan-Moss, ["REST worst practices"][cite]

인증(Authentication)이란, 들어오는 요청을 요청의 발신 사용자나 서명에 사용된 토큰 같은 식별 자격 증명(credentials) 집합과 연결하는 메커니즘입니다. 이후 [permission] 및 [throttling] 정책은 이러한 자격 증명을 사용해 요청을 허용할지 여부를 판단할 수 있습니다.

REST framework는 기본 제공되는 여러 인증 방식을 제공하며, 커스텀 방식도 구현할 수 있습니다.

인증은 항상 뷰의 가장 시작 지점에서 실행되며, permission 및 throttling 체크가 수행되기 전이고, 다른 어떤 코드도 진행되기 전에 실행됩니다.

`request.user` 속성은 보통 `contrib.auth` 패키지의 `User` 클래스 인스턴스로 설정됩니다.

`request.auth` 속성은 추가적인 인증 정보를 위해 사용됩니다. 예를 들어, 요청이 서명되었을 때 사용된 인증 토큰을 나타내는 데 사용될 수 있습니다.

---

**참고:** **인증 자체만으로는 들어오는 요청을 허용하거나 거부하지 않습니다.** 인증은 단지 요청이 어떤 자격 증명으로 이루어졌는지를 식별할 뿐입니다.

API의 permission 정책을 설정하는 방법은 [permissions 문서][permission]를 참고하세요.

---

## 인증이 결정되는 방식 (How authentication is determined)

인증 방식은 항상 클래스들의 리스트로 정의됩니다. REST framework는 리스트에 있는 각 클래스에 대해 인증을 시도하며, 최초로 인증에 성공한 클래스의 반환값을 사용해 `request.user`와 `request.auth`를 설정합니다.

어떤 클래스도 인증에 성공하지 못하면, `request.user`는 `django.contrib.auth.models.AnonymousUser` 인스턴스로 설정되고, `request.auth`는 `None`으로 설정됩니다.

비인증 요청에 대한 `request.user`와 `request.auth` 값은 `UNAUTHENTICATED_USER` 및 `UNAUTHENTICATED_TOKEN` 설정을 통해 변경할 수 있습니다.

## 인증 방식 설정하기 (Setting the authentication scheme)

기본 인증 방식은 `DEFAULT_AUTHENTICATION_CLASSES` 설정을 통해 전역으로 지정할 수 있습니다. 예를 들어:

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'rest_framework.authentication.BasicAuthentication',
            'rest_framework.authentication.SessionAuthentication',
        ]
    }

또는 `APIView` 기반의 클래스 기반 뷰를 사용해, 뷰(또는 뷰셋) 단위로 인증 방식을 설정할 수도 있습니다.

    from rest_framework.authentication import SessionAuthentication, BasicAuthentication
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        authentication_classes = [SessionAuthentication, BasicAuthentication]
        permission_classes = [IsAuthenticated]

        def get(self, request, format=None):
            content = {
                'user': str(request.user),  # `django.contrib.auth.User` 인스턴스.
                'auth': str(request.auth),  # None
            }
            return Response(content)

또는 함수 기반 뷰에서 `@api_view` 데코레이터를 사용하는 경우:

    @api_view(['GET'])
    @authentication_classes([SessionAuthentication, BasicAuthentication])
    @permission_classes([IsAuthenticated])
    def example_view(request, format=None):
        content = {
            'user': str(request.user),  # `django.contrib.auth.User` 인스턴스.
            'auth': str(request.auth),  # None
        }
        return Response(content)

## Unauthorized 및 Forbidden 응답 (Unauthorized and Forbidden responses)

비인증 요청이 permission에 의해 거부될 때, 적절한 에러 코드는 두 가지가 있을 수 있습니다.

* [HTTP 401 Unauthorized][http401]
* [HTTP 403 Permission Denied][http403]

HTTP 401 응답에는 항상 `WWW-Authenticate` 헤더가 포함되어야 하며, 이 헤더는 클라이언트에게 인증 방법을 안내합니다. HTTP 403 응답에는 `WWW-Authenticate` 헤더가 포함되지 않습니다.

어떤 응답이 사용될지는 인증 방식에 따라 달라집니다. 여러 인증 방식이 함께 사용될 수 있지만, 응답 타입(401/403)을 결정하는 데는 오직 하나의 방식만 사용됩니다. **응답 타입을 결정할 때는 뷰에 설정된 첫 번째 인증 클래스가 사용됩니다.**

또한 요청이 인증에는 성공할 수 있지만, 그럼에도 요청 수행 권한이 거부되는 경우가 있을 수 있습니다. 이 경우에는 인증 방식과 무관하게 항상 `403 Permission Denied` 응답이 사용됩니다.

## Django 5.1+ `LoginRequiredMiddleware`

Django 5.1+을 사용하고 [`LoginRequiredMiddleware`][login-required-middleware]를 사용하는 경우, DRF의 모든 뷰는 이 미들웨어의 적용 대상에서 제외(opt-out)된다는 점에 유의하세요. 이는 DRF의 인증이 미들웨어 적용 이후에 결정될 수 있는 인증/권한 클래스 기반이기 때문입니다. 또한 요청이 인증되지 않았을 때, 해당 미들웨어는 사용자를 로그인 페이지로 리다이렉트하는데, 이는 API 요청에는 적절하지 않으며 보통 401 상태 코드를 반환하는 편이 더 낫습니다.

REST framework는 전역 설정인 `DEFAULT_AUTHENTICATION_CLASSES` 및 `DEFAULT_PERMISSION_CLASSES`를 통해 DRF 뷰에 대해 동등한 메커니즘을 제공합니다. API 요청에 대해 “로그인되어 있어야 함”을 강제하려면 이에 맞게 설정을 변경해야 합니다.

## Apache mod_wsgi 전용 설정 (Apache mod_wsgi specific configuration)

[Apache에서 mod_wsgi로 배포][mod_wsgi_official]하는 경우, 기본 설정에서는 Authorization 헤더가 WSGI 애플리케이션으로 전달되지 않습니다. 이는 Apache가 애플리케이션 레벨이 아닌 서버 레벨에서 인증을 처리할 것이라고 가정하기 때문입니다.

Apache에 배포하면서 세션 기반이 아닌 인증을 사용한다면, 필요한 헤더가 애플리케이션으로 전달되도록 mod_wsgi를 명시적으로 설정해야 합니다. 이는 적절한 컨텍스트에 `WSGIPassAuthorization` 지시어를 추가하고 값을 `'On'`으로 설정함으로써 가능합니다.

    # 이 설정은 서버 설정, virtual host, directory 또는 .htaccess 어디에든 둘 수 있습니다.
    WSGIPassAuthorization On

---

# API 레퍼런스 (API Reference)

## BasicAuthentication

이 인증 방식은 사용자의 username과 password로 서명된 [HTTP Basic Authentication][basicauth]을 사용합니다. Basic 인증은 일반적으로 테스트에만 적합합니다.

인증에 성공하면 `BasicAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스입니다.
* `request.auth`는 `None`입니다.

비인증 상태에서 permission에 의해 거부되는 응답은 적절한 WWW-Authenticate 헤더와 함께 `HTTP 401 Unauthorized`로 반환됩니다. 예:

    WWW-Authenticate: Basic realm="api"

**참고:** 프로덕션에서 `BasicAuthentication`을 사용한다면, API가 반드시 `https`로만 제공되도록 보장해야 합니다. 또한 API 클라이언트가 로그인 시마다 username/password를 다시 요청하도록 하고, 이 정보를 영구 저장소에 절대로 저장하지 않도록 해야 합니다.

## TokenAuthentication

---

**참고:** Django REST framework가 제공하는 토큰 인증은 비교적 단순한 구현입니다.

사용자당 여러 토큰을 허용하고, 더 강한 보안 구현 세부사항 및 토큰 만료를 지원하는 구현이 필요하다면, 서드파티 패키지인 [Django REST Knox][django-rest-knox]를 참고하세요.

---

이 인증 방식은 간단한 토큰 기반 HTTP 인증을 사용합니다. 토큰 인증은 네이티브 데스크톱/모바일 클라이언트 같은 클라이언트-서버 구성에 적합합니다.

`TokenAuthentication`을 사용하려면, [인증 클래스 설정](#setting-the-authentication-scheme)에 `TokenAuthentication`을 포함시키고, `INSTALLED_APPS`에 `rest_framework.authtoken`을 추가해야 합니다.

    INSTALLED_APPS = [
        ...
        'rest_framework.authtoken'
    ]

설정을 변경한 후 `manage.py migrate`를 실행해야 합니다.

`rest_framework.authtoken` 앱은 Django 데이터베이스 마이그레이션을 제공합니다.

또한 사용자에 대한 토큰을 생성해야 합니다.

    from rest_framework.authtoken.models import Token

    token = Token.objects.create(user=...)
    print(token.key)

클라이언트가 인증하려면 토큰 키를 `Authorization` HTTP 헤더에 포함해야 합니다. 키 앞에는 문자열 리터럴 `"Token"`을 붙이고, 두 문자열 사이에는 공백으로 구분합니다. 예:

    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b

*`Bearer`처럼 다른 키워드를 헤더에서 사용하고 싶다면, `TokenAuthentication`을 서브클래싱하고 `keyword` 클래스 변수를 설정하면 됩니다.*

인증에 성공하면 `TokenAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스입니다.
* `request.auth`는 `rest_framework.authtoken.models.Token` 인스턴스입니다.

비인증 상태에서 permission에 의해 거부되는 응답은 적절한 WWW-Authenticate 헤더와 함께 `HTTP 401 Unauthorized`로 반환됩니다. 예:

    WWW-Authenticate: Token

`curl` 커맨드라인 도구는 토큰 인증 API를 테스트하는 데 유용할 수 있습니다. 예:

    curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'

---

**참고:** 프로덕션에서 `TokenAuthentication`을 사용한다면, API가 반드시 `https`로만 제공되도록 보장해야 합니다.

---

### 토큰 생성하기 (Generating Tokens)

#### 시그널 사용 (By using signals)

모든 사용자에게 토큰이 자동 생성되게 하고 싶다면, User의 `post_save` 시그널을 처리하면 됩니다.

    from django.conf import settings
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from rest_framework.authtoken.models import Token

    @receiver(post_save, sender=settings.AUTH_USER_MODEL)
    def create_auth_token(sender, instance=None, created=False, **kwargs):
        if created:
            Token.objects.create(user=instance)

이 코드는 설치된 `models.py` 모듈이나, Django 시작 시 import되는 다른 위치에 두어야 합니다.

이미 생성된 사용자들이 있다면, 다음과 같이 기존 사용자 전체에 대해 토큰을 생성할 수 있습니다.

    from django.contrib.auth.models import User
    from rest_framework.authtoken.models import Token

    for user in User.objects.all():
        Token.objects.get_or_create(user=user)

#### API 엔드포인트 노출 (By exposing an api endpoint)

`TokenAuthentication`을 사용할 때, username/password로 토큰을 발급받을 수 있는 메커니즘을 제공하고 싶을 수 있습니다. REST framework는 이를 위한 내장 뷰를 제공합니다. 사용하려면 URLconf에 `obtain_auth_token` 뷰를 추가하세요.

    from rest_framework.authtoken import views
    urlpatterns += [
        path('api-token-auth/', views.obtain_auth_token)
    ]

URL 패턴의 경로는 원하는 대로 정할 수 있습니다.

`obtain_auth_token` 뷰는 `username`과 `password` 필드를 폼 데이터 또는 JSON으로 POST했을 때, 유효하면 JSON 응답을 반환합니다.

    { 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }

기본 `obtain_auth_token` 뷰는 설정의 기본 renderer/parser를 사용하는 대신, JSON 요청/응답을 명시적으로 사용합니다.

기본적으로 `obtain_auth_token` 뷰에는 permission이나 throttling이 적용되어 있지 않습니다. throttling을 적용하려면 뷰 클래스를 오버라이드하고 `throttle_classes` 속성을 포함해야 합니다.

`obtain_auth_token`의 커스텀 버전이 필요하다면 `ObtainAuthToken` 뷰 클래스를 서브클래싱하고, URLconf에서 그 클래스를 사용하면 됩니다.

예를 들어 `token` 외에 추가 사용자 정보를 반환할 수도 있습니다.

    from rest_framework.authtoken.views import ObtainAuthToken
    from rest_framework.authtoken.models import Token
    from rest_framework.response import Response

    class CustomAuthToken(ObtainAuthToken):

        def post(self, request, *args, **kwargs):
            serializer = self.serializer_class(data=request.data,
                                               context={'request': request})
            serializer.is_valid(raise_exception=True)
            user = serializer.validated_data['user']
            token, created = Token.objects.get_or_create(user=user)
            return Response({
                'token': token.key,
                'user_id': user.pk,
                'email': user.email
            })

그리고 `urls.py`에서:

    urlpatterns += [
        path('api-token-auth/', CustomAuthToken.as_view())
    ]

#### Django admin 사용 (With Django admin)

admin 인터페이스를 통해 수동으로 토큰을 만들 수도 있습니다. 사용자 수가 매우 큰 경우, 필요에 맞게 `TokenAdmin` 클래스를 몽키패치하는 것을 권장합니다. 특히 `user` 필드를 `raw_field`로 선언하여 커스터마이징할 수 있습니다.

`your_app/admin.py`:

    from rest_framework.authtoken.admin import TokenAdmin

    TokenAdmin.raw_id_fields = ['user']

#### Django manage.py 커맨드 사용 (Using Django manage.py command)

3.6.4 버전부터는 다음 커맨드로 사용자 토큰을 생성할 수 있습니다.

    ./manage.py drf_create_token <username>

이 커맨드는 주어진 사용자의 API 토큰을 반환하며, 토큰이 없으면 생성합니다.

    Generated token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b for user user1

토큰을 재발급(예: 유출/탈취된 경우)하고 싶다면 추가 파라미터를 전달하면 됩니다.

    ./manage.py drf_create_token -r <username>

## SessionAuthentication

이 인증 방식은 Django 기본 세션 백엔드를 사용합니다. Session 인증은 웹사이트와 동일한 세션 컨텍스트에서 동작하는 AJAX 클라이언트에 적합합니다.

인증에 성공하면 `SessionAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스입니다.
* `request.auth`는 `None`입니다.

비인증 상태에서 permission에 의해 거부되는 응답은 `HTTP 403 Forbidden`으로 반환됩니다.

AJAX 스타일 API에서 SessionAuthentication을 사용한다면, `PUT`, `PATCH`, `POST`, `DELETE` 같은 “unsafe” HTTP 메서드 호출에는 유효한 CSRF 토큰을 포함해야 합니다. 자세한 내용은 [Django CSRF 문서][csrf-ajax]를 참고하세요.

!!! warning
    로그인 페이지를 만들 때는 항상 Django의 표준 로그인 뷰를 사용하세요. 그래야 로그인 뷰가 올바르게 보호됩니다.

REST framework의 CSRF 검증은 동일한 뷰에서 세션 기반/비세션 기반 인증을 모두 지원해야 하기 때문에, 표준 Django와는 약간 다르게 동작합니다. 즉, 인증된 요청만 CSRF 토큰이 필요하고, 익명 요청은 CSRF 토큰 없이도 전송될 수 있습니다. 이 동작은 로그인 뷰에는 적절하지 않으며, 로그인 뷰에는 항상 CSRF 검증이 적용되어야 합니다.

## RemoteUserAuthentication

이 인증 방식은 웹 서버가 `REMOTE_USER` 환경 변수를 설정하도록 하여, 인증을 웹 서버에 위임할 수 있게 합니다.

이를 사용하려면 `AUTHENTICATION_BACKENDS` 설정에 `django.contrib.auth.backends.RemoteUserBackend`(또는 서브클래스)를 포함해야 합니다. 기본적으로 `RemoteUserBackend`는 존재하지 않는 username에 대해 `User` 객체를 생성합니다. 이를 포함한 다른 동작을 변경하려면 [Django 문서](https://docs.djangoproject.com/en/stable/howto/auth-remote-user/)를 참고하세요.

인증에 성공하면 `RemoteUserAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스입니다.
* `request.auth`는 `None`입니다.

인증 방식 구성에 대한 정보는 웹 서버 문서를 참고하세요. 예:

* [Apache Authentication How-To](https://httpd.apache.org/docs/2.4/howto/auth.html)
* [NGINX (Restricting Access)](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/)

# 커스텀 인증 (Custom authentication)

커스텀 인증 방식을 구현하려면 `BaseAuthentication`을 상속받고 `.authenticate(self, request)` 메서드를 오버라이드하세요. 인증에 성공하면 `(user, auth)` 형태의 2-튜플을 반환하고, 그렇지 않으면 `None`을 반환해야 합니다.

일부 상황에서는 `None` 대신 `.authenticate()`에서 `AuthenticationFailed` 예외를 발생시키는 것이 더 적절할 수 있습니다.

일반적으로 권장되는 접근은 다음과 같습니다.

* 인증을 시도하지 않는 경우 `None`을 반환합니다. 그러면 다른 인증 방식들도 계속 검사됩니다.
* 인증을 시도했지만 실패한 경우 `AuthenticationFailed` 예외를 발생시킵니다. 그러면 permission 체크 여부와 무관하게 즉시 에러 응답이 반환되며, 다른 인증 방식은 더 이상 검사하지 않습니다.

또한 `.authenticate_header(self, request)` 메서드를 오버라이드할 수도 있습니다. 구현한다면, `HTTP 401 Unauthorized` 응답에서 `WWW-Authenticate` 헤더 값으로 사용할 문자열을 반환해야 합니다.

`.authenticate_header()`를 오버라이드하지 않으면, 비인증 요청이 거부될 때 인증 방식은 `HTTP 403 Forbidden` 응답을 반환합니다.

---

**참고:** 커스텀 인증기가 request 객체의 `.user` 또는 `.auth` 속성에 의해 호출될 때, `AttributeError`가 `WrappedAttributeError`로 다시 raise되는 것을 볼 수 있습니다. 이는 바깥쪽 속성 접근이 원래 예외를 억제하는 것을 방지하기 위해 필요합니다. Python은 해당 `AttributeError`가 커스텀 인증기에서 발생한 것인지 인지하지 못하고, request 객체에 `.user` 또는 `.auth` 속성이 없는 것으로 가정할 수 있습니다. 이런 오류는 인증기 구현에서 수정하거나 적절히 처리해야 합니다.

---

## 예제 (Example)

다음 예제는 `X-USERNAME`이라는 커스텀 요청 헤더에 들어 있는 username을 사용해 들어오는 모든 요청을 인증합니다.

    from django.contrib.auth.models import User
    from rest_framework import authentication
    from rest_framework import exceptions

    class ExampleAuthentication(authentication.BaseAuthentication):
        def authenticate(self, request):
            username = request.META.get('HTTP_X_USERNAME')
            if not username:
                return None

            try:
                user = User.objects.get(username=username)
            except User.DoesNotExist:
                raise exceptions.AuthenticationFailed('No such user')

            return (user, None)

---

# 서드파티 패키지 (Third party packages)

다음과 같은 서드파티 패키지도 사용할 수 있습니다.

## django-rest-knox

[Django-rest-knox][django-rest-knox] 라이브러리는 내장 TokenAuthentication 방식보다 더 안전하고 확장 가능한 방식으로 토큰 기반 인증을 다루기 위한 모델과 뷰를 제공합니다(싱글 페이지 앱 및 모바일 클라이언트 중심). 클라이언트별 토큰을 제공하며, 다른 인증(보통 basic 인증)이 제공된 경우 토큰을 생성하는 뷰, 토큰을 삭제하는 뷰(서버 강제 로그아웃), 모든 토큰을 삭제하는 뷰(사용자가 로그인한 모든 클라이언트 로그아웃)를 제공합니다.

## Django OAuth Toolkit

[Django OAuth Toolkit][django-oauth-toolkit] 패키지는 OAuth 2.0 지원을 제공하며 Python 3.4+에서 동작합니다. 이 패키지는 [jazzband][jazzband]에서 유지보수하며 훌륭한 [OAuthLib][oauthlib]를 사용합니다. 문서화가 잘 되어 있고 지원도 활발하며, 현재 **OAuth 2.0 지원을 위한 권장 패키지**입니다.

### 설치 및 설정 (Installation & configuration)

`pip`로 설치합니다.

    pip install django-oauth-toolkit

패키지를 `INSTALLED_APPS`에 추가하고 REST framework 설정을 수정합니다.

    INSTALLED_APPS = [
        ...
        'oauth2_provider',
    ]

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        ]
    }

자세한 내용은 [Django REST framework - Getting started][django-oauth-toolkit-getting-started] 문서를 참고하세요.

## Django REST framework OAuth

[Django REST framework OAuth][django-rest-framework-oauth] 패키지는 REST framework에서 OAuth1 및 OAuth2 지원을 제공합니다.

이 패키지는 과거에는 REST framework에 직접 포함되어 있었지만, 현재는 서드파티 패키지로 지원 및 유지보수됩니다.

### 설치 및 설정 (Installation & configuration)

`pip`로 설치합니다.

    pip install djangorestframework-oauth

설정 및 사용법은 Django REST framework OAuth 문서의 [authentication][django-rest-framework-oauth-authentication] 및 [permissions][django-rest-framework-oauth-permissions]를 참고하세요.

## JSON Web Token Authentication

JSON Web Token(JWT)은 토큰 기반 인증에 사용할 수 있는 비교적 새로운 표준입니다. 내장 TokenAuthentication과 달리, JWT 인증은 토큰 검증을 위해 데이터베이스를 사용할 필요가 없습니다. JWT 인증 패키지로는 [djangorestframework-simplejwt][djangorestframework-simplejwt]가 있으며, 여러 기능과 함께 플러그인 형태의 토큰 블랙리스트 앱을 제공합니다.

## Hawk HTTP Authentication

[HawkREST][hawkrest] 라이브러리는 [Mohawk][mohawk] 라이브러리를 기반으로, API에서 [Hawk][hawk] 서명 요청/응답을 다룰 수 있게 해줍니다. [Hawk][hawk]는 공유 키로 서명된 메시지를 사용해 두 당사자가 안전하게 통신하도록 합니다. 이는 [HTTP MAC access authentication][mac](OAuth 1.0 일부를 기반으로 했던 방식)을 기반으로 합니다.

## HTTP Signature Authentication

HTTP Signature(현재 [IETF 드래프트][http-signature-ietf-draft])는 HTTP 메시지에 대해 원본 인증(origin authentication)과 메시지 무결성을 달성하는 방법을 제공합니다. Amazon의 여러 서비스에서 사용하는 [Amazon HTTP Signature 방식][amazon-http-signature]과 유사하게, 요청 단위의 무상태(stateless) 인증을 허용합니다. [Elvio Toccalino][etoccalino]가 유지보수하는 (구버전) [djangorestframework-httpsignature][djangorestframework-httpsignature] 패키지는 쉽게 사용할 수 있는 HTTP Signature 인증 메커니즘을 제공합니다. 또한 업데이트된 포크 버전인 [djangorestframework-httpsignature][djangorestframework-httpsignature]의 [drf-httpsig][drf-httpsig]를 사용할 수도 있습니다.

## Djoser

[Djoser][djoser] 라이브러리는 회원가입, 로그인, 로그아웃, 비밀번호 재설정, 계정 활성화 등 기본 동작을 처리하는 뷰 세트를 제공합니다. 커스텀 유저 모델과 함께 동작하며 토큰 기반 인증을 사용합니다. Django 인증 시스템의 “바로 사용할 수 있는” REST 구현입니다.

## DRF Auth Kit

[DRF Auth Kit][drf-auth-kit] 라이브러리는 JWT 쿠키, 소셜 로그인, 다중 인증(MFA), 포괄적인 사용자 관리를 포함한 현대적인 REST 인증 솔루션을 제공합니다. DRF Spectacular로 자동 OpenAPI 스키마 생성을 지원하고, 완전한 타입 안정성을 제공하는 것이 특징입니다. 여러 인증 유형(JWT, DRF Token, 또는 Custom)을 지원하며 50개 이상의 언어에 대한 국제화도 포함합니다.

## django-rest-auth / dj-rest-auth

이 라이브러리는 회원가입, 인증(소셜 미디어 인증 포함), 비밀번호 재설정, 사용자 상세 조회 및 업데이트 등을 위한 REST API 엔드포인트 세트를 제공합니다. AngularJS, iOS, Android 등 클라이언트 앱이 Django 백엔드와 독립적으로 통신하여 사용자 관리를 REST API로 수행할 수 있습니다.

현재 이 프로젝트에는 두 개의 포크가 있습니다.

* [Django-rest-auth][django-rest-auth]는 원래 프로젝트이지만, [현재 업데이트가 거의 이루어지지 않습니다](https://github.com/Tivix/django-rest-auth/issues/568).
* [Dj-rest-auth][dj-rest-auth]는 더 최신의 포크입니다.

## drf-social-oauth2

[Drf-social-oauth2][drf-social-oauth2]는 Facebook, Google, Twitter, Orcid 등 주요 소셜 oauth2 벤더로 인증할 수 있도록 도와주는 프레임워크입니다. 손쉬운 설정으로 JWT 형태로 토큰을 생성합니다.

## drfpasswordless

[drfpasswordless][drfpasswordless]는 Django REST Framework의 TokenAuthentication 방식에 비밀번호 없는(passwordless) 지원을 추가합니다(예: Medium, Square Cash에서 영감을 받은 방식). 사용자는 이메일 주소나 휴대폰 번호 같은 연락처로 전송된 토큰을 이용해 로그인/가입합니다.

## django-rest-authemail

[django-rest-authemail][django-rest-authemail]는 username 대신 이메일 주소를 인증에 사용하는 사용자 가입 및 인증을 위한 RESTful API를 제공합니다. 회원가입, 가입 이메일 인증, 로그인, 로그아웃, 비밀번호 재설정, 비밀번호 재설정 검증, 이메일 변경, 이메일 변경 검증, 비밀번호 변경, 사용자 상세 등의 엔드포인트가 제공됩니다. 동작하는 예제 프로젝트와 자세한 안내도 포함되어 있습니다.

## Django-Rest-Durin

[Django-Rest-Durin][django-rest-durin]은 Web/CLI/Mobile 등 여러 API 클라이언트에 대해 하나의 인터페이스로 토큰 인증을 제공하는 것을 목표로 합니다. API 클라이언트마다 토큰 설정을 다르게 할 수 있으며, 커스텀 모델/뷰/permission을 통해 사용자당 여러 토큰을 지원합니다. 토큰 만료 시간은 API 클라이언트별로 다를 수 있고, Django Admin에서 커스터마이징할 수 있습니다.

자세한 내용은 [문서](https://django-rest-durin.readthedocs.io/en/latest/index.html)에서 확인할 수 있습니다.

## django-pyoidc

[dango-pyoidc][django_pyoidc]는 OpenID Connect(OIDC) 인증을 지원합니다. 이를 통해 사용자 관리를 Identity Provider에 위임할 수 있으며, Single-Sign-On(SSO) 구현에 사용할 수 있습니다. 토큰 정보를 사용자 모델에 매핑하는 방법을 커스터마이징하거나, OIDC audience를 사용해 접근 제어를 하는 등 대부분의 사용 케이스를 지원합니다.

자세한 내용은 [문서](https://django-pyoidc.readthedocs.io/latest/index.html)에서 확인할 수 있습니다.

[cite]: https://jacobian.org/writing/rest-worst-practices/
[http401]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.2
[http403]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.4
[basicauth]: https://tools.ietf.org/html/rfc2617
[permission]: permissions.md
[throttling]: throttling.md
[csrf-ajax]: https://docs.djangoproject.com/en/stable/howto/csrf/#using-csrf-protection-with-ajax
[mod_wsgi_official]: https://modwsgi.readthedocs.io/en/develop/configuration-directives/WSGIPassAuthorization.html
[django-oauth-toolkit-getting-started]: https://django-oauth-toolkit.readthedocs.io/en/latest/rest-framework/getting_started.html
[django-rest-framework-oauth]: https://jpadilla.github.io/django-rest-framework-oauth/
[django-rest-framework-oauth-authentication]: https://jpadilla.github.io/django-rest-framework-oauth/authentication/
[django-rest-framework-oauth-permissions]: https://jpadilla.github.io/django-rest-framework-oauth/permissions/
[django-oauth-toolkit]: https://github.com/evonove/django-oauth-toolkit
[jazzband]: https://github.com/jazzband/
[oauthlib]: https://github.com/idan/oauthlib
[djangorestframework-simplejwt]: https://github.com/davesque/django-rest-framework-simplejwt
[etoccalino]: https://github.com/etoccalino/
[djangorestframework-httpsignature]: https://github.com/etoccalino/django-rest-framework-httpsignature
[drf-httpsig]: https://github.com/ahknight/drf-httpsig
[amazon-http-signature]: https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html
[http-signature-ietf-draft]: https://datatracker.ietf.org/doc/draft-cavage-http-signatures/
[hawkrest]: https://hawkrest.readthedocs.io/en/latest/
[hawk]: https://github.com/hueniverse/hawk
[mohawk]: https://mohawk.readthedocs.io/en/latest/
[mac]: https://tools.ietf.org/html/draft-hammer-oauth-v2-mac-token-05
[djoser]: https://github.com/sunscrapers/djoser
[django-rest-auth]: https://github.com/Tivix/django-rest-auth
[dj-rest-auth]: https://github.com/jazzband/dj-rest-auth
[drf-social-oauth2]: https://github.com/wagnerdelima/drf-social-oauth2
[django-rest-knox]: https://github.com/James1345/django-rest-knox
[drfpasswordless]: https://github.com/aaronn/django-rest-framework-passwordless
[django-rest-authemail]: https://github.com/celiao/django-rest-authemail
[django-rest-durin]: https://github.com/eshaan7/django-rest-durin
[login-required-middleware]: https://docs.djangoproject.com/en/stable/ref/middleware/#django.contrib.auth.middleware.LoginRequiredMiddleware
[drf-auth-kit]: https://github.com/huynguyengl99/drf-auth-kit
