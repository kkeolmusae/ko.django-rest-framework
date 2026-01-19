---
source:
    - throttling.py
---

# Throttling

> HTTP/1.1 420 Enhance Your Calm
>
> [Twitter API rate limiting response][cite]

Throttling은 [permissions]와 유사하게, 요청이 허용되어야 하는지를 판단합니다.  
Throttle은 **일시적인 상태**를 나타내며, 클라이언트가 API에 요청할 수 있는 **요청 속도(rate)**를 제어하는 데 사용됩니다.

Permissions와 마찬가지로, **여러 개의 throttle을 함께 사용할 수 있습니다**.  
예를 들어, 인증되지 않은 요청에는 더 엄격한 throttle을 적용하고, 인증된 요청에는 덜 제한적인 throttle을 적용할 수 있습니다.

또 다른 시나리오로는, API의 특정 부분이 특히 리소스를 많이 사용하는 경우, 해당 부분에만 다른 제약을 두고 싶을 때 여러 throttle을 사용하는 경우가 있습니다.

또한 **burst(단기 폭주) 제한**과 **지속적인 제한**을 동시에 적용하기 위해 여러 throttle을 사용할 수도 있습니다.  
예를 들어, 사용자를 *분당 최대 60회*, *하루 최대 1000회* 요청으로 제한할 수 있습니다.

Throttle은 반드시 요청 속도 제한만을 의미하지는 않습니다.  
예를 들어, 스토리지 서비스는 대역폭 기준으로 throttle을 걸 수도 있고, 유료 데이터 서비스는 접근 가능한 레코드 수 기준으로 throttle을 적용할 수도 있습니다.

**REST framework에서 제공하는 애플리케이션 레벨 throttling은 보안 수단이나 brute force, DoS 공격에 대한 방어책으로 간주되어서는 안 됩니다.  
악의적인 공격자는 IP를 쉽게 위조할 수 있으며, 기본 throttling 구현은 Django의 cache 프레임워크를 사용하고 비원자적(non-atomic) 연산을 기반으로 하므로, 요청 수 계산에 약간의 오차가 발생할 수 있습니다.**

**REST framework의 throttling은 비즈니스 티어 구분이나 서비스 과사용 방지와 같은 정책을 구현하기 위한 용도로 의도되었습니다.**

## How throttling is determined

Permissions 및 authentication과 마찬가지로, REST framework에서 throttling은 **클래스 목록(list of classes)**으로 정의됩니다.

뷰의 본문 로직이 실행되기 전에, 리스트에 포함된 각 throttle이 순차적으로 검사됩니다.  
어느 하나라도 실패하면 `exceptions.Throttled` 예외가 발생하며, 뷰의 본문은 실행되지 않습니다.

## Setting the throttling policy

기본 throttling 정책은 전역 설정으로 지정할 수 있으며,  
`DEFAULT_THROTTLE_CLASSES` 와 `DEFAULT_THROTTLE_RATES` 를 사용합니다. 예시는 다음과 같습니다.

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': [
            'rest_framework.throttling.AnonRateThrottle',
            'rest_framework.throttling.UserRateThrottle'
        ],
        'DEFAULT_THROTTLE_RATES': {
            'anon': '100/day',
            'user': '1000/day'
        }
    }

`DEFAULT_THROTTLE_RATES` 에서 rate는 `초/분/시간/일` 단위로 지정할 수 있습니다.  
`/` 뒤에 `s`, `m`, `h`, `d` 를 사용하며,  
`second`, `minute`, `hour`, `day` 또는 `sec`, `min`, `hr` 와 같은 확장 표현도 사용할 수 있습니다.  
실제로는 **첫 글자만** 판별에 사용됩니다.

또한, `APIView` 기반의 클래스 뷰에서 **뷰 또는 뷰셋 단위로 throttling 정책을 설정**할 수 있습니다.

    from rest_framework.response import Response
    from rest_framework.throttling import UserRateThrottle
    from rest_framework.views import APIView

    class ExampleView(APIView):
        throttle_classes = [UserRateThrottle]

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

함수 기반 뷰에서 `@api_view` 데코레이터를 사용하는 경우, 다음과 같이 설정할 수 있습니다.

    @api_view(['GET'])
    @throttle_classes([UserRateThrottle])
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

`@action` 데코레이터로 생성된 라우트에도 throttle 클래스를 지정할 수 있습니다.  
이 경우, 해당 throttle 설정은 **뷰셋 레벨 설정을 덮어씁니다**.

    @action(detail=True, methods=["post"], throttle_classes=[UserRateThrottle])
    def example_adhoc_method(request, pk=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

## How clients are identified

Throttle에서 클라이언트를 식별하기 위해 `X-Forwarded-For` HTTP 헤더와  
WSGI 환경 변수 `REMOTE_ADDR` 가 사용됩니다.

`X-Forwarded-For` 헤더가 존재하면 이를 사용하고,  
그렇지 않으면 `REMOTE_ADDR` 값을 사용합니다.

클라이언트 IP를 엄격하게 식별해야 하는 경우,  
API가 몇 개의 애플리케이션 프록시 뒤에서 동작하는지를  
`NUM_PROXIES` 설정으로 지정해야 합니다.

`NUM_PROXIES` 는 0 이상의 정수 값입니다.

- 0보다 큰 값이면, `X-Forwarded-For` 헤더에서 프록시 IP를 제외한 **마지막 IP**를 클라이언트 IP로 사용합니다.
- 0이면, 항상 `REMOTE_ADDR` 값을 사용합니다.

`NUM_PROXIES` 를 설정할 경우,  
하나의 [NAT](https://en.wikipedia.org/wiki/Network_address_translation) 게이트웨이 뒤에 있는 모든 클라이언트는 **동일한 클라이언트로 간주**된다는 점에 유의해야 합니다.

`X-Forwarded-For` 헤더와 원격 클라이언트 IP 식별 방식에 대한 추가 설명은  
[여기][identifying-clients]에서 확인할 수 있습니다.

## Setting up the cache

REST framework에서 제공하는 throttle 클래스는 Django의 cache 백엔드를 사용합니다.  
적절한 [cache 설정][cache-setting]이 필요합니다.

기본값인 `LocMemCache` 는 단순한 환경에서는 충분합니다.  
자세한 내용은 Django의 [cache 문서][cache-docs]를 참고하세요.

`'default'` 가 아닌 캐시를 사용해야 하는 경우,  
커스텀 throttle 클래스를 만들고 `cache` 속성을 지정할 수 있습니다.

    from django.core.cache import caches

    class CustomAnonRateThrottle(AnonRateThrottle):
        cache = caches['alternate']

이 경우, 해당 커스텀 throttle 클래스를  
`'DEFAULT_THROTTLE_CLASSES'` 또는 뷰의 `throttle_classes` 에 반드시 등록해야 합니다.

## A note on concurrency

기본 throttle 구현은 [race condition][race]에 취약하므로,  
높은 동시성 환경에서는 일부 요청이 초과 허용될 수 있습니다.

동시 요청 상황에서도 요청 수를 **엄격하게 보장해야 한다면**,  
직접 커스텀 throttle 클래스를 구현해야 합니다.  
자세한 내용은 [issue #5181][gh5181]을 참고하세요.

---

# API Reference

## AnonRateThrottle

`AnonRateThrottle` 은 **인증되지 않은 사용자만** 대상으로 throttle을 적용합니다.  
요청의 IP 주소를 기준으로 고유한 throttle 키를 생성합니다.

허용 요청 수는 다음 우선순위로 결정됩니다.

- 클래스의 `rate` 속성 (상속 후 직접 지정한 경우)
- `DEFAULT_THROTTLE_RATES['anon']` 설정

알 수 없는 출처의 요청을 제한하고 싶을 때 적합합니다.

## UserRateThrottle

`UserRateThrottle` 은 사용자 단위로 API 전체에 대해 요청 수를 제한합니다.  
사용자 ID를 기준으로 throttle 키를 생성합니다.

인증되지 않은 요청은 IP 주소를 기준으로 처리됩니다.

허용 요청 수는 다음 우선순위로 결정됩니다.

- 클래스의 `rate` 속성
- `DEFAULT_THROTTLE_RATES['user']` 설정

여러 개의 `UserRateThrottle` 을 동시에 적용할 수도 있습니다.  
이 경우, 각 클래스마다 고유한 `scope` 값을 지정해야 합니다.

예시는 다음과 같습니다.

    class BurstRateThrottle(UserRateThrottle):
        scope = 'burst'

    class SustainedRateThrottle(UserRateThrottle):
        scope = 'sustained'

설정은 다음과 같이 합니다.

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': [
            'example.throttles.BurstRateThrottle',
            'example.throttles.SustainedRateThrottle'
        ],
        'DEFAULT_THROTTLE_RATES': {
            'burst': '60/min',
            'sustained': '1000/day'
        }
    }

간단한 사용자 단위 전역 요청 제한에 적합합니다.

## ScopedRateThrottle

`ScopedRateThrottle` 은 API의 특정 부분에만 제한을 적용할 때 사용합니다.  
뷰에 `.throttle_scope` 속성이 정의되어 있을 때만 적용됩니다.

Throttle 키는 **scope + 사용자 ID 또는 IP 주소** 조합으로 생성됩니다.

허용 요청 수는 `DEFAULT_THROTTLE_RATES` 에서 scope 이름을 키로 하여 결정됩니다.

예시:

    class ContactListView(APIView):
        throttle_scope = 'contacts'
        ...

    class ContactDetailView(APIView):
        throttle_scope = 'contacts'
        ...

    class UploadView(APIView):
        throttle_scope = 'uploads'
        ...

설정:

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': [
            'rest_framework.throttling.ScopedRateThrottle',
        ],
        'DEFAULT_THROTTLE_RATES': {
            'contacts': '1000/day',
            'uploads': '20/day'
        }
    }

`ContactListView` 와 `ContactDetailView` 에 대한 요청은  
하루 총 1000회로 제한되며,  
`UploadView` 는 하루 20회로 제한됩니다.

---

# Custom throttles

커스텀 throttle을 만들려면 `BaseThrottle` 을 상속하고  
`.allow_request(self, request, view)` 메서드를 구현해야 합니다.

요청을 허용할 경우 `True`, 거부할 경우 `False` 를 반환합니다.

선택적으로 `.wait()` 메서드를 오버라이드할 수 있습니다.  
이 메서드는 다음 요청까지 **기다려야 할 권장 시간(초)**을 반환하거나 `None` 을 반환합니다.

`.wait()` 는 `.allow_request()` 가 `False` 를 반환한 경우에만 호출됩니다.

`.wait()` 가 구현되어 있고 요청이 제한되면,  
응답에 `Retry-After` 헤더가 포함됩니다.

## Example

다음은 요청의 10분의 1을 무작위로 차단하는 throttle 예제입니다.

    import random

    class RandomRateThrottle(throttling.BaseThrottle):
        def allow_request(self, request, view):
            return random.randint(1, 10) != 1

[cite]: https://developer.twitter.com/en/docs/basics/rate-limiting
[permissions]: permissions.md
[identifying-clients]: http://oxpedia.org/wiki/index.php?title=AppSuite:Grizzly#Multiple_Proxies_in_front_of_the_cluster
[cache-setting]: https://docs.djangoproject.com/en/stable/ref/settings/#caches
[cache-docs]: https://docs.djangoproject.com/en/stable/topics/cache/#setting-up-the-cache
[gh5181]: https://github.com/encode/django-rest-framework/issues/5181
[race]: https://en.wikipedia.org/wiki/Race_condition#Data_race
