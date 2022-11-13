---
title: "[DeliveryFood] 테스트와 빌드 자동화를 위한 Jenkins CI 적용하기"
excerpt: "CI적용으로 push, PR 때마다 테스트와 빌드를 자동화해보기"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood, Jenkins, 자동화, CI]

toc: true
toc_sticky: true
 
date: 2022-11-07
last_modified_at: 2022-11-07
---

CI는 애자일 방법론에 최적화 되어 있다고 한다. 테스트, 빌드를 빠르게 하여 결과에 대한 피드백을 받을 수 있게 한다는게 장점이다.

이제까지 업무를 하며 게임 빌드 과정을 서브 옵션처럼 담당해왔기 때문에 백엔드에서 Jenkins를 통한 CI를 적용하는건 생각보다 간단했던것 같다. 게임 개발을 하며 CI를 사용한적은 없었는데 CI를 사용하지 않았던 이유에 대해 생각해보면, 일단 협업에서의 리소스가 필수적인게 큰 것 같다. 기획에서 올려주는 데이터 테이블이라던지 UI 리소스. 캐릭터 관련 애니메이션, 모델, 이펙트들이 모두 올라오고나서야 정상적인 테스트들이 가능하기 때문이다.

보통 자동화 빌드를 위한 젠킨스 파이프라인을 구성해왔었다. 전 회사에서는 직접 플러그인을 설치하고 파이프라인을 구성하는 방식이었는데, 현 회사에서는 빌드용 파이썬 프로젝트를 통해 자동화 빌드를 하고 있었다. 두 과정 모두 겪어보며 많이 배울 수 있었다. 가장 큰 차이점은 파이썬 프로젝트를 통해 빌드를 구성해 놓으니 디버깅이 가능하다는 것이었다. 파이프라인을 통해 만들어놓은 것보다 빌드가 어디서 뻗는지(?) 자세한 분석이 가능했다.

<br>
# 🚀 CI와 Jenkins
---
- CI(Continuous Integration): 지속적 통함
- CD(Continuous Deployment): 지속적 배포

 기능을 개발할 때는 코드를 여러번 수정하게 된다. 이 과정에서 git과 같은 형상 관리 시스템에서 변경 사항을 가져오고, 소스코드를 빌드하거나 단위 테스트를 진행하는 것과 같이 여러 과정을 거친다. 
 
 CI환경에서는 젠킨스와 같은 도구를 이용해 빠르게 진행할 수 있다.

## 📝 젠킨스 자동화
 오늘날 젠킨스는 온갖 종류의 개발 작업을 지원하기 위한 약 1,400가지의 플러그인을 가지고 있는 선두적인 오픈소스 자동화 서버다. 가와구치가 애초에 해결하려 했던 지속적인 통합과 지속적인 자바 코드 전달, 즉 프로젝트 빌드, 테스트 실행, 정적 코드 분석 시행, 그리고 배포 작업은 사람들이 젠킨스를 사용해 자동화하고 있는 여러 가지 프로세스들 가운데 한가지일 뿐이다. 

 이 1,400개의 플러그인은 5가지 영역을 포괄하고 있다(플랫폼, UI, 관리, 소스코드 관리, 그리고 가장 많이 사용되는 빌드 관리).

<br>
# 🚀 CI 적용해보기
---
## 📝 Jenkins 서버 생성 및 설정

 F-Lab에서 네이버 클라우드 크레딧을 지원해 주기 때문에 네이버 클라우드에서 젠킨스 서버를 생성했다. 젠킨스 서버 생성은 아래 가이드를 참고했다. 

> [Jenkins 시작 가이드 - Jenkins](https://guide.ncloud-docs.com/docs/devtools-devtools-1-2)

![Untitled](https://user-images.githubusercontent.com/85219306/200317196-b7895245-7f55-467c-99ef-43623b3066c6.png)

 네이버 클라우드를 통해 젠킨스 서버 생성시 최신 버전의 젠킨스가 아닌 구 버전의 젠킨스가 설치돼어 플러그인이 정상적으로 설치되지 않는다는 글을 봤었는데, 지금은 네이버 클라우드에서 디폴트 젠킨스 버전이 변경되었는지 플러그인 모두 정상적으로 설치되었다. 
 
 JDK 또한 1.8이 기본으로 설치 된다고 했는데 나같은 경우 자동으로 JDK 11이 설치되어 따로 설정을 해주지는 않았다.

- JDK 버전 확인
```sql
java -version
```
![Untitled](https://user-images.githubusercontent.com/85219306/200317461-5d1e4461-cf88-446c-8dc0-7475fe0b636e.png)

- 환경변수 확인
```sql
vi /etc/profile
```
![Untitled](https://user-images.githubusercontent.com/85219306/200317547-75b68d56-5b8a-44be-b3b3-d536d6a0653b.png)

여기까지 하면 기본 젠킨스 서버 세팅이 완료되고, 빌드를 위한 gradle 버전을 설정해주어야 한다. 프로젝트와 동일한 gradle 버전을 젠킨스 서버에 설치해줘야 한다.
프로젝트 내의 gradle 버전은 [프로젝트 이름]/gradle/wrapper/gradle-wrapper.properties에서 알 수 있다.

![Untitled](https://user-images.githubusercontent.com/85219306/200317652-3bdf534f-a9bd-4abc-b4c2-9fa30d7fbbdd.png)

토이 프로젝트의 경우 gradle 7.5 버전을 사용한다. 젠킨스 서버로 가서 gradle 7.5버전을 설치해준다.

- gradle 설치하기
```sql
wget https://services.gradle.org/distributions/gradle-7.5-bin.zip -P /tmp
```

- 설치 후 /opt/gradle 이동
```sql
cd /opt/gradle
```

- gradle 압축 풀기
```sql
sudo unzip -d /opt/gradle /tmp/gradle-*.zip
```

- gradle 설치 확인
```sql
ls /opt/gradle/gradle-7.5
```

- gradle 환경변수 세팅
```sql
export PATH=$PATH:/opt/gradle/gradle-7.5/bin
```

## 📝 Git Hub 토큰 생성
![Untitled](https://user-images.githubusercontent.com/85219306/200317810-ec58d625-66ae-4dda-aff7-688d8af052ae.png)

‘Generate new token’을 클릭해 토큰을 생성해준다.
`이때 생성된 토큰은 다시 확인할 수 없으니 꼭 복사해놓아야 한다.`<br>
생성한 토큰을 바탕으로 GitHub Credentials를 만들어줘야 한다. (Jenkins 관리 > 시스템 설정 > GitHub)

![Untitled](https://user-images.githubusercontent.com/85219306/200318599-d6acfbc5-a51f-4938-9537-da87ebd2f5c2.png)

Kind로 Secret text를 선택하고 GitHub Access token을 입력해준다. ID와 Descrpition은 프로젝트와 관련된 이름으로 지어준다.

![Untitled](https://user-images.githubusercontent.com/85219306/200318765-9dd7d9dc-cb78-4ea8-9d0e-9defc31fa843.png)

## 📝 webhook 생성
프로젝트는 gitflow 브랜치 전략에 맞게 모든 브랜치에 대해 push와 PR이 일어날때마다 test와 build 성공 여부를 알 수 있어야 한다.

이를 위해 먼저 GitHub에서 webhook을 생성해야 한다.
GitHub의 [프로젝트]의 Settings > Webhooks > Add webhook로 이동한 후 아래와 같이 설정해준다. 

payload URL같은 경우 자신의 Jenkins 주소를 입력해주고 webhook가 일어날 트리거를 ‘Just th push event’로 지정한다.

![Untitled](https://user-images.githubusercontent.com/85219306/200319237-7408beb7-b4e3-4f32-8323-befdcddcb947.png)

## 📝 Multibranch Pipeline 생성
모든 브랜치에 대에서 push와 PR이 일어날 때마다 test와 build 가 적용되도록 Multibranch Pipeline을 생성한다. (+ 새로운 Item)

![Untitled](https://user-images.githubusercontent.com/85219306/200319876-ed4801c2-e937-41b2-b5ea-5d1b2c4db683.png)

Item 생성 후 Configure에서 Branch Sources 탭으로 이동한다. Multibranch Pripeline에서는 Secret test로 Credentials 생성이 불가능하기 때문에 Kind를 ‘Username with password’로 선택하고 본인의 Github 닉네임과 발급받은 GitHub Access token으로 새로운 Cerentials를 생성해준다.

![Untitled](https://user-images.githubusercontent.com/85219306/200319978-ab9ff12e-7cf9-4925-883a-cb0f619baf81.png)

![Untitled](https://user-images.githubusercontent.com/85219306/200320057-04da846b-a6be-4618-8247-06781189de11.png)

**Repository HTTPS URL**은 자신의 프로젝트 주소를 넣어준 후 아래 Validate버튼을 눌러 정상적으로 연결이 되는지 확인해보자. 

정상적으로 연결되었다면 아래와 같은 메시지가 출력될 것이다.

![Untitled](https://user-images.githubusercontent.com/85219306/200320237-b1092201-da87-432b-b102-fb4823742e3c.png)

그리고 마지막으로 Repository에 대한 자동 스캔 구간을 지정해준다.

![Untitled](https://user-images.githubusercontent.com/85219306/200320317-3917ea41-feef-4ab1-b49a-08d69cdc4f9d.png)

## 📝 IntelliJ 프로젝트 연동
GitHub를 통해 Push 또는 PR 이벤트가 발생하면 Jenkinsfile에 작성된 Shell script를 기준으로 Jenkins에서 다음 이벤트가 실행된다.

`Jenkinsfile`은 프로젝트의 root 디렉토리 아래 추가해야 한다.

![Untitled](https://user-images.githubusercontent.com/85219306/200320392-11bcab6e-2567-4208-9236-6da186591bf4.png)

아래는 Jenkinsfile 작성 내용이며, test와 build 결과를 slack로 전송하는 내용도 추가되어있다. slack과 연동하여 GitHub webhook 이외에도 알람을 받을 수 있도록 하였다.

```bash
pipeline {
      agent any

      environment {  
              PATH = "/opt/gradle/gradle-7.5/bin:$PATH"
	      SLACK_CHANNEL = '#jenkins'
	      TEAM_DOMAIN = 'delivery-foodhq'
	      TOKEN_CREDENTIAL_ID = 'haenlee-slack'

      }

      stages {
          stage('Git Checkout') {
              steps {
                  checkout scm
                  echo 'git checkout success'
              }
          }

          stage('Build') {
              steps {
                  sh 'gradle clean build --exclude-task test'
                  echo 'build success'
              }
          }

          stage('Test') {
              steps {
                  sh 'gradle test'
                  echo 'test success'
              }
          }
      }

      post {
          success {
              slackSend (channel: SLACK_CHANNEL, color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", teamDomain: TEAM_DOMAIN, tokenCredentialId: TOKEN_CREDENTIAL_ID )
          }

          failure {
              slackSend (channel: SLACK_CHANNEL, color: '#F01717', message: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", teamDomain: TEAM_DOMAIN, tokenCredentialId: TOKEN_CREDENTIAL_ID )
          }
      }
  }
```

## 📝 GitHub Push해보기
GitHub에 새로운 내용이 push되거나 pr이 생성되면 자동으로 webhook가 실행되고 결과가 표시되는 것을 볼 수 있다.

아래 내용이 실패한 이유는 test가 실패한 것인데 DB 세팅이 로컬 DB로 잡혀있어 모두 실패가 되었다.
![Untitled](https://user-images.githubusercontent.com/85219306/200320544-889686b7-fc82-45a1-a2eb-fae7d05f5127.png)

이 결과는 Jeknins 서버에서도 확인이 가능한데, 아래와 같이 CI가 실행된 결과를 확인할 수 있다.
![Untitled](https://user-images.githubusercontent.com/85219306/200320620-2ac52d24-5a46-42c2-ae27-f998caebb7f0.png)