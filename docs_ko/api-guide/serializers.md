---
source:
    - serializers.py
---

# 시리얼라이저 (Serializers)

> 시리얼라이저의 활용성을 확장하는 것은 우리가 해결하고 싶은 문제다.
> 하지만 이는 결코 간단하지 않으며, 상당한 설계 작업이 필요하다.
>
> &mdash; Russell Keith-Magee, [Django users group][cite]

시리얼라이저는 queryset이나 모델 인스턴스와 같은 **복잡한 데이터**를  
`JSON`, `XML` 등의 콘텐츠 타입으로 쉽게 렌더링할 수 있는 **파이썬 기본 자료형**으로 변환해 준다.  
또한 시리얼라이저는 **역직렬화(deserialization)** 도 제공하여, 파싱된 데이터를 검증한 뒤 다시 복잡한 타입으로 변환할 수 있다.

REST framework의 시리얼라이저는 Django의 `Form`, `ModelForm` 클래스와 매우 유사하게 동작한다.  
응답 출력 방식을 세밀하게 제어할 수 있는 범용적인 `Serializer` 클래스와,  
모델 인스턴스 및 queryset을 다루기 위한 단축형인 `ModelSerializer` 클래스를 제공한다.

## 시리얼라이저 선언하기

예제를 위해 간단한 객체를 하나 만들어 보자.

    from datetime import datetime

    class Comment:
        def __init__(self, email, content, created=None):
            self.email = email
            self.content = content
            self.created = created or datetime.now()

    comment = Comment(email='leila@example.com', content='foo bar')

이제 `Comment` 객체와 매핑되는 데이터를 직렬화/역직렬화할 시리얼라이저를 선언해 보자.

시리얼라이저 선언 방식은 폼 선언과 매우 비슷하다.

    from rest_framework import serializers

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

## 객체 직렬화 (Serializing objects)

이제 `CommentSerializer`를 사용해 단일 댓글 또는 댓글 목록을 직렬화할 수 있다.

    serializer = CommentSerializer(comment)
    serializer.data
    # {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}

이 단계에서 모델 인스턴스는 파이썬 기본 자료형으로 변환되었다.  
마지막으로 이를 `json`으로 렌더링한다.

    from rest_framework.renderers import JSONRenderer

    json = JSONRenderer().render(serializer.data)
    json
    # b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'

## 객체 역직렬화 (Deserializing objects)

역직렬화 과정도 유사하다.  
먼저 스트림을 파싱해 파이썬 기본 자료형으로 변환한다.

    import io
    from rest_framework.parsers import JSONParser

    stream = io.BytesIO(json)
    data = JSONParser().parse(stream)

그 다음, 이를 검증된 데이터 딕셔너리로 복원한다.

    serializer = CommentSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(...)}

## 인스턴스 저장하기

검증된 데이터를 기반으로 실제 객체 인스턴스를 반환하려면  
`.create()` 및/또는 `.update()` 메서드를 구현해야 한다.

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

        def create(self, validated_data):
            return Comment(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            return instance

Django 모델과 매핑되는 경우에는 데이터베이스에 저장하도록 구현해야 한다.

        def create(self, validated_data):
            return Comment.objects.create(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            instance.save()
            return instance

이제 역직렬화 시 `.save()`를 호출하면 객체 인스턴스가 반환된다.

    comment = serializer.save()

`.save()`는 기존 인스턴스가 전달되었는지 여부에 따라 새 객체를 생성하거나 업데이트한다.

    serializer = CommentSerializer(data=data)            # create
    serializer = CommentSerializer(comment, data=data)   # update

`.create()`와 `.update()`는 필요에 따라 구현하지 않거나, 하나만 구현하거나, 둘 다 구현할 수 있다.

#### `.save()`에 추가 인자 전달하기

뷰 코드에서 요청 데이터에 포함되지 않은 추가 정보를 주입하고 싶을 수 있다  
(예: 현재 사용자, 현재 시각 등).

    serializer.save(owner=request.user)

추가 인자는 `.create()` / `.update()` 호출 시 `validated_data`에 포함된다.

#### `.save()` 직접 오버라이드하기

경우에 따라 `.create()` / `.update()`라는 이름이 의미에 맞지 않을 수 있다.  
예를 들어 연락처 폼에서는 객체를 생성하는 대신 이메일을 전송할 수도 있다.

    class ContactForm(serializers.Serializer):
        email = serializers.EmailField()
        message = serializers.CharField()

        def save(self):
            email = self.validated_data['email']
            message = self.validated_data['message']
            send_email(from=email, message=message)

이 경우 `.validated_data`에 직접 접근해야 한다.

## 검증 (Validation)

역직렬화 시에는 반드시 `.is_valid()`를 호출해야 하며,  
검증 오류가 있으면 `.errors` 속성에 에러 정보가 담긴다.

    serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
    serializer.is_valid()
    # False
    serializer.errors
    # {'email': ['Enter a valid email address.'], 'created': ['This field is required.']}

`non_field_errors` 키에는 필드에 속하지 않는 일반 오류가 들어갈 수 있다.  
키 이름은 `NON_FIELD_ERRORS_KEY` 설정으로 변경 가능하다.

#### 검증 실패 시 예외 발생시키기

    serializer.is_valid(raise_exception=True)

기본 예외 핸들러에 의해 `HTTP 400 Bad Request` 응답이 반환된다.

#### 필드 단위 검증

`validate_<field_name>` 메서드를 추가해 필드 단위 검증을 할 수 있다.

    class BlogPostSerializer(serializers.Serializer):
        title = serializers.CharField(max_length=100)
        content = serializers.CharField()

        def validate_title(self, value):
            if 'django' not in value.lower():
                raise serializers.ValidationError("Blog post is not about Django")
            return value

---

**참고:** `required=False`인 필드는 값이 없으면 해당 검증이 실행되지 않는다.

---

#### 객체 단위 검증

여러 필드를 동시에 검증하려면 `.validate()` 메서드를 사용한다.

    class EventSerializer(serializers.Serializer):
        description = serializers.CharField(max_length=100)
        start = serializers.DateTimeField()
        finish = serializers.DateTimeField()

        def validate(self, data):
            if data['start'] > data['finish']:
                raise serializers.ValidationError("finish must occur after start")
            return data

#### Validator 사용

필드에 validator를 직접 지정할 수 있다.

    def multiple_of_ten(value):
        if value % 10 != 0:
            raise serializers.ValidationError('Not a multiple of ten')

    class GameRecord(serializers.Serializer):
        score = serializers.IntegerField(validators=[multiple_of_ten])

또는 `Meta.validators`로 객체 단위 validator를 지정할 수 있다.

자세한 내용은 [validators 문서](validators.md)를 참고하라.

## 초기 데이터와 인스턴스 접근

* `.instance` : 초기 객체 (없으면 `None`)
* `.initial_data` : 전달된 원본 데이터 (`data`가 없으면 존재하지 않음)

## 부분 업데이트 (Partial updates)

    serializer = CommentSerializer(comment, data={'content': 'foo bar'}, partial=True)

## 중첩 객체 처리

시리얼라이저는 다른 시리얼라이저를 필드로 포함해 중첩 구조를 표현할 수 있다.

    class UserSerializer(serializers.Serializer):
        email = serializers.EmailField()
        username = serializers.CharField(max_length=100)

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

`required=False`, `many=True` 옵션으로 선택적/리스트 관계를 표현할 수 있다.

## 쓰기 가능한 중첩 표현

중첩된 객체에 오류가 있으면 필드 이름 아래에 중첩되어 표시된다.

중첩 구조를 저장하려면 `.create()` / `.update()`를 직접 구현해야 하며,  
기본 `ModelSerializer`는 이를 지원하지 않는다.

자동 처리를 원한다면 [DRF Writable Nested][thirdparty-writable-nested] 같은 패키지를 사용할 수 있다.

## ModelSerializer

`ModelSerializer`는 모델 필드에 매핑되는 시리얼라이저를 자동 생성해 준다.

특징:

* 모델 기반 필드 자동 생성
* `unique_together` 같은 validator 자동 생성
* 기본 `.create()` / `.update()` 구현 제공

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ['id', 'account_name', 'users', 'created']

`fields` 또는 `exclude` 중 하나는 **반드시** 지정해야 한다(3.3.0 이후 필수).

## HyperlinkedModelSerializer

관계를 기본 키 대신 **하이퍼링크**로 표현한다.

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Account
            fields = ['url', 'id', 'account_name', 'users', 'created']

`context={'request': request}`를 전달해야 절대 URL이 생성된다.

## ListSerializer

`many=True`로 여러 객체를 직렬화/역직렬화할 때 내부적으로 사용된다.

* `allow_empty`
* `max_length`
* `min_length`

다중 생성/업데이트 로직을 커스터마이징하려면 `list_serializer_class`를 사용한다.

## BaseSerializer

대체 직렬화/역직렬화 방식을 구현하기 위한 저수준 API다.

오버라이드 가능 메서드:

* `.to_representation()`
* `.to_internal_value()`
* `.create()`
* `.update()`

Browsable API에서 HTML 폼은 생성되지 않는다.

## 고급 사용법

* 직렬화/역직렬화 동작 오버라이드
* 시리얼라이저 상속
* 런타임에 필드 동적 변경
* 커스텀 기본 필드 매핑

---

# 서드파티 패키지

* Django REST marshmallow
* Serpy
* MongoengineModelSerializer
* GeoFeatureModelSerializer
* HStoreSerializer
* Dynamic REST
* DRF Writable Nested
* DRF FlexFields
* DRF Dynamic Fields
* DRF Encrypt Content
* Shapeless Serializers

등 다양한 확장 패키지가 제공된다.

[cite]: https://groups.google.com/d/topic/django-users/sVFaOfQi4wY/discussion
[thirdparty-writable-nested]: serializers.md#drf-writable-nested