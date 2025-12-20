# Tutorial 1: Serialization

## Introduction

이 튜토리얼에서는 간단한 pastebin 형태의 코드 하이라이팅 Web API를 만들어봅니다.  
이 과정을 통해 REST framework를 구성하는 다양한 컴포넌트를 소개하고, 전체 구조가 어떻게 맞물려 동작하는지에 대해 종합적으로 이해할 수 있도록 합니다.

이 튜토리얼은 비교적 상세하게 구성되어 있으므로, 시작하기 전에 쿠키 하나와 좋아하는 음료 한 잔을 준비하는 것을 권장합니다.  
빠른 개요만 필요하다면 [quickstart] 문서를 먼저 확인하는 것이 좋습니다.

---

**Note**: 이 튜토리얼의 코드는 GitHub의 [encode/rest-framework-tutorial][repo] 레포지토리에 공개되어 있습니다. 자유롭게 클론하여 실제 동작하는 코드를 확인해 보시기 바랍니다.

---

## Setting up a new environment

다른 작업 중인 프로젝트들과 패키지 설정이 섞이지 않도록, [venv]를 사용해 새로운 가상 환경을 먼저 생성하겠습니다.

```bash
python3 -m venv env
source env/bin/activate
```

가상 환경에 진입했으면 필요한 패키지를 설치합니다.

```bash
pip install django
pip install djangorestframework
pip install pygments  # 코드 하이라이팅을 위해 사용합니다
```

**Note:** 가상 환경을 종료하려면 언제든지 `deactivate` 명령을 사용하면 됩니다.  
자세한 내용은 [venv documentation][venv]을 참고하세요.

## Getting started

이제 본격적으로 코딩을 시작할 준비가 되었습니다.  
먼저 새 프로젝트를 생성합니다.

```bash
cd ~
django-admin startproject tutorial
cd tutorial
```

다음으로 간단한 Web API를 만들기 위한 앱을 하나 생성합니다.

```bash
python manage.py startapp snippets
```

이제 새로 만든 `snippets` 앱과 `rest_framework` 앱을 `INSTALLED_APPS`에 추가해야 합니다.  
`tutorial/settings.py` 파일을 열어 다음과 같이 수정합니다.

```text
INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets',
]
```

이제 모든 준비가 끝났습니다.

## Creating a model to work with

이 튜토리얼에서는 코드 스니펫을 저장하기 위한 간단한 `Snippet` 모델을 작성합니다.  
`snippets/models.py` 파일을 열어 아래 내용을 작성하세요.

참고로, 좋은 프로그래밍 관례에는 주석이 포함되지만, 이 문서에서는 코드에 집중하기 위해 주석을 생략했습니다.

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default="")
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(
        choices=LANGUAGE_CHOICES, default="python", max_length=100
    )
    style = models.CharField(choices=STYLE_CHOICES, default="friendly", max_length=100)

    class Meta:
        ordering = ["created"]
```

이제 초기 마이그레이션을 생성하고 데이터베이스를 동기화합니다.

```bash
python manage.py makemigrations snippets
python manage.py migrate snippets
```

## Creating a Serializer class

Web API를 만들기 위한 첫 단계는 `Snippet` 인스턴스를 `json`과 같은 표현 형식으로 직렬화/역직렬화할 수 있는 방법을 제공하는 것입니다.  
이를 위해 Django Form과 매우 유사한 방식으로 동작하는 serializer를 정의합니다.

`snippets` 디렉터리에 `serializers.py` 파일을 생성하고 아래 내용을 추가하세요.

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={"base_template": "textarea.html"})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default="python")
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default="friendly")

    def create(self, validated_data):
        """
        주어진 validated_data를 사용해 새로운 `Snippet` 인스턴스를 생성합니다.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        주어진 validated_data를 사용해 기존 `Snippet` 인스턴스를 수정합니다.
        """
        instance.title = validated_data.get("title", instance.title)
        instance.code = validated_data.get("code", instance.code)
        instance.linenos = validated_data.get("linenos", instance.linenos)
        instance.language = validated_data.get("language", instance.language)
        instance.style = validated_data.get("style", instance.style)
        instance.save()
        return instance
```

serializer 클래스의 첫 부분에서는 직렬화/역직렬화될 필드를 정의합니다.  
`create()`와 `update()` 메서드는 `serializer.save()` 호출 시 실제 모델 인스턴스를 생성하거나 수정하는 방법을 정의합니다.

serializer는 Django의 `Form` 클래스와 매우 유사하며, `required`, `max_length`, `default`와 같은 검증 옵션도 동일하게 제공합니다.

또한 필드 옵션을 통해 HTML 렌더링 방식도 제어할 수 있습니다.  
위 예제의 `{'base_template': 'textarea.html'}` 옵션은 Django Form에서 `widget=widgets.Textarea`를 사용하는 것과 동일합니다.  
이는 이후 브라우저에서 제공되는 API UI를 제어할 때 특히 유용합니다.

나중에는 `ModelSerializer`를 사용해 더 간결한 구현을 할 수 있지만, 지금은 명시적인 정의를 유지하겠습니다.

## Working with Serializers

이제 새로 만든 Serializer 클래스를 실제로 사용해 보겠습니다.  
Django shell에 진입합니다.

```bash
python manage.py shell
```

필요한 모듈을 임포트한 뒤, 테스트용 스니펫 몇 개를 생성합니다.

```pycon
>>> from snippets.models import Snippet
>>> from snippets.serializers import SnippetSerializer
>>> from rest_framework.renderers import JSONRenderer
>>> from rest_framework.parsers import JSONParser

>>> snippet = Snippet(code='foo = "bar"\n')
>>> snippet.save()

>>> snippet = Snippet(code='print("hello, world")\n')
>>> snippet.save()
```

이제 인스턴스를 직렬화해 보겠습니다.

```pycon
>>> serializer = SnippetSerializer(snippet)
>>> serializer.data
{'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}
```

이 시점에서 모델 인스턴스는 Python 기본 자료형으로 변환되었습니다.  
이를 `json`으로 렌더링합니다.

```pycon
>>> content = JSONRenderer().render(serializer.data)
>>> content
b'{"id":2,"title":"","code":"print(\\"hello, world\\")\\n","linenos":false,"language":"python","style":"friendly"}'
```

역직렬화 과정도 비슷합니다.  
먼저 스트림을 Python 기본 자료형으로 파싱합니다.

```pycon
>>> import io

>>> stream = io.BytesIO(content)
>>> data = JSONParser().parse(stream)
```

그 다음, 이를 다시 완전한 객체 인스턴스로 복원합니다.

```pycon
>>> serializer = SnippetSerializer(data=data)
>>> serializer.is_valid()
True
>>> serializer.validated_data
{'title': '', 'code': 'print("hello, world")', 'linenos': False, 'language': 'python', 'style': 'friendly'}
>>> serializer.save()
<Snippet: Snippet object>
```

Django Form을 다뤄본 경험이 있다면 매우 익숙하게 느껴질 것입니다.  
이 유사성은 이후 serializer를 사용하는 view를 작성할 때 더 분명해집니다.

QuerySet도 직렬화할 수 있으며, 이 경우 `many=True` 옵션을 사용합니다.

```pycon
>>> serializer = SnippetSerializer(Snippet.objects.all(), many=True)
>>> serializer.data
[{'id': 1, 'title': '', 'code': 'foo = "bar"\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}, {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}, {'id': 3, 'title': '', 'code': 'print("hello, world")', 'linenos': False, 'language': 'python', 'style': 'friendly'}]
```

## Using ModelSerializers

현재 `SnippetSerializer`는 `Snippet` 모델에 이미 정의된 정보들을 상당 부분 중복하고 있습니다.  
코드를 좀 더 간결하게 만들 수 있다면 좋을 것입니다.

Django의 `Form` / `ModelForm` 관계와 마찬가지로, REST framework 역시 `Serializer`와 `ModelSerializer`를 제공합니다.

이제 `ModelSerializer`를 사용해 serializer를 리팩터링해 보겠습니다.  
`snippets/serializers.py` 파일을 열고 `SnippetSerializer`를 아래와 같이 교체하세요.

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ["id", "title", "code", "linenos", "language", "style"]
```

serializer의 또 다른 장점은 인스턴스를 출력해 전체 필드 구성을 확인할 수 있다는 점입니다.

```pycon
>>> from snippets.serializers import SnippetSerializer

>>> serializer = SnippetSerializer()
>>> print(repr(serializer))
SnippetSerializer():
    id = IntegerField(label='ID', read_only=True)
    title = CharField(allow_blank=True, max_length=100, required=False)
    code = CharField(style={'base_template': 'textarea.html'})
    linenos = BooleanField(required=False)
    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```

`ModelSerializer`는 마법 같은 기능을 제공하는 것이 아니라, 다음 작업을 자동으로 처리해 주는 편의 클래스일 뿐입니다.

- 필드 집합 자동 결정
- `create()` 및 `update()` 메서드의 기본 구현 제공

## Writing regular Django views using our Serializer

이제 serializer를 사용해 API view를 작성해 보겠습니다.  
일단은 REST framework의 고급 기능을 사용하지 않고, 일반 Django view로 구현합니다.

`snippets/views.py` 파일을 열고 다음을 추가합니다.

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

API의 루트 view는 모든 스니펫을 조회하거나, 새 스니펫을 생성하는 기능을 제공합니다.

```python
@csrf_exempt
def snippet_list(request):
    """
    모든 코드 스니펫을 나열하거나 새 스니펫을 생성합니다.
    """
    if request.method == "GET":
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == "POST":
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

CSRF 토큰이 없는 클라이언트에서도 POST 요청을 허용해야 하므로 `csrf_exempt`를 사용했습니다.  
실제 서비스에서는 권장되지 않지만, 지금 단계에서는 충분합니다.

개별 스니펫을 조회, 수정, 삭제하는 view도 작성합니다.

```python
@csrf_exempt
def snippet_detail(request, pk):
    """
    코드 스니펫을 조회, 수정 또는 삭제합니다.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == "GET":
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == "PUT":
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == "DELETE":
        snippet.delete()
        return HttpResponse(status=204)
```

이제 URL을 연결합니다.  
`snippets/urls.py` 파일을 생성합니다.

```python
from django.urls import path
from snippets import views

urlpatterns = [
    path("snippets/", views.snippet_list),
    path("snippets/<int:pk>/", views.snippet_detail),
]
```

루트 URL 설정도 수정합니다 (`tutorial/urls.py`).

```python
from django.urls import path, include

urlpatterns = [
    path("", include("snippets.urls")),
]
```

현재는 잘못된 JSON이나 지원하지 않는 HTTP 메서드에 대해 적절한 에러 처리를 하지 못합니다.  
이 부분은 이후 개선할 예정입니다.

## Testing our first attempt at a Web API

이제 샘플 서버를 실행합니다.

```pycon
>>> quit()
```

...그리고 Django 개발 서버를 실행합니다.

```bash
python manage.py runserver

Validating models...

0 errors found
Django version 5.0, using settings 'tutorial.settings'
Starting Development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

다른 터미널에서 API를 테스트합니다.

[curl] 또는 [httpie]를 사용할 수 있습니다.  
여기서는 httpie를 사용하겠습니다.

```bash
pip install httpie
```

전체 스니펫 목록을 조회합니다.

```bash
http GET http://127.0.0.1:8000/snippets/ --unsorted

HTTP/1.1 200 OK
...
[
    {
        "id": 1,
        "title": "",
        "code": "foo = \"bar\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    },
    {
        "id": 2,
        "title": "",
        "code": "print(\"hello, world\")\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    },
    {
        "id": 3,
        "title": "",
        "code": "print(\"hello, world\")",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }
]
```

특정 스니펫도 조회할 수 있습니다.

```bash
http GET http://127.0.0.1:8000/snippets/2/ --unsorted

HTTP/1.1 200 OK
...
{
    "id": 2,
    "title": "",
    "code": "print(\"hello, world\")\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

브라우저에서 직접 URL에 접속해도 동일한 JSON을 확인할 수 있습니다.

## Where are we now

지금까지 우리는 Django Form과 유사한 직렬화 API와 기본적인 Django view를 사용한 Web API를 구축했습니다.

아직 에러 처리나 고급 기능은 부족하지만, 동작하는 API를 만드는 데에는 충분합니다.

다음 단계는 [part 2 of the tutorial][tut-2]에서 이어서 살펴보겠습니다.

[quickstart]: quickstart.md
[repo]: https://github.com/encode/rest-framework-tutorial
[venv]: https://docs.python.org/3/library/venv.html
[tut-2]: 2-requests-and-responses.md
[httpie]: https://github.com/httpie/httpie#installation
[curl]: https://curl.haxx.se/
