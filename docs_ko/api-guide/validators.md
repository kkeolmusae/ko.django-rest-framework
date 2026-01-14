---
source:
  - validators.py
---

# Validators

> Validators can be useful for reusing validation logic between different types of fields.
>
> &mdash; [Django documentation][cite]

대부분의 DRF 검증은 **필드 기본 검증**(예: max_length, Email 형식 등)이나, 시리얼라이저/필드에 작성하는 **명시적 validation 메서드**로 해결한다.  
하지만 검증 로직을 여러 곳에서 재사용해야 한다면 **validator 함수/클래스**로 분리해 재사용 컴포넌트로 만들 수 있다.

## DRF에서의 Validation 방식

Django `ModelForm`은 검증이 form + model에 나뉘어 수행된다.  
반면 DRF는 검증이 **전부 serializer에서 수행**된다. 장점은:

- 관심사 분리로 동작이 더 명확해짐
- `ModelSerializer` ↔ `Serializer`로 바꾸기 쉬움(검증이 serializer에 모여 있음)
- `repr(serializer)`만 출력해도 어떤 검증이 걸리는지 확인 가능(숨은 모델 검증이 덜함)

`ModelSerializer`를 쓰면 이런 검증들이 자동으로 구성된다. `Serializer`를 쓰면 검증 규칙을 직접 명시해야 한다.

### 예시: unique 제약

모델에 `unique=True`가 있는 경우:

    class CustomerReportRecord(models.Model):
        time_raised = models.DateTimeField(default=timezone.now, editable=False)
        reference = models.CharField(unique=True, max_length=20)
        description = models.TextField()

`ModelSerializer`를 만들고 shell에서 `repr`를 찍으면:

    >>> serializer = CustomerReportSerializer()
    >>> print(repr(serializer))
    CustomerReportSerializer():
        id = IntegerField(label='ID', read_only=True)
        time_raised = DateTimeField(read_only=True)
        reference = CharField(max_length=20, validators=[UniqueValidator(queryset=CustomerReportRecord.objects.all())])
        description = CharField(style={'type': 'textarea'})

여기서 핵심은 `reference`에 **UniqueValidator가 명시적으로 붙어 있다는 점**.

---

## UniqueValidator

모델 필드의 `unique=True` 제약을 강제하는 validator.

- `queryset` (**필수**) : 중복 검사 대상
- `message` : 실패 시 메시지
- `lookup` : 조회 lookup(기본 `'exact'`)

**적용 위치:** *serializer field*

    from rest_framework.validators import UniqueValidator

    slug = SlugField(
        max_length=100,
        validators=[UniqueValidator(queryset=BlogPost.objects.all())]
    )

## UniqueTogetherValidator

모델의 `unique_together` 제약을 강제.

- `queryset` (**필수**)
- `fields` (**필수**) : serializer에 존재하는 필드 이름 목록
- `message`

**적용 위치:** *serializer class* (`Meta.validators`)

    from rest_framework.validators import UniqueTogetherValidator

    class ExampleSerializer(serializers.Serializer):
        class Meta:
            validators = [
                UniqueTogetherValidator(
                    queryset=ToDoItem.objects.all(),
                    fields=['list', 'position']
                )
            ]

**주의:** `UniqueTogetherValidator`는 적용된 필드들을 **암묵적으로 required 취급**한다.  
다만 `default`가 있는 필드는 예외(항상 값이 공급되므로).

---

## UniqueForDate / Month / Year Validator

`unique_for_date`, `unique_for_month`, `unique_for_year` 제약을 강제.

- `queryset` (**필수**)
- `field` (**필수**) : 유니크를 검사할 필드(serializer에 있어야 함)
- `date_field` (**필수**) : 기간 기준이 되는 날짜 필드(serializer에 있어야 함)
- `message`

**적용 위치:** *serializer class* (`Meta.validators`)

    from rest_framework.validators import UniqueForYearValidator

    class ExampleSerializer(serializers.Serializer):
        class Meta:
            validators = [
                UniqueForYearValidator(
                    queryset=BlogPostItem.objects.all(),
                    field='slug',
                    date_field='published'
                )
            ]

`date_field`는 검증 시점에 반드시 값이 있어야 하므로, 모델 default에만 의존하면 안 된다(검증이 먼저 실행됨). 보통 아래 셋 중 하나로 처리한다:

- **Writable date field**: `published = serializers.DateTimeField(required=True)` 또는 `default` 지정
- **Read-only but visible**: `published = serializers.DateTimeField(read_only=True, default=timezone.now)`
- **Hidden**: `published = serializers.HiddenField(default=timezone.now)`

> 참고: `HiddenField()`는 `partial=True`(PATCH)에서 나타나지 않는다.

---

# Advanced field defaults

여러 필드에 걸친 validator는 “클라이언트가 보내면 안 되지만 검증에는 필요한 값”이 필요한 경우가 있다. 그럴 때 `HiddenField`를 쓰면:

- `validated_data`에는 포함되지만
- 응답 출력에는 포함되지 않는다.

또한 `read_only=True` 필드는 writable 필드에서 제외되므로 `default=...`가 적용되지 않는 동작 변화가 있다(관련 공지 링크 참고).

## CurrentUserDefault

현재 유저를 기본값으로 넣고 싶을 때. serializer를 만들 때 `context={'request': request}`가 필요.

    owner = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )

## CreateOnlyDefault

create 때만 default 적용, update 때는 필드를 omit.

    created_at = serializers.DateTimeField(
        default=serializers.CreateOnlyDefault(timezone.now)
    )

---

# Validators의 한계 / 언제 직접 validate()를 쓰나

## Optional fields + unique together

기본 unique together 검증은 필드를 required로 보려는 성향이 있어서,
특정 필드에 `required=False`를 주고 싶으면 동작이 애매해질 수 있다.

이럴 땐 기본 validator를 끄고 직접 구현한다:

    class BillingRecordSerializer(serializers.ModelSerializer):
        def validate(self, attrs):
            # 커스텀 검증

        class Meta:
            extra_kwargs = {'client': {'required': False}}
            validators = []  # 기본 unique_together 제거

## Nested serializer 업데이트

update 시 유니크 검증은 “현재 인스턴스는 중복 검사에서 제외”해야 하는데,
nested update에서는 그 인스턴스 컨텍스트를 제대로 전달하기 어려울 수 있다.
이 경우도 기본 validator 제거 + `.validate()` 또는 view에서 직접 처리하는 편이 낫다.

## 복잡할 땐 repr 찍어보기

    >>> serializer = MyComplexModelSerializer()
    >>> print(serializer)

자동 생성된 필드/validator를 눈으로 확인하는 게 가장 빠른 디버깅 방법이다.

---

# Custom validators 작성

## 함수형

실패 시 `serializers.ValidationError`를 raise 하면 된다.

    def even_number(value):
        if value % 2 != 0:
            raise serializers.ValidationError('This field must be an even number.')

## 클래스형

`__call__`로 구현. 파라미터화/재사용에 유리.

    class MultipleOf:
        def __init__(self, base):
            self.base = base

        def __call__(self, value):
            if value % self.base != 0:
                raise serializers.ValidationError(
                    'This field must be a multiple of %d.' % self.base
                )

### context 접근이 필요할 때

validator에 `requires_context = True`를 두면 `__call__`이 추가 인자를 받는다.

    class MultipleOf:
        requires_context = True

        def __call__(self, value, serializer_field):
            ...

[cite]: https://docs.djangoproject.com/en/stable/ref/validators/
