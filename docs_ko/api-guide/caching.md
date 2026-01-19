# 캐싱 (Caching)

> 어떤 여자는 매우 날카로운 의식을 가지고 있었지만
> 기억력은 거의 없었다 …
> 그녀는 일할 만큼은 기억했고, 열심히 일했다.
> – Lydia Davis

REST Framework에서의 캐싱은  
Django가 제공하는 캐시 유틸리티와 함께 사용할 때 잘 동작한다.

---

## APIView 및 ViewSet에서 캐시 사용하기

Django는 클래스 기반 뷰에서 데코레이터를 사용할 수 있도록  
[`method_decorator`][decorator]를 제공한다.  

이를 활용해 [`cache_page`][page], [`vary_on_cookie`][cookie],
[`vary_on_headers`][headers] 와 같은 캐시 관련 데코레이터를
클래스 기반 뷰에 적용할 수 있다.

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie, vary_on_headers

from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework import viewsets


class UserViewSet(viewsets.ViewSet):
    # 쿠키 기준: 사용자별로 요청 URL을 2시간 동안 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    @method_decorator(vary_on_cookie)
    def list(self, request, format=None):
        content = {
            "user_feed": request.user.get_user_feed(),
        }
        return Response(content)


class ProfileView(APIView):
    # 인증 헤더 기준: 사용자별로 요청 URL을 2시간 동안 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    @method_decorator(vary_on_headers("Authorization"))
    def get(self, request, format=None):
        content = {
            "user_feed": request.user.get_user_feed(),
        }
        return Response(content)


class PostView(APIView):
    # 요청된 URL 자체를 2시간 동안 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    def get(self, request, format=None):
        content = {
            "title": "Post title",
            "body": "Post content",
        }
        return Response(content)
```

## @api_view 데코레이터와 함께 캐시 사용하기

`@api_view` 데코레이터를 사용하는 경우에는  
Django에서 제공하는 메서드 기반 캐시 데코레이터인  
[`cache_page`][page], [`vary_on_cookie`][cookie],
[`vary_on_headers`][headers] 를 직접 사용할 수 있다.

```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie

from rest_framework.decorators import api_view
from rest_framework.response import Response


@cache_page(60 * 15)
@vary_on_cookie
@api_view(["GET"])
def get_user_list(request):
    content = {"user_feed": request.user.get_user_feed()}
    return Response(content)
```

**NOTE:**  [`cache_page`][page] 데코레이터는 상태 코드가 **200인 `GET` 및 `HEAD` 요청만 캐시**한다.

[page]: https://docs.djangoproject.com/en/stable/topics/cache/#the-per-view-cache
[cookie]: https://docs.djangoproject.com/en/stable/topics/http/decorators/#django.views.decorators.vary.vary_on_cookie
[headers]: https://docs.djangoproject.com/en/stable/topics/http/decorators/#django.views.decorators.vary.vary_on_headers
[decorator]: https://docs.djangoproject.com/en/stable/topics/class-based-views/intro/#decorating-the-class
