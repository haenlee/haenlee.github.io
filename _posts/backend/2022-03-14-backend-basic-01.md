---
title:  "[Spring] 스프링 입문"
excerpt: "Spring 입문"

categories:
  - BackEnd
tags:
  - [BackEnd, Spring]

toc: true
toc_sticky: true
 
date: 2022-03-14
last_modified_at: 2022-03-14
---

## 1️⃣ 프로젝트 생성

### (1) 스프링 부트 스타터 사이트에서 프로젝트 생성
<a href="https://start.spring.io/">https://start.spring.io/</a>

- 프로젝트 선택
  - Project: Gradle Project
  - Spring Boot: 2.6.4
  - Language: Java
  - Packaging: Jar
  - Java: 11
- Project Metadata
  - groupId: hello
  - artifactId: hello-spring
- Dependencies: Spring Web, Thymeleaf
<br>

### (2) Gradle 설정
build.gradle

```markdown
plugins {
	id 'org.springframework.boot' version '2.6.4'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'hello'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-web''
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}
```
- `mavenCentral()` : dependencies 를 다운로드 받는 공개 사이트
<br>

### (3) 라이브러리
Gadle은 의존관계가 있는 라이브러리를 함께 다운로드 한다.

#### 스프링 부트 라이브러리
- spring-boot-starter-web
  - spring-boot-starter-tomcat: 톰캣 (웹서버)
  - spring-webmvc: 스프링 웹 MVC
  - spring-boot-starter-thymeleaf: 타임리프 템플릿 엔진(View)
  - spring-boot-starter(공통): 스프링 부트 + 스프링 코어 + 로깅
    - spring-boot
      - spring-core
    - <u>spring-boot-starter-logging</u> (로그 관련)
      - logback, slf4j

#### 테스트 라이브러리
- spring-boot-starter-test
  - <u>junit</u>: 테스트 프레임워크 (요즘에는 junit5를 쓰는 추세)
  - mockito: 목 라이브러리
  - assertj: 테스트 코드를 좀 더 편하게 작성하게 도와주는 라이브러리
  - spring-test: 스프링 통합 테스트 지원
<br>

### (4) View 환경설정
- 스프링 부트가 제공하는 Welcome Page 기능<br>
  `static/index.html`을 올려두면 Welcome page 기능을 제공

#### thymeleaf 템플릿엔진 동작
- 컨트롤러에서 리턴 값으로 문자를 반환하면 뷰 리졸버(viewResolver)가 화면을 찾아서 처리<br>
  - 스프링 부트 템플릿엔진 기본 viewName 매핑
  - resources:templates/ + {ViewName} + .html
<br>

### (5) 빌드하고 실행하기
  1. 콘솔(cmd)에서 프로젝트 경로로 이동
  2. 콘솔에서 gradlew build 입력<br>
     ( gradlew clean build : build 폴더 지우고 다시 빌드 )
  3. cd build/libs
  4. java -jar hello-spring-0.0.1-SNAPSHOT.jar
  5. 실행 확인
<br>

## 2️⃣ 스프링 웹 개발 기초
- 정적 컨텐츠 (파일을 웹 브라우저에 그대로 내려줌)
- MVC와 템플릿 엔진 (JSP, PHP)
- API (JSON 포맷으로 클라이언트에게 전달하는 방식)

### (1) 정적 컨텐츠
- `resources/static/hello-static.html`<br>
  resources/static 경로에 html 파일을 생성
- 스프링 컨테이너에서 먼저 hello-static 관련 컨트롤러를 찾고, 없으면 `resources/static/hello-static.html`을 반환함
<br>

### (2) MVC와 템플릿 엔진
- MVC : Model, View, Controller
<br>

### (3) API
- `@ResponseBody` 문자 반환<br>
  - `@ResponseBody`를 사용하면 뷰 리졸버(viewResolver)를 사용하지 않음<br>
    (뷰 리졸버(viewReslover)대신에 HttpMessageConverter가 동작)
  - 대신에 HTTP의 BODY에 문자 내용을 직접 반환<br>
    (HTML BODY TAG를 말하는 것이 아님)
  - 기본 문자 처리 : StringHttpMessageConverter
  - 기본 객체 처리 : MappingJackson2HttpMessageConverter<br>
    `@ResponseBody`를 사용하고, 객체를 반환하면 객체가 JSON으로 변환됨
  - byte 처리 등등 기타 여러 HttpMessageConverter가 기본으로 등록되어 있음

> 클라이언트의 HTTP Accept 헤더와 서버의 컨트롤러 반환 타입 정보 둘을 조합해서 HttpMessageConverter가 선택된다.

### 회원 관리 예제 - 백엔드 개발
#### 비즈니스 요구사항 정리
- 데이터 : 회원ID, 이름
- 기능 : 회원 등록, 조회
- 아직 데이터 저장소가 선정되지 않음(가상의 시나리오)

##### 일반적인 웹 애플리케이션 계층 구조
![image](https://user-images.githubusercontent.com/85219306/158118841-57b3afbe-705c-4656-bc1f-fc98c7b53299.png)
- 컨트롤러 : 웹 MVC의 컨트롤러 역할
- 서비스 : 핵심 비즈니스 로직 구현
- 리포지토리 : 데이터베이스에 접근, 도메인 객체를 DB에 저장하고 관리
- 도메인 : 비즈니스 도메인 객체, 예) 회원, 주문, 쿠폰 등등 주로 데이터베이스에 저장하고 관리됨

##### 클래스 의존관계
![image](https://user-images.githubusercontent.com/85219306/158135736-f5fea6ff-f43c-49fa-8e39-dd0af551cda3.png)
- 아직 데이터 저장소가 선정되지 않아서, 우선 인터페이스로 구현 클래스를 변경할 수 있도록 설계
- 데이터 저장소는 RDB, NoSQL 등등 다양한 저장소를 고민중인 상황으로 가정
- 개발을 진행하기 위해서 초기 개발 단계에서는 구현체로 가벼운 메모리 기반의 데이터 저장소 사용
<br>

#### 테스트 케이스 작성
자바는 `JUnit`이라는 프레임워크로 테스트를 실행할 수 있음
> 테스트는 각각 독립적으로 실행되어야 한다.<br>
테스트 순서에 의존관계가 있는 것은 좋은 테스트가 아니다.

##### @AfterEach
한번에 여러 테스트를 실행하면 메모리 DB에 직전 테스트의 결과가 남을 수 있다.<br>
이렇게 되면 이전 테스트 때문에 다음 테스트가 실패할 가능성이 있다.<br>
@AfterEach를 사용하면 각 테스트가 종료될 때마다 이 기능을 실행한다.

##### @BeforeEach
각 테스트 실행 전에 호출된다.<br>
테스트가 서로 영향이 없도록 항상 새로운 객체를 생성하고, 의존관계도 새로 맺어준다.
<br>

## 3️⃣ 스프링 빈과 의존관계
### (1) 컴포넌트 스캔과 자동 의존관계 설정
- @Autowired
생성자에 `@Autowired`가 있으면 스프링이 연관된 객체를 스프링 컨테이너에서 찾아서 넣어준다.<br>
이렇게 객체 의존관계를 외부에서 넣어주는 것을 `DI(Dependency Injection), 의존성 주입`이라 한다.

> 생성자에 @Autowired를 사용하면 객체 생성 시점에 스프링 컨테이너에서 해당 스프링 빈을 찾아서 주입한다.<br>
생성자가 1개만 있으면 @Autowired는 생략할 수 있다.

- 컴포넌트 스캔 원리
  - `@Component`가 있으면 스프링 빈으로 자동 등록된다.
  - `@Controller`가 스프링 빈으로 자동 등록된 이유도 컴포넌트 스캔 때문이다.
  - `@Component`를 포함하는 다음 애노테이션도 스프링 빈으로 자동 등록된다.
    - @Controller
    - @Service
    - @Repository

> 스프링은 스프링 컨테이너에 스프링 빈을 등록할 때, 기본적으로 싱글톤으로 등록한다.<br>
따라서 같은 스프링 빈이면 모두 같은 인스턴스다.<br>
설정으로 싱글톤이 아니게 설정할 수 있지만, 특별한 경우를 제외하면 대부분 싱글톤을 사용한다.

<br>

### (2) 자바 코드로 직접 스프링 빈 등록하기
클래스에 `@Configuration`을 추가하고 생성자를 `@Bean`으로 등록한다.
- DI에는 필드 주입, setter 주입, 생성자 주입 이렇게 3가지 방법이 있다.<br>
  의존관계가 실행중에 동적으로 변하는 경우는 거의 없으므로 생성자 주입을 권장한다.
-실무에서는 주로 정형화된 컨트롤러, 서비스, 리포지토리 같은 코드는 컴포넌트 스캔을 사용한다.<br>
  그리고 정형화 되지 않거나, 상황에 따라 구현 클래스를 변경해야 하면 설정을 통해 스프링 빈으로 등록한다.
 
## 4️⃣ 스프링 DB 접근 기술
- 순수 jdbc
- 스프링 JdbcTemplate
- JPA
- 스프링 데이터 JPA

### 스프링 통합 테스트
- @SpringBootTest : 스프링 컨테이너와 테스트를 함께 실행한다.
- @Transactional : 테스트 케이스에 이 애노테이션이 있으면, 테스트 시작 전에 트랜잭션을 시작하고, 테스트 완료 후에 항상 롤백한다.<br>
  이렇게 하면 DB에 데이터가 남지 않으므로 다음 테스트에 영향을 주지 않는다.
<br>

### (1) 스프링 JdbcTemplate
스프링 JdbcTemplate와 MyBatis 같은 라이브러리는 JDBC API에서 반복 코드를 대부분 제거해준다.<br>
하지만 SQL은 직접 작성해야 한다.
<br>

### (2) JPA
- JPA는 기존의 반복 코드는 물론이고, 기본적인 SQL도 JPA가 직접 만들어서 실행해준다.
- JPA를 사용하면, SQL과 데이터 중심의 설계에서 객체 중심의 설계로 패러다임을 전환할 수 있다.
- JPA를 사용하면 개발 생산성을 크게 높일 수 있다.
<br>

### (3) 스프링 데이터 JPA
스프링 부트와 JPA만 사용해도 개발 생산성이 정말 많이 증가하고, 개발해야할 코드도 확연히 줄어든다.<br>
여기에 스프링 데이터 JPA를 사용하면, 리포지토리에 구현 클래스 없이 인터페이스 만으로 개발을 완료할 수 있다.<br>
그리고 반복 개발해온 기본 CRUD 기능도 스프링 데이터 JPA가 모두 제공한다.

- 스프링 데이터 JPA 제공 기능
  - 인터페이스를 통한 기본적인 CRUD
  - findByName(), findByEmail() 처럼 메서드 이름 만으로 조회 기능 제공
  - 페이징 기능 자동 제공

> 실무에서는 JPA와 스프링 데이터 JPA를 기본으로 사용하고, 복잡한 동적 쿼리는 Querydsl이라는 라이브러리를 사용하면 된다.<br>
  Querydsl을 사용하면 쿼리도 자바 코드로 안전하게 작성할 수 있고, 동적 쿼리도 편리하게 작성할 수 있다.<br>
  이 조합으로 해결하기 어려운 쿼리는 JPA가 제공하는 네이티브 쿼리를 사용하거나, 스프링 JdbcTemplate를 사용하면 된다.

<br>

## 5️⃣ AOP
Aspect Oriented Programming

### AOP가 필요한 상황
- 모든 메서드의 호출 시간을 측정하고 싶다면?
- 공통 관심 사항(cross-cutting concern) vs 핵심 관심 사항(core concern)
- 회원 가입 시간, 회원 조회 시간을 측정하고 싶다면?

#### 문제
- 회원 가입, 회원 조회에 시간을 측정하는 기능은 핵심 관심 사항이 아니다.
- 시간을 측정하는 로직은 공통 관심 사항이다.
- 시간을 측정하는 로직과 핵심 비즈니스의 로직이 섞여서 유지보수가 어렵다.
- 시간을 측정하는 로직을 별도의 공통 로직으로 만들기 매우 어렵다.
- 시간을 측정하는 로직을 변경할 때 모든 로직을 찾아가면서 변경해야 한다.

> 공통 관심 사항(cross-cutting concern) vs 핵심 관심 사항(core concern) 분리

#### 해결
- 회원 가입, 회원 조회등 핵심 관심사항과 시간을 측정하는 공통 관심 사항을 분리한다.
- 시간을 측정하는 로직을 별도의 공통 로직으로 만들었다.
- 핵심 관심 사항을 깔끔하게 유지할 수 있따.
- 변경이 필요하면 이 로직만 변경하면 된다.
- 원하는 적용 대상을 선택할 수 있다. (@Around)