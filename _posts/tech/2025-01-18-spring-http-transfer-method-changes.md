---
layout: post
title: Spring RestClient/RestTemplate 요청이 실패하는 이유 - 데이터 타입에 따른 전송 방식 차이
description: >
  Spring Framework 6.1에서 RestClient/RestTemplate의 요청 본문 전송 방식이 변경되었습니다.
  데이터 타입에 따른 전송 방식 차이와 Chunked Transfer Encoding 관련 문제 해결 방법을 알아봅니다.
image: /assets/img/blog/common/spring.png
category: Tech
tags: [ Spring ]
---

* toc
{:toc}

## 요약

Spring 6.1부터 RestTemplate(이하 생략)과 RestClient의 요청 본문 처리 방식이 데이터 타입에 따라 달라졌습니다.
`byte[]`나 `String` 타입은 기존처럼 Content-Length 헤더와 함께 전송되지만,
`Object` 타입의 경우 Chunked Transfer Encoding을 사용하여 Streaming 방식으로 전송됩니다.
이러한 변경으로 인해 Chunked Transfer Encoding을 지원하지 않는 일부 API에서 호환성 문제가 발생할 수 있어 주의가 필요합니다.

## 문제 상황

Spring 6의 RestClient를 사용하여 외부 API를 호출하는 과정에서 문제가 발생했습니다.
코드 상으로는 문제가 없어 보였으나 계속해서 500 응답을 받았고, 에러 메시지에 상세한 설명이 없어 원인 파악이 어려웠습니다.

문제가 된 코드는 다음과 같습니다.

```java
// 500 에러가 발생하는 코드
public void callApi(TestObj testObj) {
    restClient.post()
        .uri("/api/...")
        .body(testObj)
        .retrieve()
        .toBodilessEntity();
}
```

Postman으로 요청을 보냈을 때 정상적으로 응답을 받았기에 코드 레벨에서 문제 해결을 위한 여러 시도를 해보았습니다.
그러다 Object를 직접 JSON String으로 직렬화하여 전송했을 때 정상적으로 동작하는 것을 확인했습니다.

```java
// 정상 동작하는 코드
public void callApi(TestObj testObj) {
    String jsonBody = objectMapper.writeValueAsString(testObj);
    restClient.post()
        .uri("/api/...")
        .body(jsonBody) // String으로 변환하여 전달
        .retrieve()
        .toBodilessEntity();
}
```

## 패킷 확인

저는 위와 같은 이유로 'RestClient 내부적으로 Object 직렬화가 잘못되고 있는 건가?' 라는 의심을 하게 되었습니다. 
그래서 Object가 어떻게 직렬화되어 외부로 보내지고 있는지 두 눈으로 직접 확인해 보고자 Wireshark로 패킷을 캡쳐해보았습니다.

- 실패 케이스의 HTTP Message

```text
Hypertext Transfer Protocol, has 2 chunks (including last chunk)
    POST / HTTP/1.1\r\n
    Connection: Upgrade, HTTP2-Settings\r\n
    Host: localhost:8080\r\n
    HTTP2-Settings: AAEAAEAAAAIAAAAAAAMAAAAAAAQBAAAAAAUAAEAAAAYABgAA\r\n
    Transfer-encoding: chunked\r\n
    Upgrade: h2c\r\n
    User-Agent: Java-http-client/21.0.5\r\n
    Content-Type: application/json\r\n
    \r\n
    [Full request URI: http://localhost:8080/]
    HTTP chunked response
        Data chunk (43 octets)
            Chunk size: 43 octets
            Chunk data: 7b226e616d65223a226e696c67696c222c22616765223a39392c22686f626279223a2274656e6e6973227d
            Chunk boundary: 0d0a
        End of chunked encoding
            Chunk size: 0 octets
        \r\n
    File Data: 43 bytes
```

- 성공 케이스의 HTTP Message

```text
Hypertext Transfer Protocol
    POST / HTTP/1.1\r\n
    Connection: Upgrade, HTTP2-Settings\r\n
    Content-Length: 43\r\n
    Host: localhost:8080\r\n
    HTTP2-Settings: AAEAAEAAAAIAAAAAAAMAAAAAAAQBAAAAAAUAAEAAAAYABgAA\r\n
    Upgrade: h2c\r\n
    User-Agent: Java-http-client/21.0.5\r\n
    Content-Type: application/json\r\n
    \r\n
    [Response in frame: 205178]
    [Full request URI: http://localhost:8080/]
    File Data: 43 bytes
```

위 두 결과를 비교해 보니 예상하지 못한 부분에서 차이점이 명확히 보였습니다.

바로 Chunked Transfer Encoding 입니다. 
실패했던 케이스는 요청 본문이 Streaming 방식으로 쪼개어 전송된 경우였고,
이를 통해 'API 서버가 Chunked Transfer Encoding을 지원하지 않아 발생한 문제'로 원인을 특정할 수 있었습니다.

## Chunked Transfer Encoding 란?

Chunked Transfer Encoding은 HTTP/1.1에서 도입된 전송 방식으로, 데이터를 청크(chunk) 단위로 나누어 순차적으로 전송하는 방식입니다.

자세한 내용은 링크로 대체합니다.
[rfc7230#section-4.1](https://datatracker.ietf.org/doc/html/rfc7230#section-4.1)

## RestClient가 청크 전송을 사용하는 경우는?

RestClient는 기본적으로 JdkClientHttpRequestFactory를 사용하며, 이때 사용되는 Http1Request의 `headers()` 구현을 살펴보면,
Content-Length가 0보다 작을 경우(-1은 크기를 알 수 없다는 의미) 헤더에 `Transfer-encoding: chunked`를 추가하여 청크 전송을 수행합니다.

```java
// jdk.internal.net.http.Http1Request
List<ByteBuffer> headers() {
    // ...
    if (requestPublisher != null) {
        contentLength = requestPublisher.contentLength();
        if (contentLength == 0) {
            systemHeadersBuilder.setHeader("Content-Length", "0");
        } else if (contentLength > 0) {
            systemHeadersBuilder.setHeader("Content-Length", Long.toString(contentLength));
            streaming = false;
        } else {
            streaming = true;
            systemHeadersBuilder.setHeader("Transfer-encoding", "chunked");
        }
    }
    // ...
}
```

SimpleClientHttpRequestFactory에서도 마찬가지로, SimpleClientHttpRequest의 executeInternal 메소드를 살펴보면
Content-Length가 0보다 작을 경우 청크 전송을 수행하도록 구현되어 있습니다.

```java
// org.springframework.http.client.SimpleClientHttpRequest
protected ClientHttpResponse executeInternal(HttpHeaders headers, @Nullable Body body) throws IOException {
    if (this.connection.getDoOutput()) {
        long contentLength = headers.getContentLength();
        if (contentLength >= 0) {
            this.connection.setFixedLengthStreamingMode(contentLength);
        } else {
            this.connection.setChunkedStreamingMode(this.chunkSize);
        }
    }
    // ...
}
```

이처럼 ClientHttpRequestFactory의 여러 구현체들이 Content-Length의 유무에 따라 청크 전송을 수행하는 것을 확인할 수 있었습니다.

## MessageConverter

그렇다면 RestClient에서 `byte[]`나 `String`을 body에 담을 때는 청크 전송이 되지 않고,
`Object`를 body에 담을 때는 청크 전송이 발생한 이유가 무엇일까요?
이를 이해하기 위해서는 RestClient에 등록되어 사용되는 MessageConverter에 대해 살펴볼 필요가 있습니다.

MessageConverter는 HTTP 요청/응답의 본문(body)을 자바 객체로 변환하거나, 자바 객체를 HTTP 응답 본문으로 변환하는 역할을 수행합니다.
RestClient에는 기본적으로 5개의 MessageConverter가 등록되어 있습니다.

- `ByteArrayHttpMessageConverter`
- `StringHttpMessageConverter`
- `ResourceHttpMessageConverter`
- `AllEncompassingFormHttpMessageConverter`
- `MappingJackson2HttpMessageConverter`

MessageConverter들은 각각 지원하는 Media Type과 Class Type이 정의되어 있어서,
header의 Content-Type 값과 body에 담긴 데이터의 타입에 따라 적절한 컨버터가 선택됩니다.

`byte[]`를 body에 담을 경우에는 ByteArrayHttpMessageConverter가,
`String`을 body에 담을 경우에는 StringHttpMessageConverter가 사용됩니다.
이들의 내부 구현을 살펴보면, `getContentLength()` 메소드를 Override하여 컨텐츠 길이를 제공하고 있음을 확인할 수 있습니다.

```java
// org.springframework.http.converter.ByteArrayHttpMessageConverter
@Override
protected Long getContentLength(byte[] bytes, @Nullable MediaType contentType) {
    return (long)bytes.length;
}
```

```java
// org.springframework.http.converter.StringHttpMessageConverter
@Override
protected Long getContentLength(String str, @Nullable MediaType contentType) {
    Charset charset = getContentTypeCharset(contentType);
    return (long)str.getBytes(charset).length;
}
```

`Object`를 body에 담는 경우에는 MappingJackson2HttpMessageConverter가 사용되는데,
이 컨버터는 `getContentLength()` 메소드를 별도로 Override하지 않고 있습니다.
따라서 Abstract 클래스에서 구현된 `null`을 반환하는 메소드를 그대로 사용하게 됩니다.

```java
// org.springframework.http.converter.AbstractHttpMessageConverter
@Nullable
protected Long getContentLength(T t, @Nullable MediaType contentType) throws IOException {
    return null;
}
```

앞서 설명한 것처럼 여러 ClientHttpRequestFactory들은 Content-Length가 없는 경우 청크 전송을 수행하게 됩니다.
이러한 이유들로 인해 RestClient를 사용할 때 body에 어떤 타입이 담겼는지에 따라 다른 전송 방식이 사용되는 것입니다.

## Spring Framework 6.1 Release Notes

사실 이렇게 동작하게 된건 얼마 되지 않았습니다. 이전에는 요청/응답 본문을 버퍼에 모아서 한 번에 전송하는 것이 기본 동작이었습니다.
그러나 Spring Framework 6.1 릴리즈에서 메모리 최적화를 위해 ClientHttpRequestFactory 구현체들의 기본 동작 방식을 변경한 것입니다.

[Spring Framework 6.1 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes#web-applications)

릴리즈 노트 중 해당 내용은 다음과 같습니다.

```text
To reduce memory usage in RestClient and RestTemplate, most ClientHttpRequestFactory implementations no longer buffer
request bodies before sending them to the server. As a result, for certain content types such as JSON, the contents size
is no longer known, and a Content-Length header is no longer set. If you would like to buffer request bodies like
before, simply wrap the ClientHttpRequestFactory you are using in a BufferingClientHttpRequestFactory.
```

Object 타입을 전송할 때는 객체의 최종 크기를 미리 예측하기 어렵고, 직렬화 과정에서 예상보다 큰 JSON 데이터가 생성될 수 있습니다.
이전처럼 버퍼에 모든 데이터를 담아 전송하는 Buffering 방식을 사용할 경우, 상황에 따라 메모리 사용량이 기하급수적으로 증가할 위험이 있습니다.

그러나 Streaming 방식에서는 직렬화와 전송이 동시에 진행되므로, 대용량 데이터 처리 시에도 메모리를 효율적으로 사용할 수 있습니다.
스프링 측에서는 이러한 장점을 활용하기 위해 기본 동작 방식을 Buffering 방식에서 Streaming 방식으로 변경한 것입니다.

## 해결 방법

그렇다면 현 상황을 해결하기 위해 어떤 방법을 시도할 수 있을까요?

API 서버에 청크 전송 지원을 요청하는 것이 가장 근본적인 해결책이 될 수 있지만, 현실적으로 어려울 것 같습니다.
따라서 클라이언트 측에서 청크 전송을 사용하지 않는 방향으로 해결 방법을 찾아야 합니다.

### 직접 직렬화

가장 간단한 방법으로는 body에 담을 때 직접 `byte[]`나 `String`으로 직렬화하여 담는 방법입니다.

### MappingJackson2HttpMessageConverter 커스텀

MappingJackson2HttpMessageConverter를 상속받아 getContentLength()를 Override하는 커스텀 컨버터를 만들고,
이를 RestClient에 등록된 MappingJackson2HttpMessageConverter와 교체하면 됩니다.

```java
// 커스텀 컨버터
public class FixedLengthJsonMessageConverter extends MappingJackson2HttpMessageConverter {

    @Override
    protected Long getContentLength(Object object, MediaType contentType) throws IOException {
        return calculateSize(object);
    }

    private long calculateSize(Object value) throws IOException {
        CountingOutputStream countingStream = new CountingOutputStream(NullOutputStream.INSTANCE);
        try (JsonGenerator generator = getObjectMapper().getFactory().createGenerator(countingStream)) {
            getObjectMapper().writeValue(generator, value);
        }
        return countingStream.getCount();
    }
}

// RestClient에 등록
public RestClient noChunkedClient() {
    return RestClient.builder()
        .messageConverters(converters -> {
            converters.removeIf(converter -> converter instanceof MappingJackson2HttpMessageConverter);
            converters.add(new ContentLengthJsonConverter());
        }).build();
}
```

### BufferingClientHttpRequestFactory 사용

Release Note에서는 버퍼링이 필요한 경우 이 팩토리를 사용하도록 제안했습니다.
하지만 이 방식은 모든 요청/응답 스트림을 메모리에서 버퍼링하기 때문에, 상황에 따라 메모리 사용량이 크게 증가할 수 있습니다.

따라서 이 방식을 적용하기 전에는 충분한 검토가 필요합니다.
단순히 청크 전송을 피하기 위해 요청/응답 버퍼를 사용하는 것은 메모리 관리 측면에서 우려되는 부분이 있습니다.

### 스프링 6.0 버전 사용

6.1 이전 방식처럼 요청 본문을 버퍼에 담아 한 번에 전송하는 방식으로 돌아갈 수는 있습니다.
하지만 단순히 청크 전송을 피하기 위해 Spring Framework 버전을 다운그레이드하는 것은 득보다 실이 더 큰 선택이 될 수 있습니다.

이는 최신 버전에서 제공하는 성능 개선사항이나 보안 패치 등 다양한 이점을 포기해야 하므로, 장기적인 관점에서 바람직하지 않은 해결책으로 보입니다.

## 결론

RestClient를 사용할 때 body에 담긴 데이터 타입에 따라 데이터 전송 방식이 달라진다는 것을 이해하고 있어야 합니다.
만약 통신 과정에서 알 수 없는 오류가 발생한다면, 상대방 시스템이 Chunked Transfer Encoding을 지원하는지 여부를 확인해야 합니다.
지원하지 않는 것으로 확인된다면, 상황에 맞게 앞서 설명한 해결 방법들 중 하나를 선택하여 적용하면 됩니다.

## 참고 자료

- [Spring Framework 6.1 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes#web-applications)
- [HTTP/1.1 Specification-Chunked Transfer Encoding](https://tools.ietf.org/html/rfc7230#section-4.1)
- [Spring RestClient Documentation](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
