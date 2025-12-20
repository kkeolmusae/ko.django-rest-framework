# Tutorial 6: ViewSets 및 Routers

REST framework에는 **ViewSet**이라는 추상화가 있습니다.  
이를 사용하면 개발자는 API의 상태(state)와 상호작용(interaction) 설계에 집중하고,  
URL 구성(URL conf)은 일반적인 규칙(convention)에 따라 자동으로 처리할 수 있습니다.

`ViewSet` 클래스는 일반 `View` 클래스와 거의 비슷하지만,  
`get`이나 `put` 같은 HTTP 메서드 핸들러를 직접 정의하지 않고,  
`retrieve`, `update` 등 CRUD와 관련된 동작을 제공합니다.

`ViewSet` 클래스는 최종적으로 인스턴스화 되어 실제 뷰(view)로 바인딩될 때,  
Router를 사용하여 URL 구성을 자동으로 처리합니다.

---

## ViewSet으로 리팩토링

기존의 `UserList`와 `UserDetail`을 하나의 ViewSet으로 합칩니다.

```python
from rest_framework import viewsets


class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    이 ViewSet은 자동으로 `list`와 `retrieve` 동작을 제공합니다.
    """

    queryset = User.objects.all()
    serializer_class = UserSerializer
```

- `ReadOnlyModelViewSet`을 사용하면 읽기 전용 동작만 제공 (`list`/`retrieve`)
- `queryset`과 `serializer_class`는 기존 뷰에서 지정하던 그대로 사용

---

다음으로 `SnippetList`, `SnippetDetail`, `SnippetHighlight`를 하나의 `SnippetViewSet`으로 통합합니다.

```python
from rest_framework import permissions, renderers
from rest_framework.decorators import action
from rest_framework.response import Response


class SnippetViewSet(viewsets.ModelViewSet):
    """
    이 ViewSet은 `list`, `create`, `retrieve`, `update`, `destroy` 동작을 자동 제공.

    추가로 `highlight` 커스텀 액션도 포함.
    """

    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

- `ModelViewSet`을 사용하면 기본 CRUD 동작을 모두 제공
- `@action` 데코레이터로 `highlight` 커스텀 엔드포인트 생성
  - 기본적으로 `GET` 요청 대응
  - 필요하면 `methods=['POST']` 등으로 변경 가능
  - `url_path`로 URL 패턴 이름 변경 가능

---

## ViewSet을 URL에 명시적으로 바인딩

ViewSet은 URLConf에 바인딩될 때 비로소 HTTP 메서드와 액션이 연결됩니다.

```python
from rest_framework import renderers
from snippets.views import SnippetViewSet, UserViewSet, api_root

snippet_list = SnippetViewSet.as_view({"get": "list", "post": "create"})
snippet_detail = SnippetViewSet.as_view({
    "get": "retrieve",
    "put": "update",
    "patch": "partial_update",
    "delete": "destroy"
})
snippet_highlight = SnippetViewSet.as_view(
    {"get": "highlight"}, renderer_classes=[renderers.StaticHTMLRenderer]
)
user_list = UserViewSet.as_view({"get": "list"})
user_detail = UserViewSet.as_view({"get": "retrieve"})
```

- 각 ViewSet에서 여러 뷰를 생성하며, HTTP 메서드를 적절한 액션과 연결

```python
urlpatterns = format_suffix_patterns([
    path("", api_root),
    path("snippets/", snippet_list, name="snippet-list"),
    path("snippets/<int:pk>/", snippet_detail, name="snippet-detail"),
    path("snippets/<int:pk>/highlight/", snippet_highlight, name="snippet-highlight"),
    path("users/", user_list, name="user-list"),
    path("users/<int:pk>/", user_detail, name="user-detail"),
])
```

---

## Routers 사용

ViewSet을 사용하면 URLConf를 직접 설계할 필요가 없습니다.  
Router가 ViewSet을 등록하고, URL 패턴과 엔드포인트를 자동으로 구성합니다.

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from snippets import views

# Router 생성 및 ViewSet 등록
router = DefaultRouter()
router.register(r"snippets", views.SnippetViewSet, basename="snippet")
router.register(r"users", views.UserViewSet, basename="user")

# Router가 URL을 자동 생성
urlpatterns = [
    path("", include(router.urls)),
]
```

- `DefaultRouter` 사용 시 API 루트도 자동 생성
- 따라서 기존의 `api_root` 함수는 삭제 가능

---

## View vs ViewSet 선택 시 고려 사항

- ViewSet 사용 장점:

  - URL 규칙이 일관되게 적용됨
  - 작성해야 하는 코드 양 최소화
  - API 상호작용 및 표현(representation)에 집중 가능

- 단점 / 주의사항:
  - 개별 뷰를 명시적으로 작성하는 것보다 덜 직관적
  - 일부 커스텀 동작이나 예외 처리에는 추가 설정 필요
