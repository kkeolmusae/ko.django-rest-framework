---
source:
    - mixins.py
    - generics.py
---

# 제네릭 뷰 (Generic views)

> Django의 제네릭 뷰는 공통적인 사용 패턴을 위한 지름길로 개발되었습니다.  
> 뷰 개발에서 자주 등장하는 관용구와 패턴을 추상화하여, 같은 코드를 반복하지 않고도 데이터에 대한 일반적인 뷰를 빠르게 작성할 수 있도록 합니다.
>
> &mdash; [Django Documentation][cite]

클래스 기반 뷰의 핵심적인 장점 중 하나는 재사용 가능한 동작을 조합할 수 있다는 점입니다.  
REST framework는 이를 활용하여, 자주 사용되는 패턴을 처리하는 여러 가지 사전 정의된 뷰를 제공합니다.

REST framework가 제공하는 제네릭 뷰를 사용하면 데이터베이스 모델과 밀접하게 매핑되는 API 뷰를 빠르게 구축할 수 있습니다.

제네릭 뷰가 API 요구사항에 맞지 않는 경우에는 일반 `APIView` 클래스를 직접 사용하거나,  
제네릭 뷰에서 사용하는 mixin과 베이스 클래스를 재사용하여 자신만의 재사용 가능한 제네릭 뷰 세트를 구성할 수 있습니다.

## 예제 (Examples)

일반적으로 제네릭 뷰를 사용할 때는 뷰 클래스를 상속하고 여러 클래스 속성을 설정합니다.

    from django.contrib.auth.models import User
    from myapp.serializers import UserSerializer
    from rest_framework import generics
    from rest_framework.permissions import IsAdminUser

    class UserList(generics.ListCreateAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer
        permission_classes = [IsAdminUser]

더 복잡한 경우에는 뷰 클래스의 여러 메서드를 오버라이드할 수도 있습니다. 예를 들면 다음과 같습니다.

    class UserList(generics.ListCreateAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer
        permission_classes = [IsAdminUser]

        def list(self, request):
            # `self.queryset` 대신 `get_queryset()`을 사용하는 점에 주의
            queryset = self.get_queryset()
            serializer = UserSerializer(queryset, many=True)
            return Response(serializer.data)

아주 단순한 경우에는 `.as_view()` 메서드를 사용하여 클래스 속성을 그대로 전달할 수도 있습니다.  
예를 들어 URL 설정에는 다음과 같은 항목이 포함될 수 있습니다.

    path('users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')

---

# API 참조 (API Reference)

## GenericAPIView

이 클래스는 REST framework의 `APIView` 클래스를 확장한 것으로,  
표준적인 목록(list) 및 상세(detail) 뷰에 공통적으로 필요한 동작을 추가로 제공합니다.

REST framework에서 제공하는 모든 구체적인 제네릭 뷰는  
`GenericAPIView`와 하나 이상의 mixin 클래스를 조합하여 구성됩니다.

### 속성 (Attributes)

**기본 설정 (Basic settings)**

다음 속성들은 뷰의 기본 동작을 제어합니다.

* `queryset`  
  이 뷰에서 객체를 반환하는 데 사용할 queryset입니다.  
  일반적으로 이 속성을 설정하거나 `get_queryset()` 메서드를 오버라이드해야 합니다.  
  뷰 메서드를 오버라이드하는 경우, 이 속성에 직접 접근하지 말고 반드시 `get_queryset()`을 호출해야 합니다.  
  `queryset`은 한 번만 평가되며, 그 결과는 이후 모든 요청에 대해 캐시됩니다.

* `serializer_class`  
  입력 데이터를 검증하고 역직렬화하며, 출력 데이터를 직렬화하는 데 사용할 serializer 클래스입니다.  
  일반적으로 이 속성을 설정하거나 `get_serializer_class()` 메서드를 오버라이드해야 합니다.

* `lookup_field`  
  개별 모델 인스턴스를 조회할 때 사용할 모델 필드입니다.  
  기본값은 `'pk'`입니다.  
  하이퍼링크 기반 API를 사용하는 경우, 사용자 정의 값을 사용하려면  
  API 뷰와 serializer 클래스 **모두**에서 lookup 필드를 설정해야 합니다.

* `lookup_url_kwarg`  
  객체 조회에 사용할 URL 키워드 인자입니다.  
  URL 설정에는 이 값에 해당하는 키워드 인자가 포함되어야 합니다.  
  설정하지 않으면 `lookup_field`와 동일한 값을 사용합니다.

**페이지네이션 (Pagination)**

다음 속성들은 목록 뷰에서 페이지네이션을 제어하는 데 사용됩니다.

* `pagination_class`  
  목록 결과를 페이지네이션할 때 사용할 페이지네이션 클래스입니다.  
  기본값은 `DEFAULT_PAGINATION_CLASS` 설정과 동일하며,  
  이는 `'rest_framework.pagination.PageNumberPagination'`입니다.  
  `pagination_class = None`으로 설정하면 해당 뷰에서 페이지네이션이 비활성화됩니다.

**필터링 (Filtering)**

* `filter_backends`  
  queryset을 필터링하는 데 사용할 필터 백엔드 클래스 목록입니다.  
  기본값은 `DEFAULT_FILTER_BACKENDS` 설정과 동일합니다.

---

### 메서드 (Methods)

**기본 메서드 (Base methods)**

#### `get_queryset(self)`

목록 뷰에서 사용할 queryset을 반환하며,  
상세 뷰에서는 객체 조회의 기준이 되는 queryset으로 사용됩니다.  
기본적으로 `queryset` 속성에 지정된 값을 반환합니다.

이 메서드는 `self.queryset`에 직접 접근하는 대신 항상 사용해야 합니다.  
`self.queryset`은 한 번만 평가되고 이후 요청에 대해 캐시되기 때문입니다.

요청 사용자에 따라 서로 다른 queryset을 반환하는 등  
동적인 동작을 구현하기 위해 오버라이드할 수 있습니다.

예:

    def get_queryset(self):
        user = self.request.user
        return user.accounts.all()

---

**참고(Note):**  
제네릭 뷰에서 사용하는 `serializer_class`가 ORM 관계를 가로질러 접근하면서 N+1 문제를 유발하는 경우,  
이 메서드에서 `select_related`와 `prefetch_related`를 사용해 queryset을 최적화할 수 있습니다.  
N+1 문제와 관련 메서드의 사용 사례에 대해서는 [Django 문서][django-docs-select-related]를 참고하세요.

---

### N+1 쿼리 방지 (Avoiding N+1 Queries)

객체 목록을 반환할 때(예: `ListAPIView`, `ModelViewSet`),  
serializer가 각 항목마다 관련 객체에 접근하면 N+1 쿼리 패턴이 발생할 수 있습니다.

이를 방지하려면 `get_queryset()` 메서드나 `queryset` 클래스 속성에서  
관계 유형에 따라 `select_related()` 또는 `prefetch_related()`를 사용해 queryset을 최적화해야 합니다.

**ForeignKey 및 OneToOneField의 경우**

같은 쿼리에서 관련 객체를 함께 가져오기 위해 `select_related()`를 사용합니다.

    def get_queryset(self):
        return Order.objects.select_related("customer", "billing_address")

**역참조 및 다대다 관계의 경우**

관련 객체 컬렉션을 효율적으로 로딩하기 위해 `prefetch_related()`를 사용합니다.

    def get_queryset(self):
        return Book.objects.prefetch_related("categories", "reviews__user")

**두 방법을 함께 사용하는 경우**

    def get_queryset(self):
        return (
            Order.objects
            .select_related("customer")
            .prefetch_related("items__product")
        )

이러한 최적화는 중복된 데이터베이스 접근을 줄이고 목록 뷰의 성능을 향상시킵니다.

---

#### `get_object(self)`

상세(detail) 뷰에서 사용할 객체 인스턴스를 반환합니다.  
기본적으로 `lookup_field`를 사용하여 기본 queryset에서 객체를 조회합니다.

여러 개의 URL 파라미터를 사용하는 등 더 복잡한 조회 로직이 필요한 경우 오버라이드할 수 있습니다.

예:

    def get_object(self):
        queryset = self.get_queryset()
        filter = {}
        for field in self.multiple_lookup_fields:
            filter[field] = self.kwargs[field]

        obj = get_object_or_404(queryset, **filter)
        self.check_object_permissions(self.request, obj)
        return obj

객체 수준 권한이 필요 없는 API라면  
`self.check_object_permissions` 호출을 생략하고 `get_object_or_404` 결과만 반환해도 됩니다.

#### `filter_queryset(self, queryset)`

주어진 queryset에 현재 사용 중인 필터 백엔드를 적용하여 새로운 queryset을 반환합니다.

예:

    def filter_queryset(self, queryset):
        filter_backends = [CategoryFilter]

        if 'geo_route' in self.request.query_params:
            filter_backends = [GeoRouteFilter, CategoryFilter]
        elif 'geo_point' in self.request.query_params:
            filter_backends = [GeoPointFilter, CategoryFilter]

        for backend in list(filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, view=self)

        return queryset

#### `get_serializer_class(self)`

사용할 serializer 클래스를 반환합니다.  
기본적으로 `serializer_class` 속성을 반환합니다.

읽기/쓰기 작업에 따라 서로 다른 serializer를 사용하거나,  
사용자 유형에 따라 다른 serializer를 제공하고 싶은 경우 오버라이드할 수 있습니다.

예:

    def get_serializer_class(self):
        if self.request.user.is_staff:
            return FullAccountSerializer
        return BasicAccountSerializer

**저장 및 삭제 훅 (Save and deletion hooks)**

다음 메서드들은 mixin 클래스에서 제공되며,  
객체 저장 또는 삭제 동작을 쉽게 커스터마이즈할 수 있도록 합니다.

* `perform_create(self, serializer)` – 새 객체 저장 시 호출 (`CreateModelMixin`)
* `perform_update(self, serializer)` – 기존 객체 수정 시 호출 (`UpdateModelMixin`)
* `perform_destroy(self, instance)` – 객체 삭제 시 호출 (`DestroyModelMixin`)

이 훅들은 요청 데이터에는 없지만 요청 맥락에 의해 결정되는 값을 설정할 때 특히 유용합니다.

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)

또한 저장 전·후에 이메일 발송이나 로그 기록과 같은 추가 동작을 넣는 데도 유용합니다.

    def perform_update(self, serializer):
        instance = serializer.save()
        send_email_confirmation(user=self.request.user, modified=instance)

`ValidationError()`를 발생시켜 추가 검증 로직을 구현하는 데에도 사용할 수 있습니다.

    def perform_create(self, serializer):
        queryset = SignupRequest.objects.filter(user=self.request.user)
        if queryset.exists():
            raise ValidationError('이미 가입 요청이 존재합니다')
        serializer.save(user=self.request.user)

**기타 메서드 (Other methods)**

다음 메서드들은 일반적으로 오버라이드할 필요는 없지만,  
`GenericAPIView`를 사용해 커스텀 뷰를 작성할 때 내부적으로 호출될 수 있습니다.

* `get_serializer_context(self)` – serializer에 전달할 추가 컨텍스트 반환 (`request`, `view`, `format`)
* `get_serializer(self, instance=None, data=None, many=False, partial=False)` – serializer 인스턴스 반환
* `get_paginated_response(self, data)` – 페이지네이션된 `Response` 반환
* `paginate_queryset(self, queryset)` – queryset을 페이지네이션하거나, 비활성화 시 `None` 반환
* `filter_queryset(self, queryset)` – 필터 백엔드를 적용한 queryset 반환

---

# 믹스인 (Mixins)

mixin 클래스는 기본적인 뷰 동작을 제공하는 액션 메서드들을 정의합니다.  
`.get()`이나 `.post()` 같은 핸들러 메서드를 직접 정의하지 않는다는 점에 유의하세요.  
이를 통해 동작을 더 유연하게 조합할 수 있습니다.

mixin 클래스들은 `rest_framework.mixins`에서 import할 수 있습니다.

## ListModelMixin

queryset 목록을 반환하는 `.list(request, *args, **kwargs)` 메서드를 제공합니다.

queryset이 존재하면 `200 OK` 응답과 함께 직렬화된 데이터가 반환되며,  
선택적으로 페이지네이션이 적용될 수 있습니다.

## CreateModelMixin

새 모델 인스턴스를 생성하고 저장하는 `.create(request, *args, **kwargs)` 메서드를 제공합니다.

객체가 생성되면 `201 Created` 응답과 함께 직렬화된 객체가 반환됩니다.  
직렬화 결과에 `url` 키가 포함되어 있다면, 해당 값이 `Location` 헤더에 설정됩니다.

요청 데이터가 유효하지 않은 경우 `400 Bad Request` 응답이 반환됩니다.

## RetrieveModelMixin

기존 모델 인스턴스를 반환하는 `.retrieve(request, *args, **kwargs)` 메서드를 제공합니다.

객체를 찾을 수 있으면 `200 OK`, 그렇지 않으면 `404 Not Found`가 반환됩니다.

## UpdateModelMixin

기존 모델 인스턴스를 수정하고 저장하는 `.update(request, *args, **kwargs)` 메서드를 제공합니다.

또한 `PATCH` 요청을 위한 `.partial_update(request, *args, **kwargs)` 메서드도 제공합니다.

성공 시 `200 OK` 응답이 반환되며,  
유효하지 않은 데이터가 전달된 경우 `400 Bad Request`가 반환됩니다.

## DestroyModelMixin

기존 모델 인스턴스를 삭제하는 `.destroy(request, *args, **kwargs)` 메서드를 제공합니다.

삭제 성공 시 `204 No Content`, 실패 시 `404 Not Found`를 반환합니다.

---

# 구체적인 뷰 클래스 (Concrete View Classes)

다음 클래스들은 실제로 가장 많이 사용되는 제네릭 뷰들입니다.  
특별한 커스터마이징이 필요하지 않다면 일반적으로 이 수준의 클래스를 사용하게 됩니다.

뷰 클래스들은 `rest_framework.generics`에서 import할 수 있습니다.

## CreateAPIView

**생성 전용** 엔드포인트에 사용됩니다.

`post` 메서드를 제공합니다.

확장: [GenericAPIView], [CreateModelMixin]

## ListAPIView

**읽기 전용**, **모델 인스턴스 컬렉션**을 표현하는 엔드포인트에 사용됩니다.

`get` 메서드를 제공합니다.

확장: [GenericAPIView], [ListModelMixin]

## RetrieveAPIView

**읽기 전용**, **단일 모델 인스턴스**를 표현하는 엔드포인트에 사용됩니다.

`get` 메서드를 제공합니다.

확장: [GenericAPIView], [RetrieveModelMixin]

## DestroyAPIView

**삭제 전용**, **단일 모델 인스턴스** 엔드포인트에 사용됩니다.

`delete` 메서드를 제공합니다.

확장: [GenericAPIView], [DestroyModelMixin]

## UpdateAPIView

**수정 전용**, **단일 모델 인스턴스** 엔드포인트에 사용됩니다.

`put`, `patch` 메서드를 제공합니다.

확장: [GenericAPIView], [UpdateModelMixin]

## ListCreateAPIView

**읽기/쓰기**, **모델 인스턴스 컬렉션** 엔드포인트에 사용됩니다.

`get`, `post` 메서드를 제공합니다.

확장: [GenericAPIView], [ListModelMixin], [CreateModelMixin]

## RetrieveUpdateAPIView

**읽기 또는 수정**, **단일 모델 인스턴스** 엔드포인트에 사용됩니다.

`get`, `put`, `patch` 메서드를 제공합니다.

확장: [GenericAPIView], [RetrieveModelMixin], [UpdateModelMixin]

## RetrieveDestroyAPIView

**읽기 또는 삭제**, **단일 모델 인스턴스** 엔드포인트에 사용됩니다.

`get`, `delete` 메서드를 제공합니다.

확장: [GenericAPIView], [RetrieveModelMixin], [DestroyModelMixin]

## RetrieveUpdateDestroyAPIView

**읽기/수정/삭제**, **단일 모델 인스턴스** 엔드포인트에 사용됩니다.

`get`, `put`, `patch`, `delete` 메서드를 제공합니다.

확장: [GenericAPIView], [RetrieveModelMixin], [UpdateModelMixin], [DestroyModelMixin]

---

# 제네릭 뷰 커스터마이징 (Customizing the generic views)

대부분의 경우 기존 제네릭 뷰를 사용하면서 일부 동작만 커스터마이징하면 충분합니다.  
동일한 커스터마이징 로직을 여러 곳에서 반복해서 사용한다면,  
이를 공통 클래스로 분리하는 것이 좋습니다.

## 커스텀 믹스인 생성 (Creating custom mixins)

예를 들어 URL 설정에서 여러 필드를 기준으로 객체를 조회해야 한다면  
다음과 같은 mixin 클래스를 만들 수 있습니다.

    class MultipleFieldLookupMixin:
        """
        기본 단일 필드 조회 대신,
        `lookup_fields` 속성에 지정된 여러 필드를 기준으로 객체를 조회하도록 하는 mixin
        """
        def get_object(self):
            queryset = self.get_queryset()             # 기본 queryset 가져오기
            queryset = self.filter_queryset(queryset)  # 필터 백엔드 적용
            filter = {}
            for field in self.lookup_fields:
                if self.kwargs.get(field):  # 비어 있는 필드는 무시
                    filter[field] = self.kwargs[field]
            obj = get_object_or_404(queryset, **filter)
            self.check_object_permissions(self.request, obj)
            return obj

필요한 경우 이 mixin을 뷰나 뷰셋에 적용하면 됩니다.

    class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer
        lookup_fields = ['account', 'username']

커스텀 믹스인은 특정 동작을 재사용해야 할 때 매우 좋은 선택입니다.

## 커스텀 베이스 클래스 생성 (Creating custom base classes)

여러 뷰에서 동일한 mixin을 반복적으로 사용한다면,  
이를 한 단계 더 발전시켜 공통 베이스 뷰 클래스를 만들 수 있습니다.

    class BaseRetrieveView(MultipleFieldLookupMixin,
                           generics.RetrieveAPIView):
        pass

    class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                        generics.RetrieveUpdateDestroyAPIView):
        pass

커스텀 베이스 클래스는 프로젝트 전반에 걸쳐  
일관된 동작이 반복적으로 필요할 때 유용합니다.

---

# PUT을 생성으로 사용하는 경우 (PUT as create)

REST framework 3.0 이전 버전에서는  
객체가 존재하지 않을 경우 `PUT` 요청을 생성(create)으로 처리했습니다.

하지만 `PUT`을 생성으로 허용하면 객체의 존재 여부를 노출하게 되며,  
삭제된 객체를 다시 생성하는 동작이 직관적이지 않을 수 있습니다.

"`PUT` 시 404 반환"과 "`PUT` 시 생성" 두 방식 모두 상황에 따라 유효할 수 있지만,  
3.0 버전 이후부터는 더 단순하고 명확한 동작을 위해  
기본값으로 `404` 동작을 사용합니다.

---

# 서드파티 패키지 (Third party packages)

다음 서드파티 패키지들은 추가적인 제네릭 뷰 구현을 제공합니다.

## Django Rest Multiple Models

[Django Rest Multiple Models][django-rest-multiple-models]는  
하나의 API 요청으로 여러 개의 직렬화된 모델이나 queryset을 반환할 수 있는  
제네릭 뷰(및 mixin)를 제공합니다.

[cite]: https://docs.djangoproject.com/en/stable/ref/class-based-views/#base-vs-generic-views
[GenericAPIView]: #genericapiview
[ListModelMixin]: #listmodelmixin
[CreateModelMixin]: #createmodelmixin
[RetrieveModelMixin]: #retrievemodelmixin
[UpdateModelMixin]: #updatemodelmixin
[DestroyModelMixin]: #destroymodelmixin
[django-rest-multiple-models]: https://github.com/MattBroach/DjangoRestMultipleModels
[django-docs-select-related]: https://docs.djangoproject.com/en/stable/ref/models/querysets/#django.db.models.query.QuerySet.select_related
