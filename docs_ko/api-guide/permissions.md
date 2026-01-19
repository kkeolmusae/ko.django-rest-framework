---
source:
    - permissions.py
---

# 권한 (Permissions)

> 인증(authentication)이나 식별(identification)만으로는 일반적으로 정보나 코드에 접근하기에 충분하지 않습니다. 이를 위해서는 접근을 요청하는 주체가 권한 부여(authorization)를 받아야 합니다.
>
> &mdash; [Apple Developer Documentation][cite]

[authentication] 및 [throttling]과 함께, permissions는 요청에 대해 접근을 허용할지 거부할지를 결정합니다.

권한 체크는 항상 뷰의 가장 시작 지점에서 실행되며, 다른 어떤 코드도 진행되기 전에 수행됩니다. 권한 체크는 일반적으로 `request.user` 및 `request.auth` 속성에 있는 인증 정보를 사용하여 들어오는 요청을 허용할지 판단합니다.

Permissions는 서로 다른 유형의 사용자에게 API의 서로 다른 부분에 대한 접근을 허용하거나 거부하기 위해 사용됩니다.

가장 단순한 permission 스타일은 인증된 사용자에게는 접근을 허용하고, 인증되지 않은 사용자에게는 접근을 거부하는 것입니다. 이는 REST framework의 `IsAuthenticated` 클래스에 해당합니다.

조금 덜 엄격한 스타일로는, 인증된 사용자에게는 전체 접근을 허용하되 인증되지 않은 사용자에게는 읽기 전용 접근만 허용하는 방식이 있습니다. 이는 REST framework의 `IsAuthenticatedOrReadOnly` 클래스에 해당합니다.

## 권한이 결정되는 방식 (How permissions are determined)

REST framework의 permission은 항상 permission 클래스들의 리스트로 정의됩니다.

뷰의 본문이 실행되기 전에 리스트의 각 permission이 검사됩니다.
어떤 permission 검사라도 실패하면 `exceptions.PermissionDenied` 또는 `exceptions.NotAuthenticated` 예외가 발생하고, 뷰 본문은 실행되지 않습니다.

권한 체크가 실패했을 때는 다음 규칙에 따라 "403 Forbidden" 또는 "401 Unauthorized" 응답이 반환됩니다.

* 요청은 성공적으로 인증되었지만, 권한이 거부된 경우. *&mdash; HTTP 403 Forbidden 응답이 반환됩니다.*
* 요청이 성공적으로 인증되지 않았고, 우선순위가 가장 높은 인증 클래스가 `WWW-Authenticate` 헤더를 **사용하지 않는** 경우. *&mdash; HTTP 403 Forbidden 응답이 반환됩니다.*
* 요청이 성공적으로 인증되지 않았고, 우선순위가 가장 높은 인증 클래스가 `WWW-Authenticate` 헤더를 **사용하는** 경우. *&mdash; 적절한 `WWW-Authenticate` 헤더가 포함된 HTTP 401 Unauthorized 응답이 반환됩니다.*

## 객체 수준 권한 (Object level permissions)

REST framework permission은 객체 수준(object-level) 권한도 지원합니다. 객체 수준 권한은 사용자가 특정 객체(보통 모델 인스턴스)에 대해 동작할 수 있는지 여부를 결정하는 데 사용됩니다.

객체 수준 권한은 `.get_object()`가 호출될 때 REST framework의 generic view에 의해 실행됩니다.
뷰 수준 권한과 마찬가지로, 사용자가 주어진 객체에 대해 동작할 수 없다면 `exceptions.PermissionDenied` 예외가 발생합니다.

직접 뷰를 작성하면서 객체 수준 권한을 강제하고 싶거나,
generic view의 `get_object` 메서드를 오버라이드하는 경우에는,
객체를 가져온 시점에 뷰에서 `.check_object_permissions(request, obj)`를 명시적으로 호출해야 합니다.

이 호출은 `PermissionDenied` 또는 `NotAuthenticated` 예외를 발생시키거나, 권한이 적절하다면 아무 일 없이 반환합니다.

예:

    def get_object(self):
        obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
        self.check_object_permissions(self.request, obj)
        return obj

---

**참고**: `DjangoObjectPermissions`를 제외하면,
`rest_framework.permissions`에 제공되는 permission 클래스들은 객체 권한을 체크하는 데 필요한 메서드들을 **구현하지 않습니다**.

제공되는 permission 클래스를 사용해서 객체 권한을 체크하려면,
아래 [*Custom permissions*](#custom-permissions) 섹션에서 설명하는
`has_object_permission()` 메서드를 구현하도록 해당 클래스를 서브클래싱해야 **합니다**.

---

#### 객체 수준 권한의 한계 (Limitations of object level permissions)

성능상의 이유로 generic view는 객체 목록을 반환할 때 queryset의 각 인스턴스에 대해 객체 수준 권한을 자동으로 적용하지 않습니다.

객체 수준 권한을 사용하는 경우, 종종 사용자가 볼 수 있어야 하는 인스턴스만 노출되도록 queryset을 적절히 [필터링][filtering]하는 것도 함께 필요합니다.

또한 `get_object()` 메서드가 호출되지 않기 때문에, 객체 생성 시에는 `has_object_permission()` 메서드의 객체 수준 권한이 **적용되지 않습니다**. 객체 생성을 제한하려면 Serializer 클래스에서 권한 체크를 구현하거나, ViewSet의 `perform_create()` 메서드를 오버라이드하여 구현해야 합니다.

## 권한 정책 설정 (Setting the permission policy)

기본 permission 정책은 `DEFAULT_PERMISSION_CLASSES` 설정으로 전역 지정할 수 있습니다. 예:

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.IsAuthenticated',
        ]
    }

이 설정을 지정하지 않으면, 기본값은 제한 없는 접근을 허용합니다.

    'DEFAULT_PERMISSION_CLASSES': [
       'rest_framework.permissions.AllowAny',
    ]

또한 `APIView` 기반 클래스 기반 뷰를 사용해 뷰(또는 뷰셋) 단위로 permission 정책을 설정할 수도 있습니다.

    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        permission_classes = [IsAuthenticated]

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

또는 함수 기반 뷰에서 `@api_view` 데코레이터를 사용하는 경우:

    from rest_framework.decorators import api_view, permission_classes
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response

    @api_view(['GET'])
    @permission_classes([IsAuthenticated])
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

**참고:** 클래스 속성이나 데코레이터로 새 permission 클래스를 설정하면, 뷰는 __settings.py__에 설정된 기본 리스트를 무시하게 됩니다.

`rest_framework.permissions.BasePermission`을 상속하는 한, permissions는 표준 Python 비트 연산자를 사용해 조합할 수 있습니다. 예를 들어, `IsAuthenticatedOrReadOnly`는 다음처럼 작성할 수도 있습니다:

    from rest_framework.permissions import BasePermission, IsAuthenticated, SAFE_METHODS
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ReadOnly(BasePermission):
        def has_permission(self, request, view):
            return request.method in SAFE_METHODS

    class ExampleView(APIView):
        permission_classes = [IsAuthenticated|ReadOnly]

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

**참고:** `&`(and), `|`(or), `~`(not)을 지원합니다.

---

# API 레퍼런스 (API Reference)

## AllowAny

`AllowAny` permission 클래스는 요청이 인증되었는지 여부와 **상관없이** 제한 없는 접근을 허용합니다.

이 permission은 반드시 필요하진 않습니다. permissions 설정을 빈 리스트나 튜플로 두는 것만으로도 같은 결과를 얻을 수 있습니다. 하지만 의도를 명확히 드러내기 위해 이 클래스를 지정하는 것이 유용할 수 있습니다.

## IsAuthenticated

`IsAuthenticated` permission 클래스는 인증되지 않은 사용자에 대해서는 접근을 거부하고, 그 외에는 접근을 허용합니다.

API를 등록된 사용자만 접근 가능하도록 만들고 싶을 때 적합합니다.

## IsAdminUser

`IsAdminUser` permission 클래스는 `user.is_staff`가 `True`인 경우에만 접근을 허용하고, 그렇지 않으면 접근을 거부합니다.

신뢰할 수 있는 관리자 일부에게만 API 접근을 허용하려는 경우에 적합합니다.

## IsAuthenticatedOrReadOnly

`IsAuthenticatedOrReadOnly`는 인증된 사용자가 어떤 요청이든 수행할 수 있도록 허용합니다. 인증되지 않은 사용자는 요청 메서드가 “safe” 메서드(`GET`, `HEAD`, `OPTIONS`) 중 하나인 경우에만 허용됩니다.

익명 사용자에게는 읽기 권한만 허용하고, 쓰기 권한은 인증된 사용자에게만 허용하려는 경우에 적합합니다.

## DjangoModelPermissions

이 permission 클래스는 Django의 표준 `django.contrib.auth` [모델 권한][contribauth]과 연동됩니다. 이 permission은 `.queryset` 속성 또는 `get_queryset()` 메서드를 가진 뷰에만 적용해야 합니다. 사용자가 *인증되어 있고* 해당 모델에 대해 *관련 모델 권한*이 부여된 경우에만 접근이 허용됩니다. 적절한 모델은 `get_queryset().model` 또는 `queryset.model`을 확인하여 결정됩니다.

* `POST` 요청은 모델에 대한 `add` 권한이 필요합니다.
* `PUT` 및 `PATCH` 요청은 모델에 대한 `change` 권한이 필요합니다.
* `DELETE` 요청은 모델에 대한 `delete` 권한이 필요합니다.

기본 동작은 커스텀 모델 권한을 지원하도록 오버라이드할 수도 있습니다. 예를 들어 `GET` 요청에 대해 `view` 모델 권한을 포함하고 싶을 수 있습니다.

커스텀 모델 권한을 사용하려면 `DjangoModelPermissions`를 오버라이드하고 `.perms_map` 속성을 설정하세요. 자세한 내용은 소스 코드를 참고하세요.

## DjangoModelPermissionsOrAnonReadOnly

`DjangoModelPermissions`와 유사하지만, 인증되지 않은 사용자에게도 읽기 전용 접근을 허용합니다.

## DjangoObjectPermissions

이 permission 클래스는 모델에 대해 객체별(per-object) 권한을 허용하는 Django의 표준 [객체 권한 프레임워크][objectpermissions]와 연동됩니다. 이를 사용하려면 [django-guardian][guardian] 같은 객체 수준 권한을 지원하는 permission backend를 추가로 설정해야 합니다.

`DjangoModelPermissions`와 마찬가지로, 이 permission도 `.queryset` 속성 또는 `.get_queryset()` 메서드가 있는 뷰에만 적용해야 합니다. 사용자가 *인증되어 있고* *관련 객체별 권한*과 *관련 모델 권한*이 부여된 경우에만 접근이 허용됩니다.

* `POST` 요청은 해당 모델 인스턴스에 대한 `add` 권한이 필요합니다.
* `PUT` 및 `PATCH` 요청은 해당 모델 인스턴스에 대한 `change` 권한이 필요합니다.
* `DELETE` 요청은 해당 모델 인스턴스에 대한 `delete` 권한이 필요합니다.

`DjangoObjectPermissions`는 `django-guardian` 패키지를 **필수로 요구하지 않으며**, 다른 객체 수준 백엔드도 동일하게 지원해야 합니다.

`DjangoModelPermissions`와 마찬가지로, `DjangoObjectPermissions`를 오버라이드하고 `.perms_map` 속성을 설정하여 커스텀 모델 권한을 사용할 수 있습니다. 자세한 내용은 소스 코드를 참고하세요.

---

**참고**: `GET`, `HEAD`, `OPTIONS` 요청에 대해 객체 수준 `view` 권한이 필요하고, 객체 수준 권한 백엔드로 django-guardian을 사용 중이라면 [`djangorestframework-guardian` 패키지][django-rest-framework-guardian]가 제공하는 `DjangoObjectPermissionsFilter` 클래스를 고려해보세요. 이 클래스는 list 엔드포인트가 “사용자가 적절한 view 권한을 가진 객체”만 포함해 반환하도록 보장합니다.

---

# 커스텀 권한 (Custom permissions)

커스텀 permission을 구현하려면 `BasePermission`을 오버라이드하고 다음 메서드 중 하나 또는 둘 다를 구현합니다.

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

이 메서드들은 요청에 접근을 허용해야 하면 `True`, 그렇지 않으면 `False`를 반환해야 합니다.

요청이 읽기인지 쓰기인지 테스트해야 한다면, 요청 메서드를 `SAFE_METHODS` 상수와 비교하세요. `SAFE_METHODS`는 `'GET'`, `'OPTIONS'`, `'HEAD'`를 포함한 튜플입니다. 예:

    if request.method in permissions.SAFE_METHODS:
        # 읽기 전용 요청에 대한 권한 체크
    else:
        # 쓰기 요청에 대한 권한 체크

---

**참고**: 인스턴스 수준의 `has_object_permission` 메서드는 뷰 수준의 `has_permission` 체크가 이미 통과한 경우에만 호출됩니다. 또한 인스턴스 수준 체크가 실행되려면, 뷰 코드에서 명시적으로 `.check_object_permissions(request, obj)`를 호출해야 합니다. generic view를 사용한다면 기본적으로 처리됩니다. (함수 기반 뷰는 객체 권한을 명시적으로 체크하고, 실패 시 `PermissionDenied`를 발생시켜야 합니다.)

---

커스텀 permission은 테스트가 실패하면 `PermissionDenied` 예외를 발생시킵니다. 예외에 연결되는 에러 메시지를 변경하려면, 커스텀 permission에 `message` 속성을 직접 구현하세요. 그렇지 않으면 `PermissionDenied`의 `default_detail` 속성이 사용됩니다. 마찬가지로 코드 식별자를 변경하려면 `code` 속성을 직접 구현하세요. 그렇지 않으면 `PermissionDenied`의 `default_code`가 사용됩니다.

    from rest_framework import permissions

    class CustomerAccessPermission(permissions.BasePermission):
        message = 'Adding customers not allowed.'

        def has_permission(self, request, view):
             ...

## Examples

다음은 들어오는 요청의 IP 주소를 차단 목록(blocklist)과 비교하여, 차단된 IP라면 요청을 거부하는 permission 클래스 예시입니다.

    from rest_framework import permissions

    class BlocklistPermission(permissions.BasePermission):
        """
        차단된 IP에 대한 전역 permission 체크.
        """

        def has_permission(self, request, view):
            ip_addr = request.META['REMOTE_ADDR']
            blocked = Blocklist.objects.filter(ip_addr=ip_addr).exists()
            return not blocked

전역 권한(모든 요청에 대해 실행되는 권한) 외에도, 특정 객체 인스턴스에 영향을 주는 동작에만 적용되는 객체 수준 권한도 만들 수 있습니다. 예:

    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        객체의 소유자만 수정할 수 있도록 하는 객체 수준 권한.
        모델 인스턴스에 `owner` 속성이 있다고 가정.
        """

        def has_object_permission(self, request, view, obj):
            # 읽기 권한은 모든 요청에 대해 허용하므로
            # GET, HEAD, OPTIONS 요청은 항상 허용합니다.
            if request.method in permissions.SAFE_METHODS:
                return True

            # 인스턴스에 `owner`라는 속성이 있어야 합니다.
            return obj.owner == request.user

generic view는 적절한 객체 수준 권한을 체크하지만, 커스텀 뷰를 작성한다면 객체 수준 권한 체크를 직접 수행해야 합니다. 객체 인스턴스를 얻은 뒤 `self.check_object_permissions(request, obj)`를 호출하면 됩니다. 이 호출은 객체 수준 권한 체크가 실패하면 적절한 `APIException`을 발생시키고, 성공하면 그냥 반환합니다.

또한 generic view는 단일 모델 인스턴스를 조회(retrieve)하는 뷰에 대해서만 객체 수준 권한을 체크합니다. list 뷰에서 객체 수준 필터링이 필요하다면 queryset을 별도로 필터링해야 합니다. 자세한 내용은 [filtering 문서][filtering]를 참고하세요.

# 접근 제한 방법 개요 (Overview of access restriction methods)

REST framework는 케이스별로 접근 제한을 커스터마이징할 수 있는 세 가지 방법을 제공합니다. 이들은 서로 다른 시나리오에 적용되며, 효과와 한계도 다릅니다.

* `queryset`/`get_queryset()`: 데이터베이스의 기존 객체에 대한 “가시성”을 제한합니다. queryset은 어떤 객체가 리스트에 노출되는지, 어떤 객체를 수정/삭제할 수 있는지를 제한합니다. `get_queryset()`은 현재 액션에 따라 다른 queryset을 적용할 수 있습니다.
* `permission_classes`/`get_permissions()`: 현재 액션, 요청, 대상 객체를 기반으로 하는 일반적인 permission 체크입니다. 객체 수준 권한은 retrieve/modify/deletion 액션에만 적용할 수 있습니다. list 및 create에 대한 permission 체크는 전체 객체 타입에 대해 적용됩니다. (list의 경우: queryset의 제한을 따릅니다.)
* `serializer_class`/`get_serializer()`: 입력/출력에서 모든 객체에 적용되는 인스턴스 수준 제한입니다. serializer는 request context에 접근할 수 있습니다. `get_serializer()`는 현재 액션에 따라 다른 serializer를 적용할 수 있습니다.

다음 표는 접근 제한 방법과, 액션별로 어떤 수준의 제어가 가능한지를 보여줍니다.

|                                    | `queryset` | `permission_classes` | `serializer_class` |
|------------------------------------|------------|----------------------|--------------------|
| Action: list                       | global     | global               | object-level*      |
| Action: create                     | no         | global               | object-level       |
| Action: retrieve                   | global     | object-level         | object-level       |
| Action: update                     | global     | object-level         | object-level       |
| Action: partial_update             | global     | object-level         | object-level       |
| Action: destroy                    | global     | object-level         | no                 |
| Can reference action in decision   | no**       | yes                  | no**               |
| Can reference request in decision  | no**       | yes                  | yes                |

 \* Serializer 클래스가 list 액션에서 PermissionDenied를 발생시키면 전체 리스트가 반환되지 않게 되므로, list에서 PermissionDenied를 발생시키면 안 됩니다. <br>
 \** `get_*()` 메서드들은 현재 뷰에 접근할 수 있으며, 요청 또는 액션에 따라 다른 Serializer 또는 QuerySet 인스턴스를 반환할 수 있습니다.

---

# 서드파티 패키지 (Third party packages)

다음과 같은 서드파티 패키지도 사용할 수 있습니다.

## DRF - Access Policy

[Django REST - Access Policy][drf-access-policy] 패키지는 뷰셋 또는 함수 기반 뷰에 부착하는 선언적 정책 클래스에서 복잡한 접근 규칙을 정의할 수 있는 방법을 제공합니다. 정책은 AWS IAM 정책과 유사한 형식의 JSON으로 정의됩니다.

## Composed Permissions

[Composed Permissions][composed-permissions] 패키지는 작고 재사용 가능한 컴포넌트를 사용해(논리 연산자로) 복잡하고 다단계의 permission 객체를 정의하는 간단한 방법을 제공합니다.

## REST Condition

[REST Condition][rest-condition] 패키지는 복잡한 permissions를 간단하고 편리하게 구성하기 위한 또 다른 확장입니다. 논리 연산자로 permissions를 결합할 수 있습니다.

## DRY Rest Permissions

[DRY Rest Permissions][dry-rest-permissions] 패키지는 기본 및 커스텀 액션 각각에 대해 다른 permissions를 정의할 수 있게 해줍니다. 이 패키지는 앱 데이터 모델에 정의된 관계에서 파생되는 permissions에 적합합니다. 또한 serializer를 통해 클라이언트 앱에 permission 체크 결과를 반환하는 것도 지원합니다. 더불어 사용자별로 조회 데이터를 제한하기 위해 기본/커스텀 list 액션에 permissions를 추가하는 것도 지원합니다.

## Django Rest Framework Roles

[Django Rest Framework Roles][django-rest-framework-roles] 패키지는 여러 유형의 사용자에 대해 API를 파라미터화하기 쉽게 만들어줍니다.

## Rest Framework Roles

[Rest Framework Roles][rest-framework-roles]는 role 기반으로 뷰를 보호하는 것을 매우 쉽게 해줍니다. 특히 접근 로직을 모델과 뷰에서 깔끔하고 사람이 읽기 쉬운 방식으로 분리할 수 있게 해줍니다.

## Django REST Framework API Key

[Django REST Framework API Key][djangorestframework-api-key] 패키지는 API key 기반 권한 부여를 추가하기 위한 permission 클래스, 모델, 헬퍼를 제공합니다. 사용자 계정이 없는 내부/외부 백엔드 및 서비스(즉, *machines*)를 인증하는 데 사용할 수 있습니다. API 키는 Django의 비밀번호 해싱 인프라를 사용해 안전하게 저장되며, Django admin에서 언제든 조회/편집/폐기가 가능합니다.

## Django Rest Framework Role Filters

[Django Rest Framework Role Filters][django-rest-framework-role-filters] 패키지는 여러 역할 타입에 대한 간단한 필터링을 제공합니다.

## Django Rest Framework PSQ

[Django Rest Framework PSQ][drf-psq] 패키지는 permission 기반 규칙에 따라 액션 기반 **permission_classes**, **serializer_class**, **queryset**을 지원하도록 확장해줍니다.

## Axioms DRF PY

[Axioms DRF PY][axioms-drf-py] 패키지는 OAuth2/OIDC Authorization Server(AWS Cognito, Auth0, Okta, Microsoft Entra 등)가 발급한 JWT 토큰을 사용하여 인증과 클레임 기반의 세밀한 권한 부여(**scopes**, **roles**, **groups**, **permissions** 등 객체 수준 체크 포함)를 지원합니다.

[cite]: https://developer.apple.com/library/mac/#documentation/security/Conceptual/AuthenticationAndAuthorizationGuide/Authorization/Authorization.html
[authentication]: authentication.md
[throttling]: throttling.md
[filtering]: filtering.md
[contribauth]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#custom-permissions
[objectpermissions]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#handling-object-permissions
[guardian]: https://github.com/lukaszb/django-guardian
[composed-permissions]: https://github.com/niwibe/djangorestframework-composed-permissions
[rest-condition]: https://github.com/caxap/rest_condition
[dry-rest-permissions]: https://github.com/FJNR-inc/dry-rest-permissions
[django-rest-framework-roles]: https://github.com/computer-lab/django-rest-framework-roles
[rest-framework-roles]: https://github.com/Pithikos/rest-framework-roles
[djangorestframework-api-key]: https://florimondmanca.github.io/djangorestframework-api-key/
[django-rest-framework-role-filters]: https://github.com/allisson/django-rest-framework-role-filters
[django-rest-framework-guardian]: https://github.com/rpkilby/django-rest-framework-guardian
[drf-access-policy]: https://github.com/rsinger86/drf-access-policy
[drf-psq]: https://github.com/drf-psq/drf-psq
[axioms-drf-py]: https://github.com/abhishektiwari/axioms-drf-py
