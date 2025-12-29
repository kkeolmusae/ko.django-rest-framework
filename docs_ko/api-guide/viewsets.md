---
source:
    - viewsets.py
---

# ViewSets

> 라우팅을 통해 어떤 컨트롤러를 사용할지 결정되면, 컨트롤러는 요청을 이해하고 적절한 출력을 생성하는 책임을 집니다.
>
> &mdash; [Ruby on Rails Documentation][cite]

Django REST framework는 서로 관련된 여러 뷰의 로직을 하나의 클래스에 결합할 수 있도록 `ViewSet`이라는 개념을 제공합니다. 다른 프레임워크에서도 개념적으로 유사한 구현을 ‘Resources’ 또는 ‘Controllers’와 같은 이름으로 찾아볼 수 있습니다.

`ViewSet` 클래스는 `.get()`이나 `.post()` 같은 메서드 핸들러를 제공하지 않는 **클래스 기반 View의 한 유형**이며, 대신 `.list()`, `.create()`와 같은 액션을 제공합니다.

`ViewSet`의 메서드 핸들러는 `.as_view()` 메서드를 통해 뷰가 최종적으로 생성되는 시점에 해당 액션과 바인딩됩니다.

일반적으로 viewset의 각 뷰를 urlconf에 명시적으로 등록하기보다는, router 클래스에 viewset을 등록하여 URL 구성을 자동으로 생성하도록 합니다.

## Example

시스템의 모든 사용자를 목록 조회(list)하거나 단일 사용자 조회(retrieve)할 수 있는 간단한 viewset을 정의해봅시다.

    from django.contrib.auth.models import User
    from django.shortcuts import get_object_or_404
    from myapps.serializers import UserSerializer
    from rest_framework import viewsets
    from rest_framework.response import Response

    class UserViewSet(viewsets.ViewSet):
        """
        사용자를 목록 조회하거나 단일 조회하는 간단한 ViewSet
        """
        def list(self, request):
            queryset = User.objects.all()
            serializer = UserSerializer(queryset, many=True)
            return Response(serializer.data)

        def retrieve(self, request, pk=None):
            queryset = User.objects.all()
            user = get_object_or_404(queryset, pk=pk)
            serializer = UserSerializer(user)
            return Response(serializer.data)

필요하다면 다음과 같이 이 viewset을 두 개의 개별 뷰로 바인딩할 수도 있습니다.

    user_list = UserViewSet.as_view({'get': 'list'})
    user_detail = UserViewSet.as_view({'get': 'retrieve'})

하지만 일반적으로는 이렇게 하지 않고, router에 viewset을 등록하여 urlconf가 자동으로 생성되도록 합니다.

    from myapp.views import UserViewSet
    from rest_framework.routers import DefaultRouter

    router = DefaultRouter()
    router.register(r'users', UserViewSet, basename='user')
    urlpatterns = router.urls

!!! warning
    `@action` 메서드와 함께 `.as_view()`를 사용하지 마세요.  
    이는 router 설정을 우회하며 `permission_classes` 같은 action 설정을 무시할 수 있습니다.  
    action에는 `DefaultRouter`를 사용하세요.

직접 viewset을 작성하기보다는, 기본 동작을 제공하는 기존 베이스 클래스를 사용하는 경우가 많습니다. 예를 들어 다음과 같습니다.

    class UserViewSet(viewsets.ModelViewSet):
        """
        사용자 인스턴스를 조회하고 수정하기 위한 ViewSet
        """
        serializer_class = UserSerializer
        queryset = User.objects.all()

`ViewSet`을 사용하는 것은 일반 `View`를 사용하는 것에 비해 두 가지 주요 장점이 있습니다.

* 반복되는 로직을 하나의 클래스에 결합할 수 있습니다. 위 예제에서는 `queryset`을 한 번만 정의하면 여러 뷰에서 재사용됩니다.
* router를 사용함으로써 URL conf를 직접 연결할 필요가 없습니다.

다만, 이에 따른 트레이드오프도 존재합니다. 일반 뷰와 URL conf를 사용하는 방식은 더 명시적이며 세밀한 제어가 가능합니다. ViewSet은 빠르게 개발을 시작하거나, 규모가 큰 API에서 일관된 URL 구성을 강제하고자 할 때 유용합니다.

## ViewSet actions

REST framework에 포함된 기본 router는 다음과 같은 표준 create / retrieve / update / destroy 액션에 대한 라우트를 자동으로 제공합니다.

    class UserViewSet(viewsets.ViewSet):
        """
        router 클래스에 의해 처리되는
        표준 액션들을 보여주는 예제 ViewSet

        format suffix를 사용하는 경우,
        각 액션에 format=None 인자를 포함해야 합니다.
        """

        def list(self, request):
            pass

        def create(self, request):
            pass

        def retrieve(self, request, pk=None):
            pass

        def update(self, request, pk=None):
            pass

        def partial_update(self, request, pk=None):
            pass

        def destroy(self, request, pk=None):
            pass

## ViewSet actions 정보 확인(Introspection)

요청이 dispatch되는 동안, `ViewSet`에서는 다음과 같은 속성들을 사용할 수 있습니다.

* `basename` - 생성되는 URL name의 기준이 되는 이름
* `action` - 현재 액션의 이름 (예: `list`, `create`)
* `detail` - 현재 액션이 목록 뷰인지, 상세 뷰인지를 나타내는 boolean 값
* `suffix` - viewset 타입에 대한 표시용 접미사 (`detail` 속성과 동일한 의미)
* `name` - viewset의 표시 이름 (`suffix`와 상호 배타적)
* `description` - viewset의 개별 뷰에 대한 설명

이 속성들을 활용하여 현재 액션에 따라 동작을 조정할 수 있습니다. 예를 들어 `list` 액션에만 접근 권한을 허용하려면 다음과 같이 구현할 수 있습니다.

    def get_permissions(self):
        """
        이 뷰에서 필요한 permission 인스턴스 목록을 생성하여 반환
        """
        if self.action == 'list':
            permission_classes = [IsAuthenticated]
        else:
            permission_classes = [IsAdminUser]
        return [permission() for permission in permission_classes]

**Note**: `action` 속성은 `get_parsers`, `get_authenticators`, `get_content_negotiator` 메서드에서는 사용할 수 없습니다.  
이는 해당 속성이 프레임워크 라이프사이클에서 이 메서드들이 호출된 이후에 설정되기 때문입니다.  
이 메서드들 안에서 `action`에 접근하면 `AttributeError`가 발생합니다.

## 추가 액션을 라우팅 대상으로 표시하기

임의로 정의한 메서드를 라우팅 대상으로 만들고 싶다면 `@action` 데코레이터를 사용할 수 있습니다.  
일반 액션과 마찬가지로, 추가 액션은 단일 객체(detail)용이거나 컬렉션(list)용일 수 있습니다.  
이를 나타내기 위해 `detail=True` 또는 `False`를 지정합니다. router는 이에 맞게 URL 패턴을 구성합니다.  
예를 들어 `DefaultRouter`는 detail 액션의 URL에 `pk`를 포함시킵니다.

추가 액션의 보다 완전한 예시는 다음과 같습니다.

    from django.contrib.auth.models import User
    from rest_framework import status, viewsets
    from rest_framework.decorators import action
    from rest_framework.response import Response
    from myapp.serializers import UserSerializer, PasswordSerializer

    class UserViewSet(viewsets.ModelViewSet):
        """
        표준 액션을 제공하는 ViewSet
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

        @action(detail=True, methods=['post'])
        def set_password(self, request, pk=None):
            user = self.get_object()
            serializer = PasswordSerializer(data=request.data)
            if serializer.is_valid():
                user.set_password(serializer.validated_data['password'])
                user.save()
                return Response({'status': 'password set'})
            else:
                return Response(serializer.errors,
                                status=status.HTTP_400_BAD_REQUEST)

        @action(detail=False)
        def recent_users(self, request):
            recent_users = User.objects.all().order_by('-last_login')

            page = self.paginate_queryset(recent_users)
            if page is not None:
                serializer = self.get_serializer(page, many=True)
                return self.get_paginated_response(serializer.data)

            serializer = self.get_serializer(recent_users, many=True)
            return Response(serializer.data)

`action` 데코레이터는 기본적으로 `GET` 요청을 라우팅하지만, `methods` 인자를 통해 다른 HTTP 메서드도 허용할 수 있습니다. 예를 들어 다음과 같습니다.

        @action(detail=True, methods=['post', 'delete'])
        def unset_password(self, request, pk=None):
           ...

`methods` 인자는 [HTTPMethod](https://docs.python.org/3/library/http.html#http.HTTPMethod)에 정의된 HTTP 메서드도 지원합니다. 아래 예시는 위와 동일한 동작을 합니다.

        from http import HTTPMethod

        @action(detail=True, methods=[HTTPMethod.POST, HTTPMethod.DELETE])
        def unset_password(self, request, pk=None):
           ...

또한 데코레이터를 통해 `permission_classes`, `serializer_class`, `filter_backends` 등 viewset 레벨의 설정을 덮어쓸 수 있습니다.

        @action(detail=True, methods=['post'], permission_classes=[IsAdminOrIsSelf])
        def set_password(self, request, pk=None):
           ...

이 두 개의 새로운 액션은 각각  
`^users/{pk}/set_password/$`, `^users/{pk}/unset_password/$` URL에서 사용할 수 있습니다.  
`url_path`와 `url_name` 파라미터를 사용하면 URL 경로와 reverse URL 이름을 변경할 수 있습니다.

모든 추가 액션을 확인하려면 `.get_extra_actions()` 메서드를 호출하세요.

### 추가 액션에 대한 HTTP 메서드 확장 라우팅

추가 액션은 여러 HTTP 메서드를 각각 다른 `ViewSet` 메서드에 매핑할 수도 있습니다.  
예를 들어 위의 비밀번호 설정/해제 메서드를 하나의 라우트로 통합할 수 있습니다.  
단, 추가 매핑 메서드는 인자를 받을 수 없습니다.

@action(detail=True, methods=["put"], name="Change Password")
def password(self, request, pk=None):
    """사용자의 비밀번호를 수정합니다."""
    ...

@password.mapping.delete
def delete_password(self, request, pk=None):
    """사용자의 비밀번호를 삭제합니다."""
    ...

## 액션 URL reverse 하기

액션의 URL을 얻고 싶다면 `.reverse_action()` 메서드를 사용하세요.  
이 메서드는 `reverse()`를 감싼 convenience wrapper로, 자동으로 뷰의 `request` 객체를 전달하고 `url_name` 앞에 `.basename`을 붙여줍니다.

`basename`은 `ViewSet`이 router에 등록될 때 제공됩니다. router를 사용하지 않는 경우에는 `.as_view()` 호출 시 `basename` 인자를 직접 지정해야 합니다.

앞선 예제를 사용하면 다음과 같습니다.

>>> view.reverse_action("set-password", args=["1"])
'<http://localhost:8000/api/users/1/set_password>'

또는 `@action` 데코레이터에 설정된 `url_name` 속성을 사용할 수도 있습니다.

>>> view.reverse_action(view.set_password.url_name, args=["1"])
'<http://localhost:8000/api/users/1/set_password>'

`.reverse_action()`의 `url_name` 인자는 `@action` 데코레이터에 전달한 값과 일치해야 합니다.  
또한 이 메서드는 `list`, `create`와 같은 기본 액션을 reverse하는 데에도 사용할 수 있습니다.

---

# API Reference

## ViewSet

`ViewSet` 클래스는 `APIView`를 상속합니다.  
`permission_classes`, `authentication_classes`와 같은 표준 속성을 사용하여 API 정책을 제어할 수 있습니다.

`ViewSet` 클래스 자체는 액션에 대한 구현을 제공하지 않으므로, 실제 사용 시에는 클래스를 상속하여 액션 메서드를 직접 정의해야 합니다.

## GenericViewSet

`GenericViewSet` 클래스는 `GenericAPIView`를 상속하며,  
`get_object`, `get_queryset`과 같은 기본 제네릭 뷰 동작을 제공하지만 액션은 기본으로 포함하지 않습니다.

`GenericViewSet`을 사용하려면 필요한 mixin 클래스를 조합하거나, 액션 메서드를 직접 구현해야 합니다.

## ModelViewSet

`ModelViewSet` 클래스는 `GenericAPIView`를 상속하며, 여러 mixin의 동작을 조합하여 다양한 액션 구현을 제공합니다.

`ModelViewSet`이 기본으로 제공하는 액션은  
`.list()`, `.retrieve()`, `.create()`, `.update()`, `.partial_update()`, `.destroy()` 입니다.

#### Example

`ModelViewSet`은 `GenericAPIView`를 확장하므로, 일반적으로 `queryset`과 `serializer_class`를 최소한으로 지정해야 합니다.

    class AccountViewSet(viewsets.ModelViewSet):
        """
        계정을 조회하고 수정하기 위한 간단한 ViewSet
        """
        queryset = Account.objects.all()
        serializer_class = AccountSerializer
        permission_classes = [IsAccountAdminOrReadOnly]

`GenericAPIView`에서 제공하는 표준 속성과 메서드 오버라이드를 그대로 사용할 수 있습니다.  
예를 들어, 동적으로 queryset을 결정하는 ViewSet은 다음과 같이 작성할 수 있습니다.

    class AccountViewSet(viewsets.ModelViewSet):
        """
        사용자와 연관된 계정만을 조회/수정하는 ViewSet
        """
        serializer_class = AccountSerializer
        permission_classes = [IsAccountAdminOrReadOnly]

        def get_queryset(self):
            return self.request.user.accounts.all()

다만 `ViewSet`에서 `queryset` 속성을 제거하면,  
연결된 [router][routers]가 모델의 basename을 자동으로 추론할 수 없으므로  
router 등록 시 `basename` 인자를 반드시 명시해야 합니다.

또한 이 클래스는 기본적으로 모든 CRUD 액션을 제공하지만,  
표준 permission 클래스를 사용하여 허용되는 동작을 제한할 수 있습니다.

## ReadOnlyModelViewSet

`ReadOnlyModelViewSet` 클래스 역시 `GenericAPIView`를 상속합니다.  
`ModelViewSet`과 유사하지만, 읽기 전용 액션인 `.list()`와 `.retrieve()`만 제공합니다.

#### Example

`ModelViewSet`과 마찬가지로 `queryset`과 `serializer_class`를 지정해야 합니다.

    class AccountViewSet(viewsets.ReadOnlyModelViewSet):
        """
        계정을 조회하기 위한 간단한 ViewSet
        """
        queryset = Account.objects.all()
        serializer_class = AccountSerializer

`GenericAPIView`에서 제공하는 모든 표준 속성과 메서드 오버라이드를 동일하게 사용할 수 있습니다.

# Custom ViewSet base classes

전체 `ModelViewSet` 액션을 제공하지 않거나, 특정 동작을 커스터마이징한  
자체 `ViewSet` 베이스 클래스를 정의해야 할 수도 있습니다.

## Example

`create`, `list`, `retrieve` 액션만 제공하는 베이스 viewset을 만들려면  
`GenericViewSet`을 상속하고 필요한 mixin을 조합하면 됩니다.

    from rest_framework import mixins, viewsets

    class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                    mixins.ListModelMixin,
                                    mixins.RetrieveModelMixin,
                                    viewsets.GenericViewSet):
        """
        `retrieve`, `create`, `list` 액션을 제공하는 ViewSet

        사용 시에는 이 클래스를 상속하고
        `.queryset`과 `.serializer_class`를 설정하세요.
        """
        pass

이처럼 공통 동작을 가진 베이스 `ViewSet` 클래스를 만들어두면,  
API 전반에서 재사용할 수 있는 일관된 동작을 제공할 수 있습니다.

[cite]: https://guides.rubyonrails.org/action_controller_overview.html
[routers]: routers.md
