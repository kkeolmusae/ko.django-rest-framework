---
source:
    - relations.py
---

# 시리얼라이저 관계 (Serializer relations)

> 데이터 구조(data structures)가 알고리즘(algorithms)보다 프로그래밍의 중심이다.
>
> &mdash; [Rob Pike][cite]

관계 필드(Relational fields)는 모델 간의 관계를 표현하는 데 사용된다.  
`ForeignKey`, `ManyToManyField`, `OneToOneField` 관계뿐 아니라 **역방향(reverse) 관계**, 그리고 `GenericForeignKey` 같은 **커스텀 관계**에도 적용할 수 있다.

---

**참고:** 관계 필드는 `relations.py`에 선언되어 있지만, 관례적으로  
`from rest_framework import serializers`로 import 한 뒤  
`serializers.<FieldName>` 형태로 사용하는 것이 권장된다.

---

---

**참고:** REST Framework는 `select_related` / `prefetch_related` 관점에서 시리얼라이저에 전달된 queryset을 자동 최적화하지 않는다. 너무 “마법”이 많아지기 때문이다.  
`source` 속성으로 ORM 관계를 가로지르는 필드를 가진 시리얼라이저는, 관련 객체를 가져오기 위해 추가 DB 조회가 발생할 수 있다. 이런 추가 조회를 피하도록 쿼리를 최적화하는 책임은 개발자에게 있다.

예를 들어, 아래 시리얼라이저는 `tracks`가 prefetch 되어 있지 않다면 `tracks` 평가 시마다 DB hit가 발생할 수 있다:

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = serializers.SlugRelatedField(
            many=True,
            read_only=True,
            slug_field='title'
        )

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

    # 각 album 객체에 대해 tracks를 DB에서 가져와야 함
    qs = Album.objects.all()
    print(AlbumSerializer(qs, many=True).data)

`AlbumSerializer`를 `many=True`로 큰 queryset에 적용하면 심각한 성능 문제가 될 수 있다.  
다음처럼 queryset을 최적화하면 문제를 해결할 수 있다:

    qs = Album.objects.prefetch_related('tracks')
    # 추가 DB hit 없이 처리 가능
    print(AlbumSerializer(qs, many=True).data)

---

#### 관계(relationships) 살펴보기

`ModelSerializer`를 사용하면 필드와 관계가 자동 생성된다. 자동 생성된 필드를 확인하면 관계 표현 방식을 어떻게 커스터마이즈할지 결정하는 데 도움이 된다.

Django shell에서 다음처럼 확인할 수 있다:

    >>> from myapp.serializers import AccountSerializer
    >>> serializer = AccountSerializer()
    >>> print(repr(serializer))
    AccountSerializer():
        id = IntegerField(label='ID', read_only=True)
        name = CharField(allow_blank=True, max_length=100, required=False)
        owner = PrimaryKeyRelatedField(queryset=User.objects.all())

# API Reference

관계 필드 유형을 설명하기 위해, 음악 앨범과 앨범의 트랙 모델을 예시로 사용한다.

    class Album(models.Model):
        album_name = models.CharField(max_length=100)
        artist = models.CharField(max_length=100)

    class Track(models.Model):
        album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
        order = models.IntegerField()
        title = models.CharField(max_length=100)
        duration = models.IntegerField()

        class Meta:
            unique_together = ['album', 'order']
            ordering = ['order']

        def __str__(self):
            return '%d: %s' % (self.order, self.title)

## StringRelatedField

`StringRelatedField`는 대상 객체를 그 객체의 `__str__` 메서드로 표현한다.

예:

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = serializers.StringRelatedField(many=True)

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

직렬화 결과 예:

    {
        'album_name': 'Things We Lost In The Fire',
        'artist': 'Low',
        'tracks': [
            '1: Sunflower',
            '2: Whitetail',
            '3: Dinosaur Act',
            ...
        ]
    }

이 필드는 읽기 전용이다.

**인자(Arguments)**:

* `many` - to-many 관계에 적용하는 경우 `True`로 설정한다.

## PrimaryKeyRelatedField

`PrimaryKeyRelatedField`는 대상 객체를 기본키(primary key)로 표현한다.

예:

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

직렬화 결과 예:

    {
        'album_name': 'Undun',
        'artist': 'The Roots',
        'tracks': [
            89,
            90,
            91,
            ...
        ]
    }

기본적으로 이 필드는 읽기/쓰기 가능이지만, `read_only`로 동작을 바꿀 수 있다.

**인자(Arguments)**:

* `queryset` - 입력 검증 시 모델 인스턴스 조회에 사용할 queryset. 관계 필드는 `queryset`을 명시하거나 `read_only=True`를 설정해야 한다.
* `many` - to-many 관계에 적용하는 경우 `True`.
* `allow_null` - nullable 관계에서 `None` 또는 빈 문자열을 허용할지 여부. 기본 `False`.
* `pk_field` - PK 값의 직렬화/역직렬화 방식을 제어할 필드. 예: `pk_field=UUIDField(format='hex')`.

## HyperlinkedRelatedField

`HyperlinkedRelatedField`는 대상 객체를 하이퍼링크(URL)로 표현한다.

예:

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = serializers.HyperlinkedRelatedField(
            many=True,
            read_only=True,
            view_name='track-detail'
        )

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

직렬화 결과 예:

    {
        'album_name': 'Graceland',
        'artist': 'Paul Simon',
        'tracks': [
            'http://www.example.com/api/tracks/45/',
            'http://www.example.com/api/tracks/46/',
            'http://www.example.com/api/tracks/47/',
            ...
        ]
    }

기본적으로 이 필드는 읽기/쓰기 가능이지만 `read_only`로 바꿀 수 있다.

---

**참고**: 이 필드는 `lookup_field`, `lookup_url_kwarg`로 지정된 단일 URL 파라미터(예: pk 또는 slug)를 받는 URL에 매핑되는 객체를 대상으로 설계되었다.  
더 복잡한 형태의 하이퍼링크 표현이 필요하다면 아래의 [custom hyperlinked fields](#custom-hyperlinked-fields) 섹션처럼 커스터마이즈해야 한다.

---

**인자(Arguments)**:

* `view_name` - 관계 대상 뷰 이름. [표준 router][routers]를 쓰면 `<modelname>-detail` 형태. **필수**
* `queryset` - 입력 검증 시 모델 인스턴스 조회에 사용할 queryset. `queryset` 또는 `read_only=True` 중 하나는 필요.
* `many` - to-many 관계에 적용하는 경우 `True`.
* `allow_null` - nullable 관계에서 `None` 또는 빈 문자열 허용 여부. 기본 `False`.
* `lookup_field` - 조회에 사용할 대상 필드. 기본 `'pk'`.
* `lookup_url_kwarg` - URLconf에서 lookup_field에 대응하는 kwarg 이름. 기본적으로 `lookup_field`와 동일.
* `format` - format suffix를 사용하는 경우, 대상 링크에도 동일 format을 적용(필요 시 `format` 인자로 override 가능).

## SlugRelatedField

`SlugRelatedField`는 대상 객체를 대상의 특정 필드 값(slug)으로 표현한다.

예:

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = serializers.SlugRelatedField(
            many=True,
            read_only=True,
            slug_field='title'
         )

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

직렬화 결과 예:

    {
        'album_name': 'Dear John',
        'artist': 'Loney Dear',
        'tracks': [
            'Airport Surroundings',
            'Everything Turns to You',
            'I Was Only Going Out',
            ...
        ]
    }

기본적으로 읽기/쓰기 가능이지만 `read_only`로 변경 가능하다.

쓰기 가능한 `SlugRelatedField`로 사용할 경우, 보통 `slug_field`는 `unique=True`인 모델 필드여야 한다.

**인자(Arguments)**:

* `slug_field` - 대상을 표현할 때 사용할 필드. 인스턴스를 유일하게 식별 가능한 필드여야 한다(예: `username`). **필수**
* `queryset` - 입력 검증 시 모델 인스턴스 조회에 사용할 queryset. `queryset` 또는 `read_only=True` 중 하나는 필요.
* `many` - to-many 관계에 적용하는 경우 `True`.
* `allow_null` - nullable 관계에서 `None` 또는 빈 문자열 허용 여부. 기본 `False`.

## HyperlinkedIdentityField

이 필드는 `HyperlinkedModelSerializer`의 `'url'` 필드처럼 **자기 자신(identity)** 을 하이퍼링크로 표현할 때 사용한다. 객체의 다른 속성에도 사용할 수 있다.

예:

    class AlbumSerializer(serializers.HyperlinkedModelSerializer):
        track_listing = serializers.HyperlinkedIdentityField(view_name='track-list')

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'track_listing']

직렬화 결과 예:

    {
        'album_name': 'The Eraser',
        'artist': 'Thom Yorke',
        'track_listing': 'http://www.example.com/api/track_list/12/',
    }

이 필드는 항상 읽기 전용이다.

**인자(Arguments)**:

* `view_name` - 관계 대상 뷰 이름. [표준 router][routers]를 쓰면 `<model_name>-detail` 형태. **필수**
* `lookup_field` - 조회에 사용할 대상 필드. 기본 `'pk'`.
* `lookup_url_kwarg` - URLconf에서 lookup_field에 대응하는 kwarg 이름. 기본 `lookup_field`와 동일.
* `format` - format suffix 사용 시 적용할 format(override 가능).

---

# 중첩 관계 (Nested relationships)

앞서 다룬 “다른 엔티티를 참조(reference)”하는 방식과 달리,  
참조 대상 엔티티를 객체 표현 안에 **임베드/중첩(nested)** 시킬 수도 있다.  
이런 중첩 관계는 **시리얼라이저를 필드로 사용**해 표현한다.

to-many 관계를 표현하려면 시리얼라이저 필드에 `many=True`를 추가한다.

## 예시

    class TrackSerializer(serializers.ModelSerializer):
        class Meta:
            model = Track
            fields = ['order', 'title', 'duration']

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = TrackSerializer(many=True, read_only=True)

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

중첩 표현 예:

    >>> album = Album.objects.create(album_name="The Gray Album", artist='Danger Mouse')
    >>> Track.objects.create(album=album, order=1, title='Public Service Announcement', duration=245)
    <Track: Track object>
    >>> Track.objects.create(album=album, order=2, title='What More Can I Say', duration=264)
    <Track: Track object>
    >>> Track.objects.create(album=album, order=3, title='Encore', duration=159)
    <Track: Track object>
    >>> serializer = AlbumSerializer(instance=album)
    >>> serializer.data
    {
        'album_name': 'The Gray Album',
        'artist': 'Danger Mouse',
        'tracks': [
            {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
            {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
            {'order': 3, 'title': 'Encore', 'duration': 159},
            ...
        ],
    }

## 쓰기 가능한 중첩 시리얼라이저 (Writable nested serializers)

기본적으로 중첩 시리얼라이저는 읽기 전용이다.  
중첩 필드에 대해 쓰기(create/update)를 지원하려면, 자식 관계를 어떻게 저장할지 명시하기 위해 `create()` 및/또는 `update()`를 구현해야 한다.

예:

    class TrackSerializer(serializers.ModelSerializer):
        class Meta:
            model = Track
            fields = ['order', 'title', 'duration']

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = TrackSerializer(many=True)

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

        def create(self, validated_data):
            tracks_data = validated_data.pop('tracks')
            album = Album.objects.create(**validated_data)
            for track_data in tracks_data:
                Track.objects.create(album=album, **track_data)
            return album

    >>> data = {
        'album_name': 'The Gray Album',
        'artist': 'Danger Mouse',
        'tracks': [
            {'order': 1, 'title': 'Public Service Announcement', 'duration': 245},
            {'order': 2, 'title': 'What More Can I Say', 'duration': 264},
            {'order': 3, 'title': 'Encore', 'duration': 159},
        ],
    }
    >>> serializer = AlbumSerializer(data=data)
    >>> serializer.is_valid()
    True
    >>> serializer.save()
    <Album: Album object>

---

# 커스텀 관계 필드 (Custom relational fields)

기존 관계 표현 방식이 요구사항에 맞지 않는 드문 경우,  
모델 인스턴스에서 출력 표현을 어떻게 생성할지 완전히 직접 정의하는 커스텀 관계 필드를 구현할 수 있다.

커스텀 관계 필드를 만들려면 `RelatedField`를 상속하고 `.to_representation(self, value)`를 구현한다.  
`value`는 보통 모델 인스턴스(관계 대상)이다.

읽기/쓰기 필드로 만들려면 [`.to_internal_value(self, data)`][to_internal_value]도 구현해야 한다.

`context` 기반으로 동적 queryset이 필요하다면, `.queryset`을 지정하는 대신 `.get_queryset(self)`를 override 할 수도 있다.

## 예시

트랙을 `order`, `title`, `duration`으로 구성된 문자열로 직렬화하는 관계 필드:

    import time

    class TrackListingField(serializers.RelatedField):
        def to_representation(self, value):
            duration = time.strftime('%M:%S', time.gmtime(value.duration))
            return 'Track %d: %s (%s)' % (value.order, value.name, duration)

    class AlbumSerializer(serializers.ModelSerializer):
        tracks = TrackListingField(many=True)

        class Meta:
            model = Album
            fields = ['album_name', 'artist', 'tracks']

직렬화 결과 예:

    {
        'album_name': 'Sometimes I Wish We Were an Eagle',
        'artist': 'Bill Callahan',
        'tracks': [
            'Track 1: Jim Cain (04:39)',
            'Track 2: Eid Ma Clack Shaw (04:19)',
            'Track 3: The Wind and the Dove (04:34)',
            ...
        ]
    }

---

# 커스텀 하이퍼링크 필드 (Custom hyperlinked fields)

단일 lookup 필드만으로 표현할 수 없는 URL(예: 여러 kwarg가 필요한 URL)을 표현하려면  
`HyperlinkedRelatedField`를 override 해서 동작을 커스터마이즈할 수 있다.

Override 가능한 메서드 2개:

**get_url(self, obj, view_name, request, format)**  
객체 인스턴스를 URL 표현으로 매핑한다.  
`view_name`, `lookup_field` 설정이 URLconf와 맞지 않으면 `NoReverseMatch`가 발생할 수 있다.

**get_object(self, view_name, view_args, view_kwargs)**  
쓰기 가능한 하이퍼링크 필드를 지원하려면, 들어온 URL을 다시 객체로 매핑해야 하므로 이 메서드를 override 한다.  
읽기 전용이면 override 필요 없다.  
URLconf 인자에 해당하는 객체를 반환해야 하며, `ObjectDoesNotExist`를 발생시킬 수 있다.

## 예시

고객(customer) URL이 다음처럼 두 개의 kwarg를 받는다고 하자:

    /api/<organization_slug>/customers/<customer_pk>/

기본 구현은 단일 lookup만 지원하므로, 다음처럼 override 해야 한다:

    from rest_framework import serializers
    from rest_framework.reverse import reverse

    class CustomerHyperlink(serializers.HyperlinkedRelatedField):
        # 클래스 속성으로 정의하면 인자로 매번 넘길 필요가 없다.
        view_name = 'customer-detail'
        queryset = Customer.objects.all()

        def get_url(self, obj, view_name, request, format):
            url_kwargs = {
                'organization_slug': obj.organization.slug,
                'customer_pk': obj.pk
            }
            return reverse(view_name, kwargs=url_kwargs, request=request, format=format)

        def get_object(self, view_name, view_args, view_kwargs):
            lookup_kwargs = {
               'organization__slug': view_kwargs['organization_slug'],
               'pk': view_kwargs['customer_pk']
            }
            return self.get_queryset().get(**lookup_kwargs)

이 스타일을 generic view와 함께 쓰려면, 뷰 쪽 `.get_object`도 lookup 로직에 맞게 override 해야 한다.

가능하면 API 표현은 “평평한(flat)” 스타일을 권장하지만, 중첩 URL 스타일도 과하지 않게 쓰면 괜찮다.

---

# 추가 메모 (Further notes)

## `queryset` 인자

`queryset` 인자는 *쓰기 가능한(writable)* 관계 필드에서만 필요하다.  
이때 사용자 입력(원시 값)을 모델 인스턴스로 매핑하기 위한 조회에 사용된다.

2.x에서는 `ModelSerializer` 사용 시 일부 상황에서 자동으로 `queryset`을 추정할 수 있었다.  
지금은 쓰기 가능한 관계 필드에서는 **항상 명시적인 `queryset`** 을 사용하도록 바뀌었다.

이렇게 하면 `ModelSerializer`의 숨은 “마법”이 줄어들고, 필드 동작이 명확해지며,  
`ModelSerializer`와 완전한 `Serializer` 사이를 옮기기도 쉬워진다.

## HTML 표시 커스터마이즈

Browsable API의 `<select>` 입력에서 사용할 `choices`의 문자열 표현은 모델의 `__str__`을 사용한다.

표시 문자열을 커스터마이즈하려면 `RelatedField` 서브클래스의 `display_value()`를 override 한다:

    class TrackPrimaryKeyRelatedField(serializers.PrimaryKeyRelatedField):
        def display_value(self, instance):
            return 'Track: %s' % (instance.title)

## 선택 필드 컷오프 (Select field cutoffs)

Browsable API에서 관계 필드는 기본적으로 최대 1000개까지만 선택 항목을 표시한다.  
더 많으면 비활성 옵션으로 `"More than 1000 items…"`가 표시된다.

이 동작은 너무 많은 관계 항목이 렌더링되어 템플릿이 느려지는 것을 방지하기 위한 것이다.

제어 가능한 인자:

* `html_cutoff` - 표시할 최대 choice 개수. 제한 비활성화는 `None`. 기본 `1000`.
* `html_cutoff_text` - 컷오프 발생 시 표시할 문구. 기본 `"More than {count} items…"`

전역 설정 `HTML_SELECT_CUTOFF`, `HTML_SELECT_CUTOFF_TEXT`로도 제어 가능하다.

컷오프가 걸리는 경우, `style`로 select 대신 일반 input을 쓰도록 할 수도 있다:

    assigned_to = serializers.SlugRelatedField(
       queryset=User.objects.all(),
       slug_field='username',
       style={'base_template': 'input.html'}
    )

## 역방향 관계 (Reverse relations)

역방향 관계는 `ModelSerializer`와 `HyperlinkedModelSerializer`에 의해 자동 포함되지 않는다.  
포함하려면 `fields`에 명시적으로 추가해야 한다.

예:

    class AlbumSerializer(serializers.ModelSerializer):
        class Meta:
            fields = ['tracks', ...]

보통 관계에 `related_name`을 지정해 해당 이름을 필드명으로 쓰는 것이 좋다:

    class Track(models.Model):
        album = models.ForeignKey(Album, related_name='tracks', on_delete=models.CASCADE)
        ...

`related_name`이 없다면 자동 생성된 이름을 써야 한다(예: `track_set`):

    class AlbumSerializer(serializers.ModelSerializer):
        class Meta:
            fields = ['track_set', ...]

자세한 내용은 [reverse relationships][reverse-relationships] Django 문서를 참고하자.

## 제네릭 관계 (Generic relationships)

GenericForeignKey를 직렬화하려면, 대상 표현 방식을 명확히 정하기 위해 커스텀 필드를 정의해야 한다.

예시 모델:

    class TaggedItem(models.Model):
        """
        제네릭 관계로 임의의 모델 인스턴스를 태깅한다.

        See: https://docs.djangoproject.com/en/stable/ref/contrib/contenttypes/
        """
        tag_name = models.SlugField()
        content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
        object_id = models.PositiveIntegerField()
        tagged_object = GenericForeignKey('content_type', 'object_id')

        def __str__(self):
            return self.tag_name

태그가 달릴 수 있는 두 모델:

    class Bookmark(models.Model):
        """
        Bookmark는 URL과 0개 이상의 태그로 구성된다.
        """
        url = models.URLField()
        tags = GenericRelation(TaggedItem)


    class Note(models.Model):
        """
        Note는 텍스트와 0개 이상의 태그로 구성된다.
        """
        text = models.CharField(max_length=1000)
        tags = GenericRelation(TaggedItem)

대상 타입에 따라 다르게 직렬화하는 커스텀 필드 예:

    class TaggedObjectRelatedField(serializers.RelatedField):
        """
        `tagged_object` 제네릭 관계를 위한 커스텀 필드.
        """

        def to_representation(self, value):
            """
            태그 대상 객체를 간단한 텍스트로 직렬화.
            """
            if isinstance(value, Bookmark):
                return 'Bookmark: ' + value.url
            elif isinstance(value, Note):
                return 'Note: ' + value.text
            raise Exception('Unexpected type of tagged object')

중첩 표현이 필요하면 `.to_representation()` 안에서 적절한 시리얼라이저를 사용할 수도 있다:

        def to_representation(self, value):
            """
            Bookmark는 BookmarkSerializer로,
            Note는 NoteSerializer로 직렬화.
            """
            if isinstance(value, Bookmark):
                serializer = BookmarkSerializer(value)
            elif isinstance(value, Note):
                serializer = NoteSerializer(value)
            else:
                raise Exception('Unexpected type of tagged object')

            return serializer.data

역방향 제네릭 키(`GenericRelation`)는 대상 타입이 항상 정해져 있으므로 일반 관계 필드를 사용할 수 있다.

자세한 내용은 [generic relations][generic-relations] Django 문서를 참고하자.

## Through 모델이 있는 ManyToManyField

`through` 모델이 지정된 `ManyToManyField`를 대상으로 하는 관계 필드는 기본적으로 읽기 전용으로 설정된다.

through 모델을 가진 `ManyToManyField`를 가리키는 관계 필드를 명시적으로 쓴다면,  
반드시 `read_only=True`로 설정하자.

through 모델의 [추가 필드][django-intermediary-manytomany]를 표현하려면,  
through 모델을 [중첩 객체][dealing-with-nested-objects]로 직렬화하는 방식을 고려할 수 있다.

---

# 서드파티 패키지 (Third Party Packages)

## DRF Nested Routers

[drf-nested-routers 패키지][drf-nested-routers]는 중첩 리소스(nested resources)를 다루기 위한 router 및 관계 필드를 제공한다.

## Rest Framework Generic Relations

[rest-framework-generic-relations][drf-nested-relations] 라이브러리는 GenericForeignKey에 대해 읽기/쓰기 직렬화를 제공한다.

[rest-framework-gm2m-relations][drf-gm2m-relations] 라이브러리는 [django-gm2m][django-gm2m-field]에 대해 읽기/쓰기 직렬화를 제공한다.

[cite]: http://users.ece.utexas.edu/~adnan/pike.html
[reverse-relationships]: https://docs.djangoproject.com/en/stable/topics/db/queries/#following-relationships-backward
[routers]: https://www.django-rest-framework.org/api-guide/routers#defaultrouter
[generic-relations]: https://docs.djangoproject.com/en/stable/ref/contrib/contenttypes/#id1
[drf-nested-routers]: https://github.com/alanjds/drf-nested-routers
[drf-nested-relations]: https://github.com/Ian-Foote/rest-framework-generic-relations
[drf-gm2m-relations]: https://github.com/mojtabaakbari221b/rest-framework-gm2m-relations
[django-gm2m-field]: https://github.com/tkhyn/django-gm2m
[django-intermediary-manytomany]: https://docs.djangoproject.com/en/stable/topics/db/models/#intermediary-manytomany
[dealing-with-nested-objects]: https://www.django-rest-framework.org/api-guide/serializers/#dealing-with-nested-objects
[to_internal_value]: https://www.django-rest-framework.org/api-guide/serializers/#to_internal_valueself-data
