---
title: "[DeliveryFood] 안정적인 배포 자동화를 위한 Jenkins CD 적용하기"
excerpt: "CD 적용으로 main 브랜치에 머지될때마다 자동으로 배포하기"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood, Jenkins, 자동화, CD]

toc: true
toc_sticky: true
 
date: 2022-11-12
last_modified_at: 2022-11-12
---

- CD는 지속적인 서비스 제공(Continuous Delivery) 또는 지속적인 배포(Continuous Deployment)를 의미한다.
- CI의 연장선이며 개발자의 변경 사항을 리포지토리에서 프로덕션 환경까지 자동으로 배포하는 것을 의미한다.

CI를 통해 통합된 코드는 에러 없이 동작 가능하다는게 증명된다. 그럼 이제 증명된 애플리케이션을 일반 사용자가 사용할 수 있도록 서버에 배포 해야 한다.
물론 배포는 개발자가 직접 실행할 수도 있지만, 반복되는 작업은 많아질것이고 비효율적인 시간도 늘어날 것이다. 이를 도와주기 위해 배포 자동화가 필요하다.

이전에 CI 적용이 끝났고, 이번엔 CD를 적용해보려고 한다. 프로젝트는 `Github Flow 브랜칭 전략`을 따르고 있기 때문에 main 브랜치에 push나 merge가 이루어지면 배포되도록 하려고 한다.
CI를 적용했을 때처럼 Jenkins를 사용하며 기본 세팅부터가 아닌 CI 세팅이 되어 있다는 가정하에 적용을 해보았다.

🔗 관련 포스트 참고  
Docker위에 Jenkins를 설치하고 Docker 이미지를 배포할 수 있다.  
[Docker란?](https://haenlee.github.io/cs/etc/cs-etc-06/)
{: .notice--success}

📢 **배포 과정**
![image](https://user-images.githubusercontent.com/85219306/201474564-fd4d9830-86ea-41db-85c8-6aa6e9fc86aa.png)

<br>
# 🚀 CD 준비
---
## 📝 서버 생성
가장 먼저 필요한 것은 배포 서버가 준비되어야 한다. 배포 서버는 NCloud에서 제공해주는 서비스를 이용했고, 배포에 사용될 서버를 하나 생성해주었다.
서버를 생성하고 서버의 방화벽 설정과 외부에서 접속을 확인하기 위한 공인IP도 따로 신청해주었다.

> [네이버 클라우드 서버 생성](https://guide.ncloud-docs.com/docs/compute-compute-1-1-v2)

## 📝 Java 설치
서버가 생성되었으면 .jar 파일을 실행하기 위한 JDK를 설치해주어야 한다. 

- JDK 11 설치
```sql
yum install -y java-11-openjdk-devel.x86_64
```

- Java 버전 확인
```sql
java --version
```

- 환경 변수 설정
    - Java 경로 확인  
    `which` 명령으로 실행 파일 경로를 확인하고, `readlink` 명령으로 심볼릭 링크 파일의 원본 경로를 확인한다.

        ```sql
        which java
        ```

        ```sql
        readlink -f /bin/java
        ```
    위 명령어를 실행하면
        ```sql
        /usr/lib/jvm/java-11-openjdk-11.0.17.0.8-2.el7_9.x86_64/bin/java
        ```
경로를 알 수 있는데 전체 경로에서 `/bin/java를 제외한 디렉토리 경로`를 환경 변수로 설정한다.

    - 환경 변수 등록
        ```sql
        vi /etc/profile
        ```
    vi 에디터를 실행 한후 맨 아래에 환경 변수를 추가해준다.
        ```sql
        ...
        (중략)
        ...
        export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.17.0.8-2.el7_9.x86_64
        ```

<br>
# 🚀 배포 서버 접근 설정
---
## 📝 Jenkins 서버 SSH key 생성
Jenkins 서버에서 배포 서버로 접근하기 위해서는 배포 서버에 Jenkins 서버의 SSH public key가 필요하다.

- SSH key 생성  
Jenkins 서버에서 key를 생성한다.
```sql
ssh-keygen
```
명령어를 입력하면 /home/jenkins/.ssh 경로에 `id_rsa, id_rsa.pub` 파일 2개가 생성된다.
    - id_rsa: private key  
    절대로 타인에게 노출이 되면 안된다.
    - id_rsa.pub: public key  
    접속하려는 원격 버서의 authorized_key에 입력한다.

- public key 복사
```sql
cat /home/jenkins/.ssh/id_rsa
```
명령어를 통해 나온 key는 나머지 세팅 과정에서 필요하다.

## 📝 배포 서버 authorized_key 설정
비밀번호 대신 원격 서버에 접속하려면 .ssh 디렉토리에 authorized_key 파일이 필요하다.
- .ssh 디렉토리가 없는 경우  
.ssh 디렉토리를 생성해준다.
```sql
mkdir ~/.ssh
```

- authorized_key 등록
```sql
vi ~/.ssh/authorized_key
```
authorized_key 파일에 Jenkins 서버에서 발급받은 `id_rsa.pub`를 입력한다.

- 권한 설정
.ssh 디렉토리는 중요한 정보가 담긴 디렉토리이기 때문에, permission 권한을 수정해줘야 한다.
```sql
chmod 700 ~/.ssh
chmod 644 ~/.ssh/authorized_keys
```

<br>
# 🚀 Jenkins 'Publish Over SSH' 플러그인 설치
---
## 📝 플러그인 설치
`Dashboard > Jenkins 관리 > 플러그인 관리` 에서 Publish Over SSH를 검색한 후 설치한다.
![image](https://user-images.githubusercontent.com/85219306/201477601-740782e0-3f0d-4040-8689-60e9dfb5aab0.png)

## 📝 플러그인 세팅
`Dashboard > Jenkins 관리 > 시스템 설정` 에서 하단의 Publish over SSH로 이동한다.
![image](https://user-images.githubusercontent.com/85219306/201478318-b81718a7-736c-4ced-b11f-4dbab2313f9b.png)
- Name: SSH Server를 구분할 수 있는 이름
- HostName: SSH 이용해 파일을 전송하려는 원격 서버(NCloud 기준 서버 접속용 공인 IP)
- Username: 원격 서버 계정명
- Remote Directory: 원격 서버 디렉토리 입력
- key: Jenkins 서버에서 복사해놓은 id_rsa 

📢 `Test Configuration`을 클릭하여 정상적으로 인증에 성공하는지 테스트하기
![image](https://user-images.githubusercontent.com/85219306/201478856-cdc377a8-0b48-4801-a3c6-cb5da2b492f1.png)

<br>
# 🚀 배포 서버 .sh 스크립트 생성
---
빌드가 완료되어 .jar 파일이 배포 서버에 전달되고 난 후 실행되는 스크립트가 필요하다.
스크립트는 프로젝트명으로 실행되고 있는 프로세스가 있다면 프로세스를 중단하고 .jar를 실행한다.  

스크립트의 위치는 배포 서버 루트 경로 하위에 미리 /deploy 디렉토리를 만들어 두었고, /deploy에 .jar와 .sh 스크립트가 위치하게 했다.

```console
#!/bin/bash

echo "PID Check..."

CURRENT_PID=$(ps -ef | grep java | grep "delivery-food"* | awk '{print $2}')

echo "$CURRENT_PID"

if [ -z CURRENT_PID ]; then
    echo "Project is not running"
else
    kill -9 $CURRENT_PID  
    sleep 5
fi

echo "Deploy Project...."

nohup java -jar /root/deploy/delivery-food-0.0.1-SNAPSHOT.jar > /root/deploy/nohup.out 2>&1 &

echo "Done"
```

<br>
# 🚀 Jenkinsfile 생성
---
`Dashboard > [item 이름] > pipeline Syntax`의 Steps에서 `sshPublish: Send build artifacts over SSH`를 선택한다.
![image](https://user-images.githubusercontent.com/85219306/201479953-eb4a9c7c-f577-4017-af9f-84160eb7bd6a.png)
- Source files: 원격 서버로 보낸 파일의 경로를 입력  
    - ant 패턴으로 `**/*.jar` 도 가능
- Remote prefix: Source file에서 지정한 경로에서 제거할 prefix를 지정
- Remote directory: Source file이 저장될 원격 서버의 경로
- Exec command: 파일 전송이 모두 끝난 이후 원격 서버에서 실행할 명령어

📢 'Generate Pipeline Script'를 클릭하여 나온 내용을 Jenkinsfile에 작성해준다.
![image](https://user-images.githubusercontent.com/85219306/201480009-034bc974-5816-4fad-a317-56f285ace954.png)

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

        stage('Deploy') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'cd-server',
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false,
                                    excludes: '',
                                    execCommand: 'sh /root/deploy/run.sh',
                                    execTimeout: 120000,
                                    flatten: false,
                                    makeEmptyDirs: false,
                                    noDefaultExcludes: false,
                                    patternSeparator: '[, ]+',
                                    remoteDirectory: '/deploy',
                                    remoteDirectorySDF: false,
                                    removePrefix: 'build/libs',
                                    sourceFiles: 'build/libs/*jar'
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: true
                        )
                    ]
                )
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

<br>
# 🚀 CD 테스트 해보기
---
main 브랜치에 push나 merge가 이루어지면 이런식으로 자동으로 배포가 이루어짐을 알 수 있다.
![image](https://user-images.githubusercontent.com/85219306/201480219-aea227fe-107a-4443-a098-e62a51476475.png)

또한 배포 서버로 접속시 미리 설정해 놓은 index.html이 정상적으로 노출되는것을 확인할 수 있다.
![image](https://user-images.githubusercontent.com/85219306/201480796-c9f3457c-803d-4e33-9ffd-e05169146939.png)




<br>
# 🚀 마무리
---
1. 프로젝트 개발
2. Github에 push
3. Jenkins에서 build
4. build한 결과 jar파일을 web server에 SSH 송신
5. web server에서 jar파일 실행(배포)

위 과정으로 배포가 이루어지도록 하였다.

Jenkins로 배포 과정을 만들며 도커를 적용해볼지 고민이 많았었는데, 간단하게나마 CD를 적용해보는게 배포 과정을 이해하는데 도움이 될 것 같았다.
앞으로의 계획은 시간이 되는데로 도커를 적용해 보거나 로드 밸런싱을 적용해 무중단 배포를 해보려고 한다. 

> [[Jenkins] Jenkins - Spring Boot 프로젝트 jar 배포](https://dev-overload.tistory.com/39)  
> [[Jenkins] 빌드부터 배포까지 3 - SSH 업로드와 스크립트 실행](https://onethejay.tistory.com/151)  
> [[Jenkins] Jenkins를 이용한 배포 자동화(3) - Publish Over SSH 를 이용한 원격 서버 배포](https://minsoolog.tistory.com/45)