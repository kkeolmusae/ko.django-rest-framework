---
source:
    - request.py
---

# Requests

> REST 기반 웹 서비스를 하고 있다면 ... `request.POST`는 무시해야 합니다.
>
> &mdash; Malcom Tredinnick, [Django developers group][cite]

REST framework의 `Request` 클래스는 표준 `HttpRequest`를 확장하여, REST framework의 유연한 요청 파싱과 요청 인증 기능을 추가로 제공합니다.

---

# Request 파싱

REST framework의 `Request` 객체는 유연한 요청 파싱 기능을 제공하여, JSON 데이터나 기타 미디어 타입의 요청을 일반적인 폼 데이터와 동일한 방식으로 다룰 수 있게 해줍니다.

## .data

`request.data`는 요청 본문(body)의 파싱된 내용을 반환합니다. 이는 표준 `request.POST` 및 `request.FILES` 속성과 유사하지만 다음과 같은 차이점이 있습니다.

* 파일 입력과 비파일 입력을 모두 포함한 **모든 파싱된 콘텐츠**를 포함합니다.
* `POST` 이외의 HTTP 메서드도 지원하므로 `PUT`, `PATCH` 요청의 본문 데이터에도 접근할 수 있습니다.
* 단순한 폼 데이터만이 아니라 REST framework의 **유연한 파서 시스템**을 지원합니다.  
  예를 들어, 들어오는 [JSON 데이터]를 [폼 데이터]를 다루는 것과 동일한 방식으로 처리할 수 있습니다.

자세한 내용은 [parsers 문서]를 참고하세요.

## .query_params

`request.query_params`는 `request.GET`의 의미를 더 명확히 표현한 동의어입니다.

코드의 가독성과 명확성을 위해 Django의 표준 `request.GET` 대신 `request.query_params`를 사용하는 것을 권장합니다.  
쿼리 파라미터는 `GET` 요청에만 존재하는 것이 아니라, 어떤 HTTP 메서드에도 포함될 수 있기 때문입니다.

## .parsers

`APIView` 클래스 또는 `@api_view` 데코레이터는 이 속성을 자동으로 설정합니다.  
이는 뷰에 지정된 `parser_classes` 또는 `DEFAULT_PARSER_CLASSES` 설정을 기반으로 한 `Parser` 인스턴스 목록입니다.

일반적으로 이 속성에 직접 접근할 필요는 없습니다.

---

**참고:**  
클라이언트가 잘못된 형식의 콘텐츠를 전송한 경우, `request.data`에 접근할 때 `ParseError`가 발생할 수 있습니다.  
기본적으로 REST framework의 `APIView` 클래스나 `@api_view` 데코레이터는 이 오류를 처리하여 `400 Bad Request` 응답을 반환합니다.

또한, 파싱할 수 없는 `Content-Type`으로 요청이 전송되면 `UnsupportedMediaType` 예외가 발생하며, 기본 동작으로 `415 Unsupported Media Type` 응답이 반환됩니다.

---

# Content negotiation

요청 객체는 콘텐츠 네고시에이션(content negotiation) 단계의 결과를 확인할 수 있는 몇 가지 속성을 제공합니다.  
이를 통해 미디어 타입에 따라 서로 다른 직렬화 방식을 선택하는 등의 동작을 구현할 수 있습니다.

## .accepted_renderer

콘텐츠 네고시에이션 단계에서 선택된 renderer 인스턴스입니다.

## .accepted_media_type

콘텐츠 네고시에이션 단계에서 허용(accepted)된 미디어 타입을 나타내는 문자열입니다.

---

# Authentication

REST framework는 요청 단위(per-request)로 유연한 인증을 제공하며, 다음과 같은 기능을 지원합니다.

* API의 서로 다른 영역에 대해 서로 다른 인증 정책 사용
* 여러 인증 정책을 동시에 지원
* 들어오는 요청과 연관된 사용자 정보 및 토큰 정보 제공

## .user

`request.user`는 일반적으로 `django.contrib.auth.models.User` 인스턴스를 반환하지만, 사용 중인 인증 정책에 따라 달라질 수 있습니다.

요청이 인증되지 않은 경우, 기본값은 `django.contrib.auth.models.AnonymousUser` 인스턴스입니다.

자세한 내용은 [authentication 문서]를 참고하세요.

## .auth

`request.auth`는 추가적인 인증 컨텍스트를 반환합니다.  
정확한 동작은 인증 정책에 따라 다르지만, 일반적으로 요청이 인증에 사용된 토큰 인스턴스일 수 있습니다.

요청이 인증되지 않았거나 추가 컨텍스트가 없는 경우, 기본값은 `None`입니다.

자세한 내용은 [authentication 문서]를 참고하세요.

## .authenticators

`APIView` 클래스 또는 `@api_view` 데코레이터는 이 속성을 자동으로 설정합니다.  
이는 뷰에 지정된 `authentication_classes` 또는 `DEFAULT_AUTHENTICATORS` 설정을 기반으로 한 `Authentication` 인스턴스 목록입니다.

일반적으로 이 속성에 직접 접근할 필요는 없습니다.

---

**참고:**  
`.user` 또는 `.auth` 속성에 접근할 때 `WrappedAttributeError`가 발생할 수 있습니다.  
이 오류는 인증기(authenticator) 내부에서 발생한 일반적인 `AttributeError`에서 비롯됩니다.

다만, 이 오류를 그대로 두면 바깥의 속성 접근 과정에서 무시될 수 있기 때문에, REST framework에서는 이를 다른 예외 타입으로 다시 발생시킵니다.  
Python은 해당 `AttributeError`가 인증기에서 발생한 것임을 인식하지 못하고, 요청 객체에 `.user` 또는 `.auth` 속성이 없는 것으로 잘못 판단할 수 있습니다.

이 경우, 인증기 구현 자체를 수정해야 합니다.

---

# Browser enhancements

REST framework는 브라우저 환경을 위한 몇 가지 확장 기능을 제공하며, 브라우저 기반의 `PUT`, `PATCH`, `DELETE` 폼을 지원합니다.

## .method

`request.method`는 요청의 HTTP 메서드를 **대문자 문자열**로 반환합니다.

브라우저 기반의 `PUT`, `PATCH`, `DELETE` 폼도 투명하게 지원됩니다.

자세한 내용은 [browser enhancements 문서]를 참고하세요.

## .content_type

`request.content_type`은 HTTP 요청 본문의 미디어 타입을 나타내는 문자열을 반환합니다.  
미디어 타입이 제공되지 않은 경우에는 빈 문자열을 반환합니다.

일반적으로는 REST framework의 기본 요청 파싱 동작에 의존하므로, 이 속성에 직접 접근할 필요는 없습니다.

만약 요청의 콘텐츠 타입에 직접 접근해야 한다면,  
`request.META.get('HTTP_CONTENT_TYPE')` 대신 `.content_type` 속성을 사용하는 것이 좋습니다.  
이는 브라우저 기반의 비폼(non-form) 콘텐츠도 투명하게 지원하기 때문입니다.

자세한 내용은 [browser enhancements 문서]를 참고하세요.

## .stream

`request.stream`은 요청 본문의 콘텐츠를 나타내는 스트림을 반환합니다.

일반적으로는 REST framework의 기본 요청 파싱 동작을 사용하므로, 요청 콘텐츠에 직접 접근할 필요는 없습니다.

---

# 표준 HttpRequest 속성

REST framework의 `Request`는 Django의 `HttpRequest`를 확장하므로,  
`request.META`, `request.session`과 같은 표준 속성과 메서드도 그대로 사용할 수 있습니다.

다만, 구현상의 이유로 `Request` 클래스는 `HttpRequest`를 상속(inherit)하지 않고,  
컴포지션(composition) 방식을 사용하여 확장되었습니다.

[cite]: https://groups.google.com/d/topic/django-developers/dxI4qVzrBY4/discussion
[parsers 문서]: parsers.md
[JSON 데이터]: parsers.md#jsonparser
[폼 데이터]: parsers.md#formparser
[authentication 문서]: authentication.md
[browser enhancements 문서]: ../topics/browser-enhancements.md
