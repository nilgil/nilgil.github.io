---
layout: post
title: 톰캣은 어떻게 트래픽을 인지하고 처리하는 걸까?
description: >
  톰캣의 NIO Connector가 어떻게 트래픽을 처리하는지 소스 코드 레벨에서 상세히 분석합니다. 
  Acceptor, Poller, Executor 등 핵심 컴포넌트들의 동작 원리와 소켓 통신 과정을 심층적으로 살펴봅니다.
image: /assets/img/blog/common/tomcat.png
category: Tech
tags: [ Server, Tomcat, Traffic, Deep-Dive ]
---

* toc
{:toc}

스프링 부트를 사용할 때 별다른 설정을 하지 않으면 자동으로 톰캣을 WAS로 사용합니다.
그리고 톰캣은 특정 포트로 들어오는 요청을 받아서 서블릿 컨테이너에 넘겨 우리가 만든 비즈니스 로직이 수행되도록 합니다.
그동안 이러한 흐름을 당연하게 여겨왔고, 개발하는 데 문제는 없었습니다.
그러나 서블릿 컨테이너에 넘어오기 전 앞단에서 서버의 특정 포트로 들어오는 요청을 톰캣이 어떻게 인지하고 처리하는지에 대해 궁금증이 생겼고,
이를 알아보기 위해 톰캣의 소스 코드를 분석하기로 했습니다. _(Tomcat 10.1.31 버전 기준)_

## Connector

톰캣에서는 앞단에서 요청을 수신하고, 파싱해서 서블릿 컨테이너에 전달하는 역할을 Connector라는 컴포넌트가 담당합니다.
톰캣에는 NIO Connector, NIO2 Connector, AJP Connector까지 총 3개의 Connector가 존재하지만,
오늘은 톰캣에서 기본값으로 사용되는 NIO Connector를 기준으로 알아보겠습니다.

NIO2 Connector는 대량의 비동기 처리가 필요한 경우, AJP Connector는 앞단에 Apache 서버를 사용하는 경우
최적화를 위해 검토해 볼 가치가 있으니 궁금한 경우 자세히 알아보시길 바랍니다.
참고로 NIO Connector의 처리량으로 충분한 경우가 대부분이라고 합니다.

![](/assets/img/blog/20241201/img-1.png){:.lead}

기본값이 NIO
{:.figcaption}

![](/assets/img/blog/20241201/img-2.png){:.lead}

해당 프로토콜로 Connector 생성
{:.figcaption}

만약 NIO2 Connector나 AJP Connector를 사용하고 싶다면 다음과 같이 프로토콜 값을 수정하여 적용할 수 있습니다.

```java

@Configuration
public class TomcatConfig {
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        // [AJP Protocol]
        // return (tomcat) -> tomcat.setProtocol("AJP/1.3");
        // return (tomcat) -> tomcat.setProtocol("org.apache.coyote.ajp.AjpNioProtocol");

        // [NIO2 Connector]
        return (tomcat) -> tomcat.setProtocol("org.apache.coyote.http11.Http11Nio2Protocol");
    }
}
```

## Endpoint

Connector를 생성하는 과정을 쭉 따라가 보면 해당 프로토콜 값으로 ProtocolHandler를 생성합니다.
Connector 구현체는 하나지만 각 Connector 종류에 따라 다른 ProtocolHandler를 가지게 됩니다.
NIO Connector의 경우 위 기본값으로 사용된 `org.apache.coyote.http11.Http11NioProtocol` 인스턴스를 만들어 ProtocolHandler로 사용합니다.

![](/assets/img/blog/20241201/img-3.png){:.lead}

ProtocolHandler 인스턴스 생성
{:.figcaption}

그런데 우리는 여기서 Http11NioProtocol의 생성자에서 생성하는 NioEndpoint 인스턴스에 주목해야 합니다.

![](/assets/img/blog/20241201/img-4.png){:.lead}

NioEndpoint 인스턴스 생성
{:.figcaption}

이 NioEndpoint가 실제로 네트워크 수준의 소켓 통신(연결 수락, 데이터 읽기/쓰기)을 관리하기 때문입니다.

그리고 이어서 Connector 생성 이후의 흐름을 쭉 따라가 보면 Connector가 포함된 TomcatWebServer를 시작하는 과정에서 Connector와 NioEndpoint의 초기화 작업이 이뤄집니다.

## ServerSocket 생성

NioEndpoint의 초기화 작업 중 initServerSocket()이라는 함수를 호출합니다.
이 함수는 내부적으로 JNI(Java Native Interface)를 사용하여 시스템 콜을 호출해
커널에 Socket을 생성하고, 해당 소켓을 서버 포트에 바인딩(Bind)시키고, Listen 상태로 만듭니다.

![](/assets/img/blog/20241201/img-5.png){:.lead}

JNI 호출하여 시스템 콜 socket 호출
{:.figcaption}

![](/assets/img/blog/20241201/img-6.png){:.lead}

![](/assets/img/blog/20241201/img-7.png){:.lead}

JNI 호출하여 시스템 콜 bind, listen 호출
{:.figcaption}

## Listening Socket

이렇게 생성된 소켓은 Listening Socket이라고 불립니다.
데이터 I/O를 위한 일반적인 소켓과는 다르게 톰캣이 종료될 때까지 존재하며, 커널에서 새로운 연결을 수립하는 용도로 사용되는 소켓입니다.
즉, NioEndpoint는 초기화 시점에 Listening Socket을 커널에 생성하고 이를 ServerSocket 타입의 필드로 매핑합니다.

![](/assets/img/blog/20241201/img-8.png){:.lead}

## Backlog Queue

Listening Socket에는 클라이언트의 연결 요청을 관리하는 Backlog Queue가 존재합니다.
내부적으로 SYN Queue와 Accept Queue로 이루어져 있으며, 각각 커널의 3way-handshake 처리 단계에서 사용됩니다.

![](/assets/img/blog/20241201/img-9.png){:.lead}

결과적으로 커널이 3way-handshake를 처리하여 수립된 연결은 Accept Queue에 쌓이게 됩니다.

## Acceptor

톰캣에는 Acceptor라고 하는 데몬 스레드가 있습니다.
이 Acceptor는 톰캣 실행 시점에 생성되어 종료될 때까지 백그라운드에서
Accept Queue에 쌓인 3way-handshake가 완료된 연결을 수락합니다.

![](/assets/img/blog/20241201/img-10.png){:.lead}

serverSock.accept()를 타고 들어가보면 JNI인 accept를 호출
{:.figcaption}

![](/assets/img/blog/20241201/img-11.png){:.lead}

JNI를 호출하여 시스템 콜 accept 호출
{:.figcaption}

accept 시스템 콜을 호출하면 커널은 해당 연결이 앞으로 데이터를 주고받는 데 사용할 소켓을 생성합니다.
이후 해당 연결은 타임아웃으로 만료되기 전까지는 3way-handshake를 다시 수행하지 않고 해당 소켓을 통해 바로 데이터를 주고받게 됩니다.

그런데 Acceptor의 run() 메서드에서는 while 문을 사용해 루프를 돌도록 되어 있습니다.
하지만 별다른 루프 지연 로직이 없어 무한 루프가 도는 것처럼 보였는데,
알고 보니 accept를 호출했을 때 Accept Queue에 쌓인 연결 요청이 없는 경우에는
Blocking 상태가 되어 루프가 돌지 않고 해당 호출에서 멈추게 된다는 것을 알게 되었습니다.

![](/assets/img/blog/20241201/img-12.png){:.lead}

endpoint.serverSocketAccept()를 타고 들어가면 accept 시스템 콜을 호출
{:.figcaption}

accept 콜 이후 Acceptor는 생성된 소켓의 등록 이벤트를 톰캣의 또 다른 데몬 스레드인 Poller에 추가한 뒤 한 번의 루프를 종료합니다.

![](/assets/img/blog/20241201/img-13.png){:.lead}

Acceptor 루프 흐름
{:.figcaption}

## Poller

Poller도 Acceptor처럼 톰캣 실행 시점에 생성되어 종료될 때까지 백그라운드에서 동작하는 데몬 스레드입니다.
Poller는 루프를 돌며 Acceptor가 추가한 소켓 등록 이벤트를 읽어 새로 생성된 소켓을 Selector에 등록합니다.
여기서 Selector는 여러 소켓의 상태 변화를 관리하는 역할을 합니다.

![](/assets/img/blog/20241201/img-14.png){:.lead}

Acceptor가 추가한 소켓 등록 이벤트에서 SocketChannel 추출하여 Selector에 등록
{:.figcaption}

또한 루프 내에서 Selector의 select()를 호출하여 상태 변화를 감지하는 시스템 콜을 호출합니다.
운영체제마다 Selector의 구현체가 다르며 리눅스는 epoll, 맥은 kqueue와 같은 시스템 콜을 사용합니다.
구현은 다르지만 결과적으로 모두 소켓들의 상태 변화 이벤트를 감지하고 수집하는 기능을 수행합니다.

![](/assets/img/blog/20241201/img-15.png){:.lead}

Poller는 이렇게 수집된 이벤트를 Executor에게 넘기며 이후 처리를 위임합니다.
만약 소켓에 읽을 데이터가 존재한다는 이벤트가 발생했다면,
소켓으로부터 데이터를 읽는 작업과 파싱하는 작업들은 Executor가 수행하게 됩니다.

![](/assets/img/blog/20241201/img-16.png){:.lead}

## Executor

Executor는 우리가 자주 보던 `http-nio-8080-exec-1`과 같은 이름의 워커 스레드들입니다.
스레드 풀로 관리되며, 기본값으로 10개의 Executor가 작업을 수행하기 위해 대기하고 있습니다.

Executor는 소켓에서 데이터를 읽고, HTTP 요청을 파싱하는 등의 작업을 통해 HttpServletRequest를 생성하고,
이를 서블릿 컨테이너에 넘겨 비즈니스 로직을 수행합니다.
그리고 응답을 다시 소켓에 입력하는 것까지의 역할을 수행한 뒤 스레드 풀에 반환되어 다음 작업을 대기합니다.

![](/assets/img/blog/20241201/img-17.png){:.lead}

소켓 데이터 읽기
{:.figcaption}

![](/assets/img/blog/20241201/img-18.png){:.lead}

헤더 파싱
{:.figcaption}

![](/assets/img/blog/20241201/img-19.png){:.lead}

서블릿 컨테이너에 전달
{:.figcaption}

## 느낀 점

처음 생각했던 것보다 더 로우 레벨로 파고들다 보니 몰랐던 것들을 굉장히 많이 접하게 된 공부였습니다.
현 글에서는 주제에서 벗어나지 않기 위해 많은 것들을 생략했지만,
'커널의 3way-handshake 처리 흐름', '톰캣의 최적화 포인트', '톰캣의 확장성을 위한 아키텍처 구조' 등
이후 많은 도움이 될 지식들이 덩굴째 따라왔습니다.
이것들도 따로 포스팅을 통해 완전히 내 것으로 만드는 작업을 잊지 말아야겠습니다.
