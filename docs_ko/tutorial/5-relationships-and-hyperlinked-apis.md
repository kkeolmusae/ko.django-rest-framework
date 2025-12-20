# Tutorial 5: 관계(Relationships) 및 하이퍼링크 API

현재 우리 API에서 엔티티 간 관계는 기본 키(primary key)를 사용하여 표현되고 있습니다.  
이번 파트에서는 관계를 하이퍼링크 방식으로 변경하여 API의 응집력과 발견성을 개선합니다.

## API 루트 엔드포인트 생성

지금은 `'snippets'`와 `'users'` 엔드포인트만 있고,  
API의 단일 진입점(entry point)은 없습니다.

`snippets/views.py`에 함수 기반 뷰를 추가하여 API 루트를 만듭니다.

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(["GET"])
def api_root(request, format=None):
    return Response(
        {
            "users": reverse("user-list", request=request, format=format),
            "snippets": reverse("snippet-list", request=request, format=format),
        }
    )
```

포인트:

1. `reverse` 함수를 사용하여 절대 URL을 반환
2. URL 패턴은 이름(name)을 지정하여 참조

## 하이라이트된 스니펫 엔드포인트 생성

우리 Pastebin API에서 빠진 또 다른 기능은 **코드 하이라이트**입니다.

- JSON이 아닌 **HTML 표현**을 반환
- REST framework에는 두 가지 HTML 렌더러가 있음
  1. 템플릿으로 렌더링된 HTML 처리
  2. 이미 렌더링된 HTML 처리 → 이번에 사용

객체 인스턴스를 그대로 반환하는 것이 아니라 **객체 속성**을 반환하기 때문에  
기존의 구체적(Generic) 뷰는 사용하지 않고,  
`GenericAPIView`를 상속하고 `.get()` 메서드를 구현합니다.

```python
from rest_framework import renderers

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = [renderers.StaticHTMLRenderer]

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```

URL 패턴에 새 view를 추가합니다.

```python
# API 루트
path("", views.api_root),

# 스니펫 하이라이트
path("snippets/<int:pk>/highlight/", views.SnippetHighlight.as_view()),
```

## 하이퍼링크 기반 관계(Hyperlinked API)

엔티티 간 관계 표현 방법:

- 기본 키(primary key) 사용
- 하이퍼링크 사용
- 관련 엔티티의 고유 슬러그 사용
- 관련 엔티티의 문자열 표현 사용
- 부모 안에 관련 엔티티 중첩(Nesting)
- 기타 커스텀 표현

REST framework는 모든 방식을 지원하며,  
Forward/Reverse 관계 또는 Generic Foreign Key에도 적용 가능.

이번 튜토리얼에서는 **하이퍼링크** 방식 사용.

- Serializer를 `ModelSerializer` → `HyperlinkedModelSerializer`로 변경

`HyperlinkedModelSerializer` 특징:

- 기본적으로 `id` 필드 포함 안 함
- `url` 필드 포함 (`HyperlinkedIdentityField`)
- 관계는 `HyperlinkedRelatedField` 사용 (기존 `PrimaryKeyRelatedField` 대신)

`snippets/serializers.py` 수정:

```python
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source="owner.username")
    highlight = serializers.HyperlinkedIdentityField(
        view_name="snippet-highlight", format="html"
    )

    class Meta:
        model = Snippet
        fields = [
            "url",
            "id",
            "highlight",
            "owner",
            "title",
            "code",
            "linenos",
            "language",
            "style",
        ]


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(
        many=True, view_name="snippet-detail", read_only=True
    )

    class Meta:
        model = User
        fields = ["url", "id", "username", "snippets"]
```

포인트:

- 새로운 `'highlight'` 필드 추가 (`url` 필드와 유사하지만 `'snippet-highlight'` URL 참조)
- format-suffixed URL(`.json`)을 사용하므로, `'highlight'` 필드는 `.html` 형식 지정 필요

---

**주의:**

`SnippetDetail`이나 `SnippetList`처럼 view 내부에서 serializer를 직접 생성할 경우,  
절대 URL을 생성하기 위해 반드시 `context={'request': request}`를 전달해야 합니다.

```python
serializer = SnippetSerializer(snippet, context={"request": request})
```

`GenericAPIView` 하위 클래스라면 `get_serializer_context()`를 편의 메서드로 사용할 수 있습니다.

---

## URL 패턴에 이름 지정

하이퍼링크 API를 사용하려면 URL 패턴 이름 지정 필수:

- API 루트 → `'user-list'`, `'snippet-list'`
- Snippet Serializer → `'snippet-highlight'`
- User Serializer → `'snippet-detail'`
- `url` 필드 → 기본적으로 `'{model_name}-detail'` (`'snippet-detail'`, `'user-detail'`)

최종 `snippets/urls.py`:

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns(
    [
        path("", views.api_root),
        path("snippets/", views.SnippetList.as_view(), name="snippet-list"),
        path(
            "snippets/<int:pk>/", views.SnippetDetail.as_view(), name="snippet-detail"
        ),
        path(
            "snippets/<int:pk>/highlight/",
            views.SnippetHighlight.as_view(),
            name="snippet-highlight",
        ),
        path("users/", views.UserList.as_view(), name="user-list"),
        path("users/<int:pk>/", views.UserDetail.as_view(), name="user-detail"),
    ]
)
```

## 페이지네이션(Pagination) 추가

스니펫 및 사용자 리스트는 많은 항목을 반환할 수 있으므로,  
페이지네이션을 적용하고 클라이언트가 페이지 단위로 조회하도록 설정.

`tutorial/settings.py`에 추가:

```python
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 10,
}
```

REST framework 설정은 `REST_FRAMEWORK` 딕셔너리 안에 모아 관리.

---

## API 브라우징

브라우저에서 API에 접속하면,  
하이퍼링크를 따라 쉽게 API를 탐색 가능.

스니펫 인스턴스에 `'highlight'` 링크도 표시되어  
하이라이트된 HTML 코드를 바로 확인할 수 있습니다.

[tutorial part 6][tut-6]에서는 ViewSet과 Router를 사용해  
API 코드를 더욱 간결하게 작성하는 방법을 다룹니다.

[tut-6]: 6-viewsets-and-routers.md
