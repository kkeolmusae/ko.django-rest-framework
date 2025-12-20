# Tutorial 4: 인증(Authentication) 및 권한(Permissions)

현재 우리 API는 코드 스니펫을 누가 수정하거나 삭제할 수 있는지에 대한 제한이 없습니다.  
좀 더 정교한 동작을 추가하여 다음을 보장하고자 합니다.

- 코드 스니펫은 항상 작성자(owner)와 연관되어야 합니다.
- 인증된 사용자만 스니펫을 생성할 수 있습니다.
- 스니펫을 생성한 사용자만 해당 스니펫을 수정하거나 삭제할 수 있습니다.
- 인증되지 않은 요청은 전체 읽기 전용 접근을 가집니다.

## 모델에 정보 추가하기

`Snippet` 모델 클래스에 몇 가지 변경을 합니다.

먼저 두 개의 필드를 추가합니다.  
하나는 스니펫을 생성한 사용자를 나타내고,  
다른 하나는 하이라이트된 HTML 코드 표현을 저장하는 필드입니다.

`models.py`의 `Snippet` 모델에 다음 두 필드를 추가합니다.

```python
owner = models.ForeignKey(
    "auth.User", related_name="snippets", on_delete=models.CASCADE
)
highlighted = models.TextField()
```

또한 모델이 저장될 때 `highlighted` 필드를 자동으로 채우도록, `pygments` 라이브러리를 사용해 하이라이트된 HTML을 생성하도록 해야 합니다.

필요한 추가 import:

```python
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```

그리고 모델 클래스에 `.save()` 메서드를 추가합니다.

```python
def save(self, *args, **kwargs):
    """
    `pygments` 라이브러리를 사용하여 코드 스니펫의 하이라이트 HTML 표현을 생성합니다.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = "table" if self.linenos else False
    options = {"title": self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos, full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super().save(*args, **kwargs)
```

모든 작업이 끝나면 데이터베이스 테이블을 업데이트해야 합니다.  
튜토리얼 목적상, 간단히 데이터베이스를 삭제하고 다시 시작하겠습니다.

```bash
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```

테스트용으로 여러 사용자를 만들어두는 것도 좋습니다.  
`createsuperuser` 명령을 사용하면 가장 빠르게 생성할 수 있습니다.

```bash
python manage.py createsuperuser
```

## User 모델 엔드포인트 추가

이제 API에서 사용자 정보를 제공하려면 serializer를 생성해야 합니다.  
`serializers.py`에 다음 내용을 추가합니다.

```python
from django.contrib.auth.models import User


class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(
        many=True, queryset=Snippet.objects.all()
    )

    class Meta:
        model = User
        fields = ["id", "username", "snippets"]
```

`'snippets'`는 User 모델의 **역방향 관계(reverse relation)** 이므로  
기본적으로 `ModelSerializer`에 포함되지 않습니다.  
따라서 명시적으로 필드를 추가해야 합니다.

이제 `views.py`에 사용자 관련 view를 추가합니다.  
사용자 조회는 읽기 전용이므로 `ListAPIView`와 `RetrieveAPIView`를 사용합니다.

```python
from django.contrib.auth.models import User


class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

`UserSerializer`도 import 해줍니다.

```python
from snippets.serializers import UserSerializer
```

마지막으로 URL conf에 view를 추가합니다. `snippets/urls.py`에 다음을 추가합니다.

```python
path("users/", views.UserList.as_view()),
path("users/<int:pk>/", views.UserDetail.as_view()),
```

## 스니펫과 사용자 연결

지금까지는 스니펫 생성 시 해당 스니펫을 생성한 사용자와 연결되지 않았습니다.  
사용자 정보는 직렬화 데이터에 포함되지 않고, 요청(request) 속성으로 들어오기 때문입니다.

이 문제는 view의 `.perform_create()` 메서드를 오버라이드하여 해결할 수 있습니다.  
`SnippetList` view 클래스에 다음을 추가합니다.

```python
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```

이제 serializer의 `create()` 메서드에는 요청으로부터 검증된 데이터와 함께 `'owner'` 필드가 전달됩니다.

## Serializer 업데이트

스니펫과 사용자가 연결되었으므로, `SnippetSerializer`에도 해당 필드를 추가합니다.

```python
owner = serializers.ReadOnlyField(source="owner.username")
```

※ `Meta` 클래스의 `fields` 목록에도 `'owner',`를 추가해야 합니다.

- `source` 인자는 어떤 속성에서 데이터를 가져올지 지정합니다.
- 점(dot) 표기법을 사용할 수 있으며, Django 템플릿 언어처럼 속성을 탐색합니다.
- `ReadOnlyField`는 읽기 전용 필드로, 직렬화된 데이터에는 포함되지만  
  역직렬화 시 모델 업데이트에는 사용되지 않습니다.
- 필요하면 `CharField(read_only=True)`로 대체할 수도 있습니다.

## View 권한 설정

이제 스니펫이 사용자와 연결되었으므로, 인증된 사용자만 CRUD 작업을 할 수 있도록 권한을 설정합니다.

REST framework는 여러 권한 클래스(permission class)를 제공합니다.  
이번 경우에는 `IsAuthenticatedOrReadOnly`를 사용하면 됩니다.

- 인증된 요청: 읽기/쓰기 가능
- 인증되지 않은 요청: 읽기 전용

`views.py`에 import:

```python
from rest_framework import permissions
```

그리고 `SnippetList`와 `SnippetDetail` view 클래스에 다음을 추가합니다.

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

## 브라우저 API 로그인 추가

현재 브라우저에서 접근하면 새 스니펫을 생성할 수 없습니다.  
로그인을 추가해야 합니다.

프로젝트 레벨 `urls.py`에서 URL conf를 수정합니다.

파일 상단에 import 추가:

```python
from django.urls import path, include
```

파일 하단에 login/logout view 패턴 추가:

```python
urlpatterns += [
    path("api-auth/", include("rest_framework.urls")),
]
```

`'api-auth/'`는 원하는 URL로 변경 가능하며, 브라우저에서 새로고침하면  
우측 상단에 'Login' 링크가 나타납니다. 로그인 후 스니펫 생성 가능.

스니펫을 몇 개 생성한 뒤, `/users/` endpoint를 확인하면  
각 사용자별 `snippets` 필드에 생성한 스니펫 ID가 표시됩니다.

## 객체 수준 권한(Object-level permissions)

모든 사용자가 스니펫을 볼 수 있으면서도,  
생성자만 수정/삭제할 수 있도록 제한하려면 커스텀 권한을 만들어야 합니다.

`snippets` 앱에 `permissions.py` 파일 생성:

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    오직 소유자만 객체를 수정 가능하도록 하는 커스텀 권한.
    """

    def has_object_permission(self, request, view, obj):
        # 읽기 권한은 모든 요청에 허용
        if request.method in permissions.SAFE_METHODS:
            return True

        # 쓰기 권한은 오직 스니펫 소유자만 허용
        return obj.owner == request.user
```

`SnippetDetail` view 클래스의 `permission_classes`에 추가합니다.

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

`IsOwnerOrReadOnly` import도 잊지 마세요.

```python
from snippets.permissions import IsOwnerOrReadOnly
```

이제 브라우저에서 확인하면, 스니펫의 `DELETE`와 `PUT` 버튼은  
해당 스니펫의 소유자일 때만 표시됩니다.

## API 인증(Authentication)

이제 API에 권한이 적용되므로, 스니펫을 수정하려면 인증이 필요합니다.  
아직 별도의 [authentication class][authentication]를 설정하지 않았으므로, 기본값(`SessionAuthentication`과 `BasicAuthentication`)이 적용됩니다.

- 브라우저 접근 시 로그인 후 세션 인증 제공
- 프로그램으로 접근 시 매 요청마다 인증 정보 제공 필요

인증 없이 스니펫 생성 시 오류 발생:

```bash
http POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
    "detail": "Authentication credentials were not provided."
}
```

사용자 이름과 비밀번호를 포함하면 정상 요청 가능:

```bash
http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"

{
    "id": 1,
    "owner": "admin",
    "title": "foo",
    "code": "print(789)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

## 요약

이제 API는 다음을 제공합니다.

- 사용자(User) 엔드포인트
- 스니펫(Snippet) 엔드포인트
- 인증 기반 CRUD 권한
- 객체 수준 권한(소유자만 수정/삭제 가능)

[tutorial part 5][tut-5]에서는 하이라이트된 스니펫을 HTML로 표시하는 endpoint를 만들고,  
API 관계를 하이퍼링크로 연결하여 API 응집도를 높이는 방법을 살펴봅니다.

[authentication]: ../api-guide/authentication.md
[tut-5]: 5-relationships-and-hyperlinked-apis.md
