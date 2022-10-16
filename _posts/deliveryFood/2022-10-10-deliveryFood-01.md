---
title: "[DeliveryFood] Spring Security 적용(1)"
excerpt: "Spring Security란?"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood]

toc: true
toc_sticky: true
 
date: 2022-10-10
last_modified_at: 2022-10-10
---

프로젝트의 로그인 기능을 구현하면서 Spring Security를 적용하려고 했다. Spring Security를 적용해보려고 한 이유는 Spring Security가 보안과 관련해서 다양한 옵션을 제공해주기 때문이었다. 개발자 입장에서는 보안관련 기능을 편하게 쓸 수 있다는 장점이 있고, 무엇보다 Spring Security 관련해서 여기저기서(?) 많이 들어보았기 때문에 한번 공부해보고 싶은 마음이 컸던 것 같다.

Spring Security를 제대로 적용하기까지는 여러가지 세팅이 필요했고, 그 세팅과정이 어렵지만 한번 적용해놓으면 가져다 쓰기만 하면 돼서 초반 세팅에 시간을 쏟았었다.

## Spring Security
Spring Security는 `엔터프라이즈 애플리케이션을 위한 인증, 권한 부여 및 기타 보안 기능을 제공하는 Java/Java EE 프레임워크`라고 한다. 설명에도 나와있듯이 Spring Security에서 가장 중요한 키워드는 인증, 권한이다. 인증, 권한을 위해서 필요한 것이 계정이고, 그 계정에 어떤 권한이 설정되는지가 중요하다.

Spring Security는 주로 서블릿 필터와 이들로 구성된 필터체인을 통해 웹 요청에 대한 보안 관련 처리를 수행한다.
서블릿 필터란 HTTP 요청을 가로채 전처리 및 후처리를 수행할 수 있도록 만들어진 자바 표준 기술이다. 서블릿 필터는 논리적으로 서블릿 컨테이너와 서블릿 사이에 위치한다. 
스프링 웹 MVC 애플리케이션이라면 이 서블릿이 바로 DispatcherServlet의 인스턴스이다.

필터는 체인으로 구성될 수 있으며, 하나의 필터가 자신의 역할을 한 후, 요청과 응답 객체를 다음 필터로 넘길 수 있다.
즉, Spring Security가 하는 일은 클라이언트의 요청이 서블릿에 도착하기 전에 여러 필터를 거치게 해서 인증, 인가 및 여러 보안처리를 하는 것이다.

### 기본 용어
- 접근 주체(Principal)
  - 보호된 리소스에 접근하는 대상
- 인증(Authentication)
  - 보호된 리소스에 접근한 대상에 대해 누구인지, 애플리케이션의 작업을 수행해도 되는 주체인지 확인하는 과정
  - ex. Form 기반 로그인 
- 인가(Authorize)
  - 해당 리소스에 대해 접근 가능한 권한을 가지고 있는지 확인하는 과정
  - After Authnetication, 인증 이후
- 권한
  - 어떤한 리소스에 대한 접근 제한
  - 모든 리소스는 접근 제어 권한이 걸려있음
  - 인가 과정에서 해당 리소스에 대한 제한된 최소한의 권한을 가졌는지 확인

## Spring Security 의 구조
### Spring Security 동작 구조
![image](https://user-images.githubusercontent.com/85219306/194808856-c669ffa2-e5f1-4324-97c7-5b700a4e5aec.png)

**<u>1. 로그인 정보를 담아 서버에 인증을 요청한다. (http request)</u>**
  - 브라우저의 로그인 화면에서 ID/PW를 입력하고 확인을 누르면, 서버에 로그인 인증 요청을 한다.
  - Spring Security에 `Chain 형태로 구성된 Filter들의 doFilter메소드가 순서에 따라 호출`되어 각각의 역할 로직들이 수행된다.

**<u>2. 인증 처리를 담당하는 UsernamePasswordAuthenticationFilter가 실행된다.</u>**
  - AbstractAuthenticationProcessingFilter 추상 클래스의 추상메소드 attemptAuthentication를 호출하여 요청을 처리하게 된다. 
  - attemptAuthentication추상메소드의 구현은 상속한 UsernamePasswordAuthenticationFilter에 구현 되어 있다.
  - 추후 외부인증을 위한 작업을 할때 `UsernamePasswordAuthenticationFilter기능을 대신하는 커스텀 필터`를 추가하게 된다.
  - 인증 성공 실패에 따라 AuthenticationSuccessHandler, AuthenticationFailureHandler를 최종적으로 호출하게 된다.
  - 인증이 성공했다면 리턴값 UsernamePasswordAuthenticationToken를 세션에 저장한다.

**<u>3. AuthenticationManager가 적절한 AuthenticationProvider를 찾는다.</u>**
  - AuthenticationManager는 인터페이스이며 구현체는 ProviderManager이다. ProviderManager는 실제 인증을 처리하는 로직이 포함된 AuthenticationProvider 인터페이스의 구현체들 중에 설정된 인증 처리방식의 구현체를 찾아 실행한다.

**<u>4. 실제 인증처리하는 AuthenticationProvider의 인증처리 메소드를 호출한다.</u>**
  - 일반적으로 클라이언트에서 ID/PW를 받아 인증하는 방식을 사용한다면 `AuthenticationProvider` 인터페이스의 구현체 AbstractUserDetailsAuthenticationProvider 추상클래스를 호출하게 되고 실제 상속한 클래스의 DaoAuthenticationProvider에서 인증 처리를 하게 된다.

**<u>5. 인증제공자는 UserDetailsService를 호출하여 사용자를 가져온다.</u>**
  - 개발자는 UserDetailsService 인터페이스를 구현해야한다. UserDetailsService의 구현체에는 일반적으로 회원정보가 DB에 있다고 한다면, 사용자의 이름(ID)로 DB를 조회하여 비밀번호가 일치하는지 확인하여 인증을 처리한다. 인증이 완료되면 UsernamePasswordAuthenticationToken에 회원정보를 담아 리턴하게 된다.

### FilterChainProxy와 SecurityFilterChain
![image](https://user-images.githubusercontent.com/85219306/194807784-1ca83a91-8bef-4be1-8f58-147e4c0fca4a.png)

FilterChainProxy는 Spring Security가 제공하는 특별한 Filter 구현체이다.
DelegatingFilterProxy는 단순히 웹 요청을 필터 빈에게 위임하기 때문에, 실제로 웹 요청을 처리함 TargetFilterBean을 설정해야 한다.
이때, Spring Security의 FilterChainProxy가 해당 빈으로 설정된다.

DelegatingFilterProxy는 FilterChainProxy로 요청 처리를 위임한다. FilterChainProxy는 위의 그림처럼 Security Filter 목록을 가진 SecurityFilterChain을 호출한다. 이 필터 체인을 구성하는 Security Filter들이 바로 각종 인가, 인증 등 다양한 기능을 제공하는 Spring Security의 핵심이다.

`그렇다면 왜 Security의 Filter들은 서블릿 컨테이너에 직접 등록되지 않고, FilterChainProxy를 거쳐서 등록되는 것일까?`<br>
FilterChainProxy를 사용했을때의 장점은 다음과 같다.

- Spring Security 서블릿 지원의 시작점을 제공한다. 따라서 스프링 시큐리티와 관련해서 트러블슈팅이 필요하다면 FilterChainProxy를 디버깅의 시작점으로 잡을 수 있다.
- FilterChainProxy는 메모리 누수 방지를 위해 SecurityContext를 비우는 등 Spring Security 사용에 있어 중심부 역할을 수행한다. 또 Spring Security의 HttpFirewall을 적용하여 특정 타입의 공격으로부터 애플리케이션을 보호한다.
- FilterChainProxy는 SecurityFilterChain이 언제 호출될지를 결정함으로써 유연성을 제공한다.

Spring Security에는 수많은 Filter들이 구현되어있다.
Spring Security들을 잘 사용하기 위해서는 이 Filter들을 활용할 수 있어야 한다.