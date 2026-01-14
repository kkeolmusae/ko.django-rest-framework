---
source:
    - fields.py
---

# 시리얼라이저 필드 (Serializer fields)

> Form 클래스의 각 필드는 데이터 검증뿐만 아니라  
> 데이터를 일관된 형식으로 “정제(cleaning)”하는 역할도 담당한다.
>
> &mdash; [Django 문서][cite]

시리얼라이저 필드는 **원시(primitives) 값 ↔ 내부 데이터 타입** 간의 변환을 처리한다.  
또한 입력 값에 대한 검증을 수행하고, 부모 객체로부터 값을 가져오거나 설정하는 역할도 담당한다.

---

**참고:** 시리얼라이저 필드는 `fields.py`에 선언되어 있지만, 관례적으로  
`from rest_framework import serializers` 형태로 import 한 뒤  
`serializers.<FieldName>` 형태로 사용하는 것이 권장된다.

---

## 핵심 인자 (Core arguments)

모든 시리얼라이저 필드 클래스의 생성자는 최소한 다음 인자들을 지원한다.  
일부 필드는 추가적인 전용 인자를 가질 수 있지만, 아래 인자들은 항상 사용 가능해야 한다.

### `read_only`

읽기 전용 필드는 API 출력에는 포함되지만, 생성(create)이나 수정(update) 시 입력값으로는 사용되지 않는다.  
입력 데이터에 `read_only` 필드가 포함되어 있더라도 무시된다.

`True`로 설정하면 직렬화 시에는 포함되지만, 역직렬화 시 인스턴스를 생성·수정하는 데는 사용되지 않는다.

기본값: `False`

### `write_only`

`True`로 설정하면 생성·수정 시 입력값으로는 사용되지만, 직렬화 결과에는 포함되지 않는다.

기본값: `False`

### `required`

기본적으로 역직렬화 시 해당 필드가 제공되지 않으면 오류가 발생한다.  
역직렬화 시 필수 입력값이 아니도록 하려면 `False`로 설정한다.

`False`로 설정하면, 직렬화 시에도 해당 속성이나 키가 존재하지 않을 경우 출력에서 생략된다.

기본값: `True`  
단, [ModelSerializer](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer)를 사용하는 경우,
`default`가 지정되었거나 모델 필드가 `blank=True` 또는 `null=True`인 경우 기본값은 `False`가 된다
(단, unique 제약 조건에 포함된 경우는 예외).

### `default`

입력값이 제공되지 않았을 때 사용할 기본값을 지정한다.  
설정하지 않으면 해당 속성은 아예 채워지지 않는다.

부분 업데이트(partial update)에서는 `default`가 적용되지 않는다.  
이 경우 입력으로 전달된 필드만 검증 결과에 포함된다.

함수나 callable을 지정할 수도 있으며, 이 경우 매번 호출된다.  
callable에 `requires_context = True` 속성이 있으면, serializer field 자체가 인자로 전달된다.

예시:

    class CurrentUserDefault:
        """
        serializer field의 default 값으로 사용 가능
        현재 사용자 반환
        """
        requires_context = True

        def __call__(self, serializer_field):
            return serializer_field.context['request'].user

직렬화 시 객체에 해당 속성이나 키가 없으면 `default` 값이 사용된다.

`default`를 지정하면 해당 필드는 자동으로 `required=False`가 된다.  
`default`와 `required`를 동시에 지정하는 것은 허용되지 않으며 오류가 발생한다.

### `allow_null`

기본적으로 `None` 값이 전달되면 오류가 발생한다.  
`True`로 설정하면 `None`을 유효한 값으로 허용한다.

명시적인 `default` 없이 `allow_null=True`를 설정하면,  
직렬화 출력에서 기본값이 `null`이 되지만 입력 역직렬화의 기본값을 의미하지는 않는다.

기본값: `False`

### `source`

필드를 채우는 데 사용할 속성 이름을 지정한다.  
`URLField(source='get_absolute_url')` 처럼 인자 없는 메서드일 수도 있고,  
`EmailField(source='user.email')` 처럼 점 표기법으로 속성을 탐색할 수도 있다.

점 표기법을 사용할 경우, 중간 객체가 없을 수 있으므로 `default` 설정이 필요할 수 있다.  
또한 ORM 관계를 통해 접근하는 경우 N+1 쿼리 문제가 발생할 수 있으므로  
`select_related`, `prefetch_related`를 적절히 사용해야 한다.  
자세한 내용은 [Django 문서][django-docs-select-related]를 참고하자.

`source='*'`는 **전체 객체를 필드로 전달**한다는 특별한 의미를 가진다.  
중첩 표현이나, 출력 생성을 위해 전체 객체가 필요한 필드에 유용하다.

기본값은 필드 이름과 동일하다.

### `validators`

입력값에 적용할 validator 함수 목록이다.  
보통 `serializers.ValidationError`를 발생시키며,  
Django의 `ValidationError`도 호환성을 위해 지원된다.

### `error_messages`

에러 코드 → 에러 메시지 매핑 딕셔너리.

### `label`

HTML 폼이나 설명 요소에서 사용될 필드의 짧은 이름.

### `help_text`

HTML 폼이나 설명 요소에서 사용될 필드 설명 텍스트.

### `initial`

HTML 폼에서 미리 채워질 초기값.

    import datetime
    from rest_framework import serializers

    class ExampleSerializer(serializers.Serializer):
        day = serializers.DateField(initial=datetime.date.today)

### `style`

렌더러가 필드를 어떻게 렌더링할지 제어하는 키-값 딕셔너리.

예시:

    # <input type="password">
    password = serializers.CharField(
        style={'input_type': 'password'}
    )

    # select 대신 radio input 사용
    color_channel = serializers.ChoiceField(
        choices=['red', 'green', 'blue'],
        style={'base_template': 'radio.html'}
    )

자세한 내용은 [HTML & Forms][html-and-forms] 문서를 참고하자.

---

# Boolean 필드

## BooleanField

불리언(Boolean) 표현 필드.

HTML 폼 입력에서는 값이 생략되면 항상 `False`로 처리된다.  
체크박스가 체크되지 않은 상태는 값이 전달되지 않기 때문이다.

Django 2.1부터 `models.BooleanField`에서 `blank` 인자가 제거되었다.  
이로 인해 Django 2.1 이후 기본 생성되는 `BooleanField`는 `required=True`와 동일하게 동작한다.  
이 동작을 제어하려면 시리얼라이저에서 명시적으로 선언하거나 `extra_kwargs`를 사용하자.

모델 대응: `django.db.models.fields.BooleanField`

**시그니처:** `BooleanField()`

---

# 문자열 필드

## CharField

문자열 표현 필드. `max_length`, `min_length` 검증 가능.

모델 대응: `CharField`, `TextField`

**시그니처:**  
`CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)`

* `max_length` – 최대 길이 검증
* `min_length` – 최소 길이 검증
* `allow_blank` – 빈 문자열 허용 여부 (기본 `False`)
* `trim_whitespace` – 앞뒤 공백 제거 여부 (기본 `True`)

문자열 필드에서는 `allow_null` 대신 `allow_blank` 사용을 권장한다.

## EmailField

유효한 이메일 주소인지 검증하는 문자열 필드.

모델 대응: `EmailField`

**시그니처:** `EmailField(max_length=None, min_length=None, allow_blank=False)`

## RegexField

정규식 패턴과 일치하는지 검증하는 문자열 필드.

**시그니처:** `RegexField(regex, max_length=None, min_length=None, allow_blank=False)`

## SlugField

`[a-zA-Z0-9_-]+` 패턴을 검증하는 `RegexField`.

**시그니처:** `SlugField(max_length=50, min_length=None, allow_blank=False)`

## URLField

URL 형식을 검증하는 필드.

**시그니처:** `URLField(max_length=200, min_length=None, allow_blank=False)`

## UUIDField

UUID 문자열을 검증하는 필드.

**시그니처:** `UUIDField(format='hex_verbose')`

---

# 숫자 필드

## IntegerField

정수 표현 필드.

**시그니처:** `IntegerField(max_value=None, min_value=None)`

## BigIntegerField

큰 정수 표현 필드.

**시그니처:** `BigIntegerField(max_value=None, min_value=None, coerce_to_string=None)`

## FloatField

부동소수점 표현 필드.

**시그니처:** `FloatField(max_value=None, min_value=None)`

## DecimalField

`Decimal` 기반의 소수 표현 필드.

**시그니처:**  
`DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)`

---

# 날짜 / 시간 필드

## DateTimeField

날짜 + 시간 표현 필드.

**시그니처:**  
`DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None, default_timezone=None)`

## DateField

날짜 표현 필드.

**시그니처:**  
`DateField(format=api_settings.DATE_FORMAT, input_formats=None)`

## TimeField

시간 표현 필드.

**시그니처:**  
`TimeField(format=api_settings.TIME_FORMAT, input_formats=None)`

## DurationField

`datetime.timedelta` 기반의 기간 필드.

**시그니처:**  
`DurationField(format=api_settings.DURATION_FORMAT, max_value=None, min_value=None)`

---

# 선택 필드

## ChoiceField

제한된 선택지 중 하나를 선택하는 필드.

**시그니처:** `ChoiceField(choices)`

## MultipleChoiceField

여러 개의 선택지를 허용하는 필드.

**시그니처:** `MultipleChoiceField(choices)`

---

# 파일 업로드 필드

## FileField

파일 표현 필드.

**시그니처:**  
`FileField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

## ImageField

이미지 파일 표현 필드.

**시그니처:**  
`ImageField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

---

# 복합 필드

## ListField

리스트를 검증하는 필드.

**시그니처:**  
`ListField(child=<FIELD>, allow_empty=True, min_length=None, max_length=None)`

## DictField

딕셔너리를 검증하는 필드.

**시그니처:**  
`DictField(child=<FIELD>, allow_empty=True)`

## JSONField

JSON 구조를 검증하는 필드.

**시그니처:**  
`JSONField(binary=False, encoder=None)`

---

# 기타 필드

## ReadOnlyField

값을 그대로 반환하는 읽기 전용 필드.

## HiddenField

사용자 입력이 아닌 default 값으로 채워지는 필드.

## ModelField

모델 필드에 위임하는 범용 필드.

## SerializerMethodField

시리얼라이저 메서드 호출 결과를 반환하는 읽기 전용 필드.

---

# 커스텀 필드

`Field`를 상속하고 `.to_representation()`, `.to_internal_value()`를 구현하면 된다.  
유효하지 않은 데이터는 `serializers.ValidationError`를 발생시켜야 한다.

---

# 서드파티 패키지

* DRF Compound Fields
* DRF Extra Fields
* djangorestframework-recursive
* django-rest-framework-gis
* django-rest-framework-hstore

[cite]: https://docs.djangoproject.com/en/stable/ref/forms/api/#django.forms.Form.cleaned_data
[html-and-forms]: ../topics/html-and-forms.md
[django-docs-select-related]: https://docs.djangoproject.com/en/stable/ref/models/querysets/#django.db.models.query.QuerySet.select_related
