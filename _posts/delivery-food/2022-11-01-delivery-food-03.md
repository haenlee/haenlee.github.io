---
title: "[DeliveryFood] Jasypt 암호화"
excerpt: ""

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood]

toc: true
toc_sticky: true
 
date: 2022-11-01
last_modified_at: 2022-11-01
---

프로젝트 CI 세팅을 한 후 로컬 DB에서 작업했던 내용들의 테스트가 모두 Fail이 되었다. Naver Cloud 에 이미 MySql 서버를 만들어 두었었고 이 서버를 main branch에서 동작할 수 있도록 하는게 목표였다.

그냥 main branch에 올려도 되겠지만, 많은 사람들이 보는 GitHub에 DB 정보를 모두 올리는게 상당히 좋지 않은 방법이었다. 그래서 젠킨스 환경 변수에 DB 정보를 추가하고 그 정보를 받아와 application.properties에 세팅할 생각이었다.

그래서 서칭하던 중 Jasypt 라는 라이브러리를 알게 되었다. 목적은 DB 정보를 숨기기만 하면 되기 때문에 간단히 사용할 수 있어서 젠킨스 환경 변수 세팅보다 쉽게 적용할 수 있었다.

<br>
# 🚀 Jasypt
---
Jasypt는 ‘**Java Simplified Encryption**’ 즉, 자바의 간단한 암호화 라이브러리다. 

일단 암호화에 대한 간단하지만 많은 기능을 지원한다.

- 단방향 및 양방향 암호화를 지원한다.
- Hibernate와의 호환성을 지원한다.
- Spring 기반 애플리케이션과 Spring Security와도 호환성을 지원한다.
- 다중 프로세서/다중 코어 시스템에서 고성능 암호화를 위한 특정 기능을 지원한다.

> [Jasypt: Java simplified encryption - Jasypt: Java simplified encryption - Main](http://www.jasypt.org/)

<br>
# 🚀 Jasypt 적용하기
---
- 가장 먼저 라이브러리를 사용할 수 있도록 디펜던시를 걸어준다.
    
    ```sql
    implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.4'
    ```
    
- JasyptConfig 클래스를 생성한다.
    
    ```java
    package com.deliveryfood.config;
    
    import org.jasypt.encryption.StringEncryptor;
    import org.jasypt.encryption.pbe.PooledPBEStringEncryptor;
    import org.jasypt.encryption.pbe.config.SimpleStringPBEConfig;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    @Configuration
    public class JasyptConfig {
    
        @Bean("jasyptStringEncryptor")
        public StringEncryptor stringEncryptor() {
            PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
            SimpleStringPBEConfig config = new SimpleStringPBEConfig();
            config.setPassword("pwkey");
            config.setAlgorithm("PBEWithMD5AndDES");
            config.setKeyObtentionIterations("1000");
            config.setPoolSize("1");
            config.setProviderName("SunJCE");
            config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
            config.setIvGeneratorClassName("org.jasypt.iv.NoIvGenerator");
            config.setStringOutputType("base64");
            encryptor.setConfig(config);
            return encryptor;
        }
    }
    ```

- 암호화 텍스트 생성
    > [https://www.devglan.com/online-tools/jasypt-online-encryption-decryption](https://www.devglan.com/online-tools/jasypt-online-encryption-decryption)
    
    홈페이지를 통해 쉽게 만들 수 있다.
    `(Select Type of Encrpytion으로 Two Way Encryption(With Secret Key)를 선택해줄 것)`
    

- application.properties 변경
    
    ```java
    spring.datasource.drive-class-name=com.mysql.cj.jdbc.Driver
    spring.datasource.url=jdbc:mysql://118.67.128.73:3306/df_db?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    spring.datasource.username=ENC(s5CDvQnd1UySeIkYPxKN2A==)
    spring.datasource.password=ENC(spIXF1Au+HxIdZngAoLFMQqeACylznX+)
    ```
    
    이런식으로 암호화 할 정보를 ENC(생성한 텍스트) 형식으로 지정한다.

## 📝 PBEWithMD5AndDES
config 설정에서 제일 흥미로웠던 점은 `PBEWithMD5AndDES` 암호화 알고리즘을 적용한 거였다. 
    
- PBE
    - Password-Based Encryption (패스워드 기반 알고리즘)
    - 패스워드를 키로 사용하는 방법
    - 키 스페이스가 TripleDES나 Blowfish 보다 훨씬 작아 안전하지 못하다.<br>
    (Blowfish: 1993년 설계된 키 방식의 대칭형 블록 암호, 현재는 고급 암호화 표준을 더 많이 사용)
    - 해시와 일반적인 대칭 암호화를 사용한다.
    - salt과 interation counts를 도입하여 보안성을 강화한다.
        - salt: salt는 해시함수를 돌리기 전에 원문에 임의의 문자열을 덧붙이는 것
        - iteration counts: 해시의 횟수
- MD5
    - 128비트 암호화 해시 함수
    - 주로 프로그램이나 파일이 원본 그대로인지 확인하는 무결성 검사등에 사용된다.
    - (계속해서 치명적 결함이 발견되어.. 보안 관련 용도로 쓰는 것을 권장하지 않는다.)
- DES
    - 대칭키 암호이며, 56비트의 키를 사용한다.
    - (마찬가지로 계속해서 치명적 결함이 발견되어..AES를 새 표준으로 사용한다.)
    
이름에서 알 수 있듯이 PBE, MD5, DES 알고리즘을 사용하는 암호화 방식인 것이다. 오래된 알고리즘과 쉽게 해독되는 알고리즘을 사용하고 있어서 토이 프로젝트가 아니라면 사용하지 않았을 방식이겠다..

<br>
# 🚀 마무리
---
이렇게 지정한 후 commit - push를 하니 CI가 scan하여 빌드와 테스트를 진행하는데, DB 세팅만으로도 테스트 코드의 Fail이 반은 없어졌다. 이제 CI 테스트가 모두 성공할 때까지 수정만 하면 된다.

젠킨스 환경 변수 설정 없이도 쉽게 DB 설정을 잡을 수 있었지만, `Jasypt에서 지원하는 암호화 방식이 불안전하기 때문에 꼭 토이 프로젝트에서만 사용하자.`