# Tutorial 3: Class-based Views

API view를 함수 기반이 아닌 클래스 기반으로 작성할 수도 있습니다.  
이 패턴은 공통 기능을 재사용할 수 있게 해주며, 코드의 [DRY][dry] 원칙을 지키는 데 도움이 됩니다.

## Rewriting our API using class-based views

먼저 root view를 클래스 기반 view로 리팩터링해보겠습니다.  
`views.py`를 조금 수정하는 것으로 충분합니다.

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    모든 스니펫을 조회하거나 새 스니펫을 생성합니다.
    """

    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

지금까지는 이전 함수 기반 view와 크게 다르지 않습니다.  
HTTP 메서드별로 코드가 더 명확하게 분리되었다는 장점이 있습니다.

다음으로 개별 인스턴스 view도 `views.py`에서 업데이트해야 합니다.

```python
class SnippetDetail(APIView):
    """
    특정 스니펫 인스턴스를 조회, 수정, 삭제합니다.
    """

    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

좋습니다. 여전히 함수 기반 view와 크게 다르지 않습니다.

클래스 기반 view를 사용하게 되었으므로, `snippets/urls.py`도 약간 수정해야 합니다.

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path("snippets/", views.SnippetList.as_view()),
    path("snippets/<int:pk>/", views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

이제 개발 서버를 실행하면 이전과 동일하게 동작할 것입니다.

## Using mixins

클래스 기반 view의 큰 장점 중 하나는, 재사용 가능한 행동 단위를 쉽게 조합할 수 있다는 것입니다.

지금까지 사용한 CRUD(create/retrieve/update/delete) 동작은 모델 기반 API view에서는 거의 비슷하게 반복됩니다.  
REST framework에서는 이러한 공통 동작을 mixin 클래스에서 제공합니다.

다음은 mixin을 사용해 view를 구성한 예시입니다 (`views.py`).

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics


class SnippetList(
    mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView
):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

여기서 일어나는 일을 살펴보겠습니다.

- `GenericAPIView`를 기반 클래스로 사용하여 핵심 기능 제공
- `ListModelMixin`과 `CreateModelMixin`을 추가하여 `.list()`와 `.create()` 기능 사용
- `get`과 `post` 메서드를 각각 적절한 mixin 액션에 바인딩

```python
class SnippetDetail(
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.GenericAPIView,
):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

비슷합니다. `GenericAPIView`를 기반으로 핵심 기능을 제공하고, mixin을 통해 `.retrieve()`, `.update()`, `.destroy()` 액션을 추가했습니다.

## Using generic class-based views

mixin을 사용하면 이전보다 코드가 조금 줄어들었지만,  
REST framework는 이미 mixin이 결합된 제네릭 클래스 기반 view를 제공합니다.  
이를 사용하면 `views.py`를 훨씬 더 간결하게 만들 수 있습니다.

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

와, 상당히 간결해졌습니다.  
많은 기능을 기본으로 제공받으며, 코드도 깔끔하고 Django 스타일에 맞습니다.

다음으로 [tutorial part 4][tut-4]에서는 API 인증(Authentication)과 권한(Permissions) 설정 방법을 살펴봅니다.

[dry]: https://en.wikipedia.org/wiki/Don't_repeat_yourself
[tut-4]: 4-authentication-and-permissions.md
