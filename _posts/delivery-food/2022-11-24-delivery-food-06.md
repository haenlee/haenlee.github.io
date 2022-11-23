---
title: "[DeliveryFood] 라이더들에게 배달 요청을 보내는 방법"
excerpt: "서버의 event를 클라이언트로 보내기"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood, Event, Polling, Long-Polling, WebSocket, SSE]

toc: true
toc_sticky: true
 
date: 2022-11-24
last_modified_at: 2022-11-24
---

일반적인 서버 - 클라이언트의 HTTP 통신 과정은 클라이언트 → 서버로 request를 전송하고 서버 → 클라이언트로 response를 전송한다.
문제는 클라이언트가 요청하지 않았어도 서버가 클라이언트로 reponse를 보낼 수 있는가? 부터 시작한다.

📢 `배달 주문이 들어오면 라이더들에게 배달 요청을 보내고 라이더 매칭 과정을 수행한다.`

사용자가 주문을 하면 음식점 주문 수락을 한 후 라이더를 요청하는데, 라이더를 요청하는 과정에서 범위 내에 있는 라이더들에게 배달 수락 여부에 대해 event를 보내게 된다.
배달 매칭 알고리즘 문서를 작성하며 일반적인 HTTP 통신을 사용해 event를 보낼 수 있는지, 아니라면 다른 방법이 있는지에 대해 정리해보았다.

🔗 관련 포스트 참고  
[브라우저 동작 방식](https://haenlee.github.io/network/network-03/)
{: .notice--success}

<br>
# 🚀 HTTP request
---
HTTP 통신 과정을 간단하게 아래와 같은 순서로 진행된다.

1. 클라이언트는 서버에게 request를 전송하기 위해 웹서버와 TCP 연결을 시도한다. (3 way Handshake)
2. 웹서버와 HTTP 메시지를 주고 받는다. (request, response)
3. 웹서버와 TCP 연결을 해제한다.

<br>
# 🚀 서버의 event를 클라이언트로 보내는방법
---
## 📝 HTTP Polling
- 클라이언트가 HTTP request를 1~2초 간격으로 보내고 서버는 그에 대한 response를 보내주는 방법

### 🔸 장점
- event를 구현하는 가장 쉬운 방법이다.
- 일정하게 갱신되는 서버 데이터의 경우 유용하다. (ex. 대시보드 갱신)

### 🔸 단점
- 대부분의 request에 비어있는 response를 받을 수 있기 때문에 `불필요한 네트워크 호출`이 많다. (HTTP 오버헤드 발생)
- 1~2초 간격으로 Polling되므로 response는 최대 2초까지 지연되고 `빠른 응답을 기대할 수 없다.`
- HTTP request connection이 계속해서 일어나기 때문에 서버의 부담이 급증한다.
- 모바일 기기의 경우 계속되는 Polling으로 배터리가 소모된다.

➡ `라이더 위치 갱신의 경우에 2초 간격으로 업데이트 되는 것은 오히려 Polling 방식이 좋다.`

## 📝 HTTP Long Polling
- Polling 방식과 비슷하지만 클라이언트에서 request를 보내고 서버에서 클라이언트에게 보낼 response가 있을 때 response를 보내는 방식
- 클라이언트는 서버의 response를 계속 기다린다.
- 클라이언트는 서버의 response를 받거나 타임아웃이 발생하면 연결을 종료하고 다시 request를 보내는 과정을 반복한다.

### 🔸 장점
- 실시간 처리가 가능하다.
- 비어있는 response를 받는 경우가 없다.

### 🔸 단점
- response의 간격이 좁다면 Polling과 크게 차이가 없어진다.
- 다수의 클라이언트에게 response를 보내게 되면 다시 다수의 클라이언트가 connection을 요청하기 때문에 서버의 부담이 급증한다.

## 📝 WebSocket
### 🔸 WebSocket이란?
두 프로그램 간의 메시지 교환을 위한 통신 방법 중 하나이다.

![image](https://user-images.githubusercontent.com/85219306/203605364-43b2c81f-bbf0-41b1-b74e-9ac5a818bd1a.png)

- 현재 인터넷 환경(HTML5)에서 많이 사용된다
- WebSocket을 지원하는 브라우저의 경우 WebSocket 프로토콜을 지원
- W3C와 IETF에 의해 자리잡은 표준 프로토콜 중 하나

#### 특징
- 양방향 통신(Full-Duplex)
    - 데이터 송수신을 동시에 처리할 수 있는 통신 방법
    - 클라이언트와 서버가 서로에게 원할 때 데이터를 주고 받을 수 있다.
    - 통상적인 Http 통신은 Client가 요청을 보내는 경우에만 Server가 응답을 하는 단방향 통신
- 실시간 네트워킹(Real Time-Networking)
    - 웹 환경에서 연속된 데이터를 빠르게 노출
    - ex) 채팅, 주식, 비디오 데이터

### 🔸 WebSocket
- 클라이언트는 서버와 WebSocket Handshake를 통해 connection을 맺는다.
- 양방향 통신이 가능하다.
- HTTP request는 요청한 클라이언트에게만 response를 보내는 방식이었는데, websocket 프로토콜을 통해 포트에 접속해 있는 모든 클라이언트에게 이벤트 방식으로 응답한다.

### 🔸 장점
- 양방향 통신이므로 서버는 언제든지 클라이언트에게 response를 보낼 수 있고 클라이언트는 언제든지 서버에게 request를 보낼 수 있다.
- Handshake를 한번만 수행하므로 오버헤드가 줄어든다.

### 🔸 단점
- websocket 프로토콜을 처리하기 위해 전이중 연결과 새로운 websocket 서버가 필요하다.

## 📝 Server-Send Events(SSE)
- 서버가 필요할 때마다 클라이언트에게 데이터를 줄 수 있게 해주는 서버 푸시 기술
- SSE를 사용하여 클라이언트는 서버와 지속적으로 연결이 가능하다.
- 서버는 이 연결을 사용하여 데이터를 클라이언트에게 보낸다.

### 🔸 장점
- 전통적인 HTTP를 통해 통신하므로 다른 프로토콜이 필요없다.
- 재접속 처리 같은 대부분의 저수준 처리가 자동으로 된다.
- 표준 기술답게 IE를 제외한 브라우저 대부분을 지원한다.
- HTML과 JavaScript만으로 구현할 수 있으므로 현재 지원되지 않는 브라우저(IE 포함)도 polyfill을 이용해 크로스 브라우징이 가능하다.  
  (여기서 polyfill이란 브라우저가 지원하지 않는 API를 플러그인이나 JavaScript 등으로 흉내 내 구현한 것을 뜻한다.)

### 🔸 단점
- 클라이언트는 SSE를 사용하여 서버에 데이터를 보낼 수 없다. (서버 → 클라이언트 단방향)  
  → 일반 HTTP request를 사용해야 한다.

> [HTTP Request vs HTTP Long-Polling vs WebSocket vs Server-Sent Events](https://amitshekhar.me/blog/http-request-long-polling-websocket-sse)  
> [Polling / Long Polling / Server Sent Event / WebSocket 요약 정리](https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-Polling-Long-Polling-Server-Sent-Event-WebSocket-%EC%9A%94%EC%95%BD-%EC%A0%95%EB%A6%AC)  
> [SSE를 이용한 실시간 웹앱](https://spoqa.github.io/2014/01/20/sse.html)