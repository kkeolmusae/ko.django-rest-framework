---
source:
    - routers.py
---

# 라우터 (Routers)

> 리소스 라우팅을 사용하면 특정 리소스형 컨트롤러에 대해 공통적으로 사용되는 모든 라우트를 빠르게 선언할 수 있습니다.  
> index, show 등의 개별 라우트를 각각 선언하는 대신, 리소스 라우트는 한 줄의 코드로 이를 모두 선언합니다.
>
> &mdash; [Ruby on Rails 문서][cite]

Rails와 같은 일부 웹 프레임워크는 애플리케이션의 URL을 들어오는 요청을 처리하는 로직과 어떻게 매핑할지 자동으로 결정해주는 기능을 제공합니다.

Django REST framework는 Django에 자동 URL 라우팅에 대한 지원을 추가하여, 뷰 로직을 URL 집합에 연결하는 간단하고 빠르며 일관된 방법을 제공합니다.

## 사용법 (Usage)

다음은 `SimpleRouter`를 사용하는 간단한 URL 설정 예제입니다.

    from rest_framework import routers

    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)
    urlpatterns = router.urls

`register()` 메서드에는 두 개의 필수 인자가 있습니다.

* `prefix` - 이 라우트 집합에 사용할 URL 접두사
* `viewset` - 뷰셋 클래스

선택적으로 다음 인자를 추가할 수도 있습니다.

* `basename` - 생성될 URL 이름의 기본(base) 값  
  지정하지 않으면, 뷰셋에 `queryset` 속성이 있는 경우 이를 기반으로 자동 생성됩니다.  
  단, 뷰셋에 `queryset` 속성이 없다면 반드시 `basename`을 명시적으로 지정해야 합니다.

위 예제는 다음과 같은 URL 패턴을 생성합니다.

* URL 패턴: `^users/$`  이름: `'user-list'`
* URL 패턴: `^users/{pk}/$`  이름: `'user-detail'`
* URL 패턴: `^accounts/$`  이름: `'account-list'`
* URL 패턴: `^accounts/{pk}/$`  이름: `'account-detail'`

---

**참고**: `basename` 인자는 뷰 이름 패턴의 앞부분을 지정하는 데 사용됩니다.  
위 예제에서는 `user`, `account` 부분에 해당합니다.

일반적으로 `basename`을 직접 지정할 필요는 없습니다.  
하지만 커스텀 `get_queryset` 메서드를 정의한 뷰셋의 경우, `.queryset` 속성이 설정되어 있지 않을 수 있습니다.  
이런 뷰셋을 등록하려고 하면 다음과 같은 에러가 발생합니다.

    'basename' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.

이는 모델 이름으로부터 `basename`을 자동으로 추론할 수 없다는 의미이므로, 뷰셋을 등록할 때 `basename`을 명시적으로 지정해야 합니다.

---

### 라우터와 함께 `include` 사용하기

라우터 인스턴스의 `.urls` 속성은 표준 Django URL 패턴 리스트입니다.  
이를 포함시키는 방법에는 여러 가지가 있습니다.

예를 들어, 기존 URL 패턴 리스트에 `router.urls`를 추가할 수 있습니다.

    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)

    urlpatterns = [
        path('forgot-password/', ForgotPasswordFormView.as_view()),
    ]

    urlpatterns += router.urls

또는 Django의 `include` 함수를 사용할 수도 있습니다.

    urlpatterns = [
        path('forgot-password', ForgotPasswordFormView.as_view()),
        path('', include(router.urls)),
    ]

애플리케이션 네임스페이스와 함께 사용할 수도 있습니다.

    urlpatterns = [
        path('forgot-password/', ForgotPasswordFormView.as_view()),
        path('api/', include((router.urls, 'app_name'))),
    ]

애플리케이션 네임스페이스와 인스턴스 네임스페이스를 모두 사용할 수도 있습니다.

    urlpatterns = [
        path('forgot-password/', ForgotPasswordFormView.as_view()),
        path('api/', include((router.urls, 'app_name'), namespace='instance_name')),
    ]

자세한 내용은 Django의 [URL 네임스페이스 문서][url-namespace-docs]와 [`include` API 레퍼런스][include-api-reference]를 참고하세요.

---

**참고**: 하이퍼링크드(serializer) 시리얼라이저에서 네임스페이싱을 사용하는 경우,  
시리얼라이저의 `view_name` 파라미터도 해당 네임스페이스를 올바르게 반영해야 합니다.  
위 예제에서는 사용자 상세 뷰에 대해 `view_name='app_name:user-detail'` 과 같이 지정해야 합니다.

자동 `view_name` 생성은 `%(model_name)-detail` 형태의 패턴을 사용합니다.  
모델 이름이 충돌하지 않는다면, 하이퍼링크드 시리얼라이저를 사용할 때는  
Django REST Framework 뷰를 **네임스페이싱하지 않는 편이 더 나을 수 있습니다**.

---

### 추가 액션을 위한 라우팅

뷰셋은 `@action` 데코레이터를 사용해 [추가 액션을 라우팅 대상으로 표시][route-decorators]할 수 있습니다.  
이 추가 액션들은 자동으로 생성되는 라우트에 포함됩니다.

예를 들어, `UserViewSet` 클래스에 `set_password` 메서드가 있다면 다음과 같습니다.

    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import action

    class UserViewSet(ModelViewSet):
        ...

        @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf])
        def set_password(self, request, pk=None):
            ...

다음과 같은 라우트가 생성됩니다.

* URL 패턴: `^users/{pk}/set_password/$`
* URL 이름: `'user-set-password'`

기본적으로 URL 패턴은 메서드 이름을 기반으로 하며,  
URL 이름은 `ViewSet.basename`과 하이픈으로 연결된 메서드 이름의 조합입니다.

기본값을 사용하지 않으려면, `@action` 데코레이터에 `url_path`와 `url_name` 인자를 직접 지정할 수 있습니다.

예를 들어 URL을 `^users/{pk}/change-password/$`로 변경하고 싶다면 다음과 같이 작성할 수 있습니다.

    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import action

    class UserViewSet(ModelViewSet):
        ...

        @action(methods=['post'], detail=True, permission_classes=[IsAdminOrIsSelf],
                url_path='change-password', url_name='change_password')
        def set_password(self, request, pk=None):
            ...

이 경우 다음과 같은 URL이 생성됩니다.

* URL 경로: `^users/{pk}/change-password/$`
* URL 이름: `'user-change_password'`

### 라우터에서 Django `path()` 사용하기

기본적으로 라우터가 생성하는 URL은 정규식을 사용합니다.  
이 동작은 라우터 생성 시 `use_regex_path=False`를 설정하면 변경할 수 있으며, 이 경우 [path 컨버터][path-converters-topic-reference]가 사용됩니다.

    router = SimpleRouter(use_regex_path=False)

라우터는 슬래시(`/`)와 점(`.`)을 제외한 모든 문자를 포함하는 lookup 값을 매칭합니다.  
보다 엄격하거나 느슨한 패턴이 필요하다면, 뷰셋에 `lookup_value_regex`를 설정하거나  
path 컨버터를 사용할 경우 `lookup_value_converter`를 설정할 수 있습니다.

예를 들어 lookup 값을 UUID로 제한하려면 다음과 같이 작성할 수 있습니다.

    class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
        lookup_field = 'my_model_id'
        lookup_value_regex = '[0-9a-f]{32}'

    class MyPathModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
        lookup_field = 'my_model_uuid'
        lookup_value_converter = 'uuid'

path 컨버터는 뷰셋 액션을 포함해 라우터에 등록된 모든 URL에 적용됩니다.

# API 가이드

## SimpleRouter

이 라우터는 `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy` 액션에 대한 기본 라우트를 포함합니다.  
또한 `@action` 데코레이터를 사용해 추가 메서드를 라우팅할 수 있습니다.

<table border=1>
    <tr><th>URL 형식</th><th>HTTP 메서드</th><th>액션</th><th>URL 이름</th></tr>
    <tr><td rowspan=2>{prefix}/</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{url_path}/</td><td>GET (또는 methods 인자로 지정된 메서드)</td><td>`@action(detail=False)` 메서드</td><td>{basename}-{url_name}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{url_path}/</td><td>GET (또는 methods 인자로 지정된 메서드)</td><td>`@action(detail=True)` 메서드</td><td>{basename}-{url_name}</td></tr>
</table>

기본적으로 `SimpleRouter`가 생성하는 URL에는 trailing slash(`/`)가 붙습니다.  
이 동작은 라우터 생성 시 `trailing_slash=False`로 변경할 수 있습니다.

    router = SimpleRouter(trailing_slash=False)

Django에서는 trailing slash가 관례이지만, Rails와 같은 일부 프레임워크에서는 기본적으로 사용하지 않습니다.  
어떤 방식을 사용할지는 취향의 문제이며, 일부 JavaScript 프레임워크는 특정 라우팅 스타일을 기대할 수도 있습니다.

## DefaultRouter

이 라우터는 `SimpleRouter`와 유사하지만, 추가로 기본 API 루트 뷰를 포함합니다.  
이 루트 뷰는 모든 list 뷰에 대한 하이퍼링크를 반환합니다.  
또한 `.json`과 같은 포맷 접미사(format suffix)에 대한 라우트도 생성합니다.

<table border=1>
    <tr><th>URL 형식</th><th>HTTP 메서드</th><th>액션</th><th>URL 이름</th></tr>
    <tr><td>[.format]</td><td>GET</td><td>자동 생성된 루트 뷰</td><td>api-root</td></tr></tr>
    <tr><td rowspan=2>{prefix}/[.format]</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{url_path}/[.format]</td><td>GET (또는 methods 인자로 지정된 메서드)</td><td>`@action(detail=False)` 메서드</td><td>{basename}-{url_name}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/[.format]</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{url_path}/[.format]</td><td>GET (또는 methods 인자로 지정된 메서드)</td><td>`@action(detail=True)` 메서드</td><td>{basename}-{url_name}</td></tr>
</table>

`SimpleRouter`와 마찬가지로, `trailing_slash=False`를 설정해 URL의 trailing slash를 제거할 수 있습니다.

    router = DefaultRouter(trailing_slash=False)

# 커스텀 라우터 (Custom Routers)

커스텀 라우터를 구현할 일은 많지 않지만, API URL 구조에 대한 특정 요구사항이 있는 경우 유용할 수 있습니다. 이를 통해 각 뷰마다 URL 패턴을 직접 작성하지 않아도 되도록, URL 구조를 재사용 가능한 형태로 캡슐화할 수 있습니다.

가장 간단한 방법은 기존 라우터 클래스를 상속받는 것입니다.  
`.routes` 속성은 각 뷰셋에 매핑될 URL 패턴의 템플릿 역할을 하며,  
`Route` 네임드 튜플의 리스트로 구성됩니다.

`Route` 네임드 튜플의 인자는 다음과 같습니다.

**url**: 라우팅될 URL 문자열  
다음과 같은 포맷 문자열을 포함할 수 있습니다.

* `{prefix}` - 이 라우트 집합에 사용할 URL 접두사
* `{lookup}` - 단일 인스턴스를 매칭하는 lookup 필드
* `{trailing_slash}` - `/` 또는 빈 문자열

**mapping**: HTTP 메서드와 뷰 메서드의 매핑

**name**: `reverse` 호출에 사용되는 URL 이름  
다음 포맷 문자열을 포함할 수 있습니다.

* `{basename}` - URL 이름의 기본값

**initkwargs**: 뷰 인스턴스 생성 시 전달될 추가 인자 딕셔너리  
`detail`, `basename`, `suffix` 인자는 예약되어 있으며,  
브라우저블 API에서 뷰 이름과 breadcrumb 생성을 위해 사용됩니다.

## 동적 라우트 커스터마이징

`@action` 데코레이터가 어떻게 라우팅되는지도 커스터마이징할 수 있습니다.  
`.routes` 리스트에 `DynamicRoute` 네임드 튜플을 포함시키면 됩니다.

`DynamicRoute`의 인자는 다음과 같습니다.

**url**: 라우팅될 URL 문자열  
`Route`와 동일한 포맷 문자열과 함께 `{url_path}`를 추가로 사용할 수 있습니다.

**name**: `reverse` 호출에 사용되는 URL 이름  
다음 포맷 문자열을 포함할 수 있습니다.

* `{basename}`
* `{url_name}` - `@action`에 제공된 `url_name`

**initkwargs**: 뷰 인스턴스 생성 시 전달될 추가 인자 딕셔너리

## 예제

다음 예제는 `list`와 `retrieve` 액션만 라우팅하며, trailing slash를 사용하지 않습니다.

    from rest_framework.routers import Route, DynamicRoute, SimpleRouter

    class CustomReadOnlyRouter(SimpleRouter):
        """
        trailing slash를 사용하지 않는 읽기 전용 API 라우터
        """
        routes = [
            Route(
                url=r'^{prefix}$',
                mapping={'get': 'list'},
                name='{basename}-list',
                detail=False,
                initkwargs={'suffix': 'List'}
            ),
            Route(
                url=r'^{prefix}/{lookup}$',
                mapping={'get': 'retrieve'},
                name='{basename}-detail',
                detail=True,
                initkwargs={'suffix': 'Detail'}
            ),
            DynamicRoute(
                url=r'^{prefix}/{lookup}/{url_path}$',
                name='{basename}-{url_name}',
                detail=True,
                initkwargs={}
            )
        ]

이제 `CustomReadOnlyRouter`가 생성하는 라우트를 간단한 뷰셋에 적용해 보겠습니다.

`views.py`:

    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        표준 액션을 제공하는 뷰셋
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer
        lookup_field = 'username'

        @action(detail=True)
        def group_names(self, request, pk=None):
            """
            해당 사용자가 속한 모든 그룹 이름을 반환합니다.
            """
            user = self.get_object()
            groups = user.groups.all()
            return Response([group.name for group in groups])

`urls.py`:

    router = CustomReadOnlyRouter()
    router.register('users', UserViewSet)
    urlpatterns = router.urls

다음과 같은 매핑이 생성됩니다.

<table border=1>
    <tr><th>URL</th><th>HTTP 메서드</th><th>액션</th><th>URL 이름</th></tr>
    <tr><td>/users</td><td>GET</td><td>list</td><td>user-list</td></tr>
    <tr><td>/users/{username}</td><td>GET</td><td>retrieve</td><td>user-detail</td></tr>
    <tr><td>/users/{username}/group_names</td><td>GET</td><td>group_names</td><td>user-group-names</td></tr>
</table>

`.routes` 속성을 설정하는 또 다른 예시는 `SimpleRouter` 클래스의 소스 코드를 참고하세요.

## 고급 커스텀 라우터

완전히 커스텀한 동작이 필요하다면 `BaseRouter`를 상속받아  
`get_urls(self)` 메서드를 오버라이드할 수 있습니다.  
이 메서드는 등록된 뷰셋을 검사하여 URL 패턴 리스트를 반환해야 합니다.

등록된 `(prefix, viewset, basename)` 튜플은 `self.registry` 속성을 통해 확인할 수 있습니다.

또한 `get_default_basename(self, viewset)` 메서드를 오버라이드하거나,  
라우터에 뷰셋을 등록할 때 항상 `basename`을 명시적으로 지정할 수도 있습니다.

# 서드파티 패키지 (Third Party Packages)

다음과 같은 서드파티 패키지들도 사용할 수 있습니다.

## DRF Nested Routers

[drf-nested-routers 패키지][drf-nested-routers]는 중첩된 리소스를 다루기 위한 라우터와 관계 필드를 제공합니다.

## ModelRouter (wq.db.rest)

[wq.db 패키지][wq.db]는 `register_model()` API를 제공하는  
고급 [ModelRouter][wq.db-router] 클래스를 제공합니다.

Django의 `admin.site.register`와 유사하게, `rest.router.register_model`에는 모델 클래스 하나만 전달하면 됩니다. URL prefix, serializer, viewset에 대한 합리적인 기본값이 모델과 전역 설정을 기반으로 자동 추론됩니다.

    from wq.db import rest
    from myapp.models import MyModel

    rest.router.register_model(MyModel)

## DRF-extensions

[`DRF-extensions` 패키지][drf-extensions]는 중첩 뷰셋, 컬렉션 레벨 컨트롤러, 커스터마이즈 가능한 엔드포인트 이름을 위한 다양한 [라우터][drf-extensions-routers]를 제공합니다.

[cite]: https://guides.rubyonrails.org/routing.html
[route-decorators]: viewsets.md#marking-extra-actions-for-routing
[drf-nested-routers]: https://github.com/alanjds/drf-nested-routers
[wq.db]: https://wq.io/wq.db
[wq.db-router]: https://wq.io/docs/router
[drf-extensions]: https://chibisov.github.io/drf-extensions/docs/
[drf-extensions-routers]: https://chibisov.github.io/drf-extensions/docs/#routers
[url-namespace-docs]: https://docs.djangoproject.com/en/stable/topics/http/urls/#url-namespaces
[include-api-reference]: https://docs.djangoproject.com/en/stable/ref/urls/#include
[path-converters-topic-reference]: https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters
