---
title:  "[Spring] 스프링 컨테이너와 스프링 빈"
excerpt: ""

categories:
  - BackEnd
tags:
  - [BackEnd, Spring]

toc: true
toc_sticky: true
 
date: 2022-03-24
last_modified_at: 2022-03-24
---

## 1️⃣ 스프링 컨테이너
IoC 방식으로 빈을 관리한다는 의미에서 애플리케이션 컨텍스트(ApplicationContext)나 빈 팩토리를 컨테이너 또는 IoC 컨테이너라고도 한다.
컨테이너라는 말 자체가 IoC의 개념을 담고 있다. 또 컨테이너라는 말은 애플리케이션 컨텍스트보다 추상적인 표현이기도 하다.
애플리케이션 컨텍스트는 그 자체로 ApplicationContext 인터페이스를 구현한 오브젝트를 가리키기도 하는데, 애플리케이션 컨텍스트 오브젝트는 하나의 애플리케이션에서 보통 여러 개가 만들어져 사용된다.
이를 통틀어서 스프링 컨테이너라고 부를 수 있다.

> 그냥 컨테이너 또는 스프링 컨테이너라고 할 때는 애플리케이션 컨텍스트(ApplicationContext)를 말함

### 1. 빈 팩토리(BeanFactory)
스프링의 IoC를 담당하는 핵심 컨테이너를 가리킨다.
빈을 등록하고, 생성하고, 조회하고, 돌려주고, 그 외에 부가적인 빈을 관리하는 기능을 담당한다.
보통은 이 빈 팩토리를 바로 사용하지 않고 이를 확장한 애플리케이션 컨텍스트를 이용한다.
BeanFactory라고 붙여쓰면 빈 팩토리가 구현하고 있는 가장 기본적인 인터페이스의 이름이 된다.
이 인터페이스에 `getBean()`과 같은 메소드가 정의되어 있다.

### 2. 애플리케이션 컨텍스트(ApplicationContext)
빈 팩토리를 확장한 IoC 컨테이너다.
빈을 등록하고 관리하는 기본적인 기능은 빈 팩토리와 동일하다. 여기에 스프링이 제공하는 각종 부가 서비스를 추가로 제공한다.
빈 팩토리라고 부를 때는 주로 빈의 생성과 제어의 관점에서 이야기하는 것이고, 애플리케이션 컨텍스트라고 할 때는 스프링이 제공하는 애플리케이션 지원 기능을 모두 포함해서 이야기 하는 것이라고 보면 된다.
스프링에서는 애플리케이션 컨텍스트라는 용어를 빈 팩토리보다 더 많이 사용한다.
ApplicationContext라고 적으면 애플리케이션 컨텍스트가 구현해야 하는 기본 인터페이스를 가리키는 것이기도 하다.
ApplicationContext는 BeanFactory를 상속한다.

- ApplicationContext가 제공하는 부가기능
  - 메시지 소스를 활용한 국제화 기능<br>
    예를 들어 한국에서 들어오면 한국어로, 영어권에서 들어오면 영어로 출력
  - 환경변수<br>
    로컬, 개발, 운영등을 구분해서 처리
  - 애플리케이션 이벤트<br>
    이벤트를 발행하고 구독하는 모델을 편리하게 지원
  - 편리한 리소스 조회<br>
    파일, 클래스패스, 외부 등에서 리소스를 편리하게 조회

### 설정 형식 지원
  - 애노테이션 기반의 자바 설정 클래스
  - XML

#### 애노테이션 기반의 자바 설정 클래스
```java
//스프링 컨테이너 생성
ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
```
스프링 컨테이너를 생성할 때는 구성 정보를 지정해주어야 한다.<br>
여기서는 `AppConfig.class`를 구성정보로 지정했다.

#### XML
최근에는 스프링 부트를 많이 사용하면서 XML 기반의 설정은 잘 사용하지 않는다. 
아직 많은 레거시 프로젝트들이 XML로 되어 있고, XML을 사용하면 컴파일 없이 빈 설정 정보를 변경할 수 있는 장점이 있다.
```java
//스프링 컨테이너 생성
ApplicationContext ac = new GenericXmlApplicationContext("appConfig.xml");
```

## 2️⃣ 스프링 빈
빈 또는 빈 오브젝트는 스프링이 IoC 방식으로 관리하는 오브젝트라는 뜻이다.
관리되는 오브젝트(managed object)라고 부르기도 한다.
스프링이 직접 그 생성과 제어를 담당하는 오브젝트만을 빈이라고 부른다.

```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }
}
```
- `@Configuration`<br>
  ApplicationContext 또는 BeanFactory가 사용할 설정정보라는 표시
- `@Bean`
  - 오브젝트 생성을 담당하는 IoC용 메소드라는 표시
  - 빈 이름은 메소드 이름을 사용
  - 빈 이름 직접 부여 가능
    `@Bean(name="memberService2")`
  - 빈 이름은 항상 다른 이름을 부여해야 함<br>
    같은 이름을 부여하면, 다른 빈이 무시되거나 기존 빈을 덮어버리거나 설정에 따라 오류가 발생함
  