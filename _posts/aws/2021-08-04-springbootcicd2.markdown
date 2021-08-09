---
layout: post
title: Travis CI, CodeDeploy를 이용한 온프레미스로의 배포 자동화 
date: 2021-08-04 10:11:11 +0900
category2: cloud
category: Study Note
tag: [aws,travisCI,CI/CD,linux,codedeploy,aws-cli]
img: travis.png 
---
<br>  

  
## Travis CI와 AWS CodeDeploy 를 이용해 SpringBoot 서비스를 AWS의 EC2가 아닌 컴퓨팅 플랫폼에 자동 배포해보자   
  


----  

**1편 - [Travis CI를 이용한 SpringBoot 서비스 빌드 자동화](/springbootcicd1/)**
  

---- 

<br>  

[이 글](https://jojoldu.tistory.com/265)을 많이 참고하여 작성한 글입니다.

<h3>1편에서 목표</h3>  
  
**Github - Travis CI - AWS S3 를 연동해 간단한 스프링부트 서비스를 깃허브에 푸쉬했을 때 Travis CI를 통해 테스트/빌드 하고 빌드 결과 파일을 S3로 전달한다.**
  
<br>
  

  
<h3>현재글에서의 목표</h3>
  
![2-3](https://user-images.githubusercontent.com/76927397/128458749-e0a7ec21-278e-40a9-b365-de278c528259.PNG)
    
**AWS S3로 전달된 자동화된 빌드 결과물을 온프레미스(또는 다른 클라우드) 인스턴스로 배포 자동화 해보자**  

* 저는 Oracle Cloud VM 인스턴스로 진행하였습니다. (Ubuntu 20.04) 

----  
<br>  


<h3>시작하기 전</h3>  

**온프레미스(on-premise)**

* 소프트웨어 등 솔루션을 클라우드 같이 원격 환경이 아닌 자체적으로 보유한 전산실 서버에 직접 설치해 운영하는 방식을 말합니다.
  

**CodeDeploy**

* Amazon EC2 인스턴스, 온프레미스 인스턴스, 서버리스 Lambda 함수 또는 Amazon ECS 서비스로 애플리케이션 배포를 자동화하는 배포 서비스입니다. [ref](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/welcome.html)
    

<br>  
  


  

----    


<h3>인스턴스 설정하기</h3>  
  
**S3로부터 zip 파일을 받아올 디렉터리 만들기**  
  
>Travis CI로 빌드 하고 성공하여 S3 에 저장된 zip 파일을 build 폴더로 압축 해제하여 복사될 예정.  
  
```shell
sudo mkdir /home/ubuntu/springboot
sudo mkdir /home/ubuntu/springboot/build
```  

<br>  



**인스턴스에서 Codedeploy 이벤트를 처리할  수 있게 CodeDeploy Agent 설치**

>의존 패키지 설치
  
```shell
sudo apt update
sudo apt install ruby-full
sudo apt install wget  
```

<br>  

> Codedeploy 에이전트 설치  
> wget https://[리소스 킷 버킷].s3.[리전].amazonaws.com/latest/install  
    
&nbsp;[CodeDeploy 리소스 킷 참고](https://docs.aws.amazon.com/codedeploy/latest/userguide/resource-kit.html#resource-kit-bucket-names)  

```shell
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
```  
```shell
sudo ./install auto > /tmp/logfile
```  
  
<br>  
  
**이렇게 나오면 설치 성공!** 
  
![2-4](https://user-images.githubusercontent.com/76927397/128467179-8bc516da-9cce-40f8-8765-5aa1d1c9b800.PNG)


<br>  

> 인스턴스 부팅 시 자동 실행을 위해 쉘 스크립트 파일을 /etc/init.d/ 에 추가해줍니다.
  
```shell
sudo vim /etc/init.d/codedeploy-startup.sh    
```
```shell
#!/bin/bash
sudo service codedeploy-agent start
```

<br>  

      
**AWS CLI 설치**
 
* 인스턴스에서 AWS 명령어를 사용할 수 있게 해줍니다.  
 
> 링크로부터 패키지를 다운로드하고 압축을 해제합니다.
 
```shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
```  

> 설치 프로그램을 실행하여 설치하면서 /usr/local/bin 에 심볼링크를 생성합니다.  

```shell
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin
```  
  
<br>  

**버전이 잘 나오면 설치 성공!**   
  
![2-5](https://user-images.githubusercontent.com/76927397/128468685-cf0d876c-6b66-4e46-9cbe-ecf6113ab942.PNG)

<br>  
  
  
**AWS 사용자 등록**  
  
>[1편에서 만들었던 사용자](/springbootcicd1#cicd1user)를 등록해줍니다.   

```shell  
sudo aws configure
```  
```  
AWS Access Key ID [None]: KEY
AWS Secret Access Key [None]: SECRETKEY
Default region name [None]: REGION(ex : ap-northeast-2)
Default output format [None]: json
```

<br>  
  
**온프레미스 인스턴스에 구성파일 추가하기**  
  
* [1편에서 만들었던 사용자](/springbootcicd1#cicd1user)로 CodeDeploy 에 온프레미스 인스턴스 등록을 위해 conf.onpremises.yml 파일을 추가해야합니다.  

* EC2 같은 경우 EC2 인스턴스에 적절한 역할을 부여해준 후 태그를 통해 CodeDeploy 애플리케이션으로 연결할 수 있지만 온프레미스 또는 다른 클라우드 인스턴스의 경우 CodeDeploy의 온프레미스 인스턴스에 따로 인스턴스를 등록해주어야 합니다.    

> Ubuntu 기준 설정 파일입니다. 다른 인스턴스의 경우 [참고](https://docs.aws.amazon.com/codedeploy/latest/userguide/register-on-premises-instance-iam-user-arn.html)  
    

> conf.onpremises.yml 을 /etc/codedeploy-agent/conf 에 추가해줍니다.
  

```shell  
sudo vim /etc/codedeploy-agent/conf/conf.onpremises.yml
```
  
```yml
---
aws_access_key_id: secret-key-id
aws_secret_access_key: secret-access-key
iam_user_arn: iam-user-arn
region: supported-region
```
  
<br>  
  
  
**AWS_REGION 환경 변수 설정**  
  
 
```shell
export AWS_REGION=supported-region
```

<br>  
  
**AWS CLI 를 이용하여 CodeDeploy 에 온프레미스 인스턴스 등록**  
   

>  CodeDeploy에 온프레미스 인스턴스를 등록해줍니다.  
 
```shell
aws deploy register-on-premises-instance --instance-name [인스턴스이름] --iam-user-arn [사용자ARN]
```
  
> 저는 ubuntudeploy1 로 인스턴스 이름을 지정했는데 CodeDeploy 콘솔에서 생성된 것을 확인할 수 있습니다.  
  
![2-6](https://user-images.githubusercontent.com/76927397/128654756-35b29cc3-50ee-4eed-b751-adc1693dd84a.PNG)
  
<br>  
  
> 잘 생성되었다면 태그를 지정해줍니다. CodeDeploy 애플리케이션을 생성할 때 이 태그를 사용하여 배포 대상을 식별할 수 있습니다.  
  
  
```shell  
aws deploy add-tags-to-on-premises-instances --instance-names [인스턴스이름] --tags Key=Name,Value=[식별값]
```   
<br>  

>저는 다음과 같이 지정하였습니다.  
  
```shell
aws deploy add-tags-to-on-premises-instances --instance-names ubuntudeploy1 --tags Key=Name,Value=ubuntudeploy1val
``` 
  

<br>  
  
<h3>IAM으로 서비스 역할 생성하기</h3> 
  
* 다양한 AWS 서비스들은 사용자가 아닌 역할을 이용해 서비스의 리소스에 접근하여 작업을 수행합니다.  
  
* 이를 서비스 역할이라고 하는데 CodeDeploy 또한 역할을 연결하여 작업을 수행하여야 하기에 서비스 역할을 생성해봅시다.
  
* 인스턴스에서 AWS CLI를 사용해 역할을 만들 수 있지만 iam:CreatRole 과 같은 또 다른 권한을 사용자에 매핑해야하고 복잡하니 콘솔에서 생성해봅시다.  
  
* [IAM 사용자에 대한 서비스 Role 생성,수정,삭제 권한에 대한 문서 링크](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html)
    
<br>  
  
**IAM 콘솔에서 역할 만들기 -> AWS 서비스 선택 -> 사용사례에서 CodeDeploy 를 선택하고 역할을 생성합니다.**   
  
![2-7](https://user-images.githubusercontent.com/76927397/128656265-a1cd1dac-d57c-4bcd-ba77-3842661e6f9c.PNG)
  
<br>  
  
![2-8](https://user-images.githubusercontent.com/76927397/128656295-895f086a-f180-4fc3-b212-9909e7c9cc4e.PNG)
  
<br>  
  

**사용자를 대신해 역할을 통해서 인스턴스에서 CodeDeploy 를 작동시킬 수 있게 허용이 되어있어야 합니다.**

> 역할 선택-> 신뢰관계 -> 신뢰관계 편집에서 해당 부분이 없다면 추가해줍니다. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "codedeploy.[리전].amazonaws.com",
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```  

<br>  


**사용자가 우리가 만든 UbuntuDeploy1-CodeDeployRole 역할에 접근할 수 있도록 접근 권한을 매핑해줍니다.**  

> 해당 정책을 생성해주고 인스턴스에 등록했던 사용자에 연결해줍니다.  

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:GenerateCredentialReport",
                "iam:GenerateServiceLastAccessedDetails",
                "iam:Get*",
                "iam:List*",
                "iam:SimulateCustomPolicy",
                "iam:SimulatePrincipalPolicy"
            ],
            "Resource": "[역할 ARN]"
        }
    ]
}
```
 
> 인스턴스에서 역할을 잘 얻어올 수 있는지 확인해봅니다.  
  
```shell
 aws iam get-role --role-name UbuntuDeploy1-CodeDeployRole
```
 
```shell
{
    "Role": {
        "Path": "/",
        "RoleName": "UbuntuDeploy1-CodeDeployRole",
        "RoleId": ,
        "Arn": "[ARN]",
        "CreateDate":  ,
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "",
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "codedeploy.ap-northeast-2.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        },
        "Description": "CodeDeploy Role For Ubuntu Instance",
        "MaxSessionDuration": 3600,
        "RoleLastUsed": {}
    }
}

```  
  
> 저 같은 경우는 역할 접근 권한을 추가하지 않고 배포할 때 Codedeploy agent 의 로그에서 다음과 같은 에러가 났었습니다.

```shell
INFO  [codedeploy-agent(42277)]: [Aws::CodeDeployCommand::Client 400 0.049208 0 retries] poll_host_command(host_identifier:"ROLE_ARN") Aws::CodeDeployCommand::Errors::AccessDeniedException 
```

<br>   

  
<h3>CodeDeploy 애플리케이션 생성</h3>  
  
**CodeDeploy 콘솔의 시작하기->애플리케이션 생성으로 애플리케이션을 생성해줍니다.**  
  
  
![2-9](https://user-images.githubusercontent.com/76927397/128658449-29736bdc-e591-4cef-abb2-e633afc75deb.PNG)
  
  
<br>  
  
**배포 그룹 생성도 해줍니다. 환경구성에서 온프레미스를 선택하고 이전에 등록했던 온프레미스 인스턴스의 태그를 입력해줍니다.**  

![2-10](https://user-images.githubusercontent.com/76927397/128658832-3ed99631-b797-419a-ac01-9ce83692fa67.PNG)

![2-11](https://user-images.githubusercontent.com/76927397/128658645-e865db4f-e0c4-4071-8bab-326bf0fa192e.PNG)
  
**기타설정**  
  
* 배포 설정 : CodeDeployDefault.OneAtTime
* 로드밸런서 : 로드 밸런싱 활성화 체크 해제  
  
<br>  
  


<h3>프로젝트에 CodeDeploy 설정하기</h3>  
  
**appspec.yml 을 .travis.yml 과 같은 위치에 생성해줍니다.**  

>appspec.yml  

```yml
version: 0.0
os: linux
files:
#S3 버킷 복사할 파일 위치
  - source:  /
#인스턴스 내에 zip 파일 복사해 압축 풀 위치
    destination: /home/ubuntu/springboot/build 
``` 
  
<br>  
  

**.travis.yml 설정 추가하기**  
  
> 이전에 추가했던 s3 설정 밑부분에 해당 설정을 추가해줍니다.

```yml
- provider: codedeploy
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY 
    bucket: springboot-deploy-test-bucket # S3 버킷
    key: springboot-test-cicd.zip # 빌드 파일 압축해서 전달
    bundle_type: zip
    application: springboot-deploytest-app # aws 콘솔에서 등록한 CodeDeploy 어플리케이션
    deployment_group: springboot-deploytest-group # aws 콘솔에서 등록한 CodeDeploy 배포 그룹
    region: ap-northeast-2
    wait-until-deployed: true
    on:
      repo: pds0309/springboot-deploy-test
      branch: master
```
  
<br>  
  
  
**.travis.yml 최종 코드**  
  
```yml
language: java
jdk:
  - openjdk8

branches:
  only:
    - master

before_install:
  - chmod +x gradlew


cache:
  directories:
    - '$HOME/.m2/repository'
    - '$HOME/.gradle'

script: "./gradlew clean build"


before_deploy:
  - zip -r springboot-test-cicd *
  - mkdir -p deploytest
  - mv springboot-test-cicd.zip deploytest/springboot-test-cicd.zip

deploy:
  - provider: s3
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY
    bucket: springboot-deploy-test-bucket
    region: ap-northeast-2
    skip_cleanup: true
    acl: public_read
    local_dir: deploytest 
    wait-until-deployed: true
    on:
      repo: pds0309/springboot-deploy-test
      branch: master
  - provider: codedeploy
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY 
    bucket: springboot-deploy-test-bucket 
    key: springboot-test-cicd.zip 
    bundle_type: zip
    application: springboot-deploytest-app
    deployment_group: springboot-deploytest-group
    region: ap-northeast-2
    wait-until-deployed: true
    on:
      repo: pds0309/springboot-deploy-test
      branch: master 
      
notifications:
  email:
    recipients:
      - ehd0309@naver.com

```

<br>  
  
**코드를 추가한 뒤 커밋,푸쉬 해줍니다. CodeDeploy 콘솔의 배포 창에서 진행과정을 상세하게 확인할 수 있습니다.**  
  
  
![2-12](https://user-images.githubusercontent.com/76927397/128660422-4c472338-4a91-4ed9-925b-0d7aff68d25a.PNG)
  
<br>  
  
**내 인스턴스에도 빌드 성공 후 자동으로 전송되어있는 것을 알 수 있습니다.**  
 
![2-14](https://user-images.githubusercontent.com/76927397/128660470-2cde375b-78c4-483b-8f06-f5a146ce8bbb.PNG)

  
<br>  
  
<h3>배포 자동화 하기</h3> 
  
* 깃 푸시로 CodeDeploy 와 S3를 통해 인스턴스까지 빌드결과를 전달하였지만 아직까지는 빌드만 자동화되어있다고 볼 수 있습니다.  
  
* 모든 과정을 성공적으로 마치고 jar 파일을 얻었다면 이를 자동으로 실행시켜줄 스크립트를 작성해봅시다.  
  
  
**Jar 파일 보관 디렉터리 생성**
   
```shell
sudo mkdir /home/ubuntu/springboot/jar
```
  
**빌드파일 실행용 deploy.sh 생성**  

```shell
sudo vim /home/ubuntu/springboot/deploy.sh
```  
  
> deploy.sh  
  
```shell
#!/bin/bash

REPOSITORY=/home/ubuntu/springboot

echo "> confirm springboot app's pid is running"

CURRENT_PID=$(pgrep -f testcicd)

echo "$CURRENT_PID"

if [ -z $CURRENT_PID ]; then
    echo "> there is no app that is running."
else
    echo "> kill -15 $CURRENT_PID"
    sudo kill -15 $CURRENT_PID
    sleep 5
fi

echo "> New Deploy"

echo "> Build File Copy"

sudo cp $REPOSITORY/build/build/libs/*.jar $REPOSITORY/jar/

JAR_NAME=$(ls $REPOSITORY/jar/ |grep 'testcicd' | tail -n 1)

echo "> JAR Name: $JAR_NAME"

```    
  
* 권한이 있으시다면 sudo 빼셔도 됩니다.
    
**실행권한 주기**  
  
```shell
sudo chmod +x deploy.sh
```  
  
**인스턴스가 순정상태라 jdk가 없어서 설치해주었습니다.**  
  
```shell
sudo apt-get update
sudo apt-get install openjdk-8-jdk
```
  
  
**deploy.sh를 실행하여 실행이 잘 되는지, 이미 실행되고 있으면 종료하고 다시 실행이 잘 되는지 확인합니다.**  
  
```shell
ubuntu@instance-deployinstance:~/springboot$ ps -ef | grep testcicd
ubuntu    103399       1 82 04:51 pts/0    00:00:45 java -jar /home/ubuntu/springboot/jar/testcicd-0.0.2-SNAPSHOT.jar
ubuntu    103435   97350  0 04:52 pts/0    00:00:00 grep --color=auto testcicd
ubuntu@instance-deployinstance:~/springboot$ sudo iptables -I INPUT 1 -p tcp --dport 8080 -j ACCEPT
ubuntu@instance-deployinstance:~/springboot$ /home/ubuntu/springboot/deploy.sh
> confirm springboot app's pid is running
103399
> kill -15 103399
> New Deploy
> Build File Copy
> JAR Name: testcicd-0.0.2-SNAPSHOT.jar
ubuntu@instance-deployinstance:~/springboot$ nohup: appending output to 'nohup.out'
```

<br>  
  
**배포 성공 시 deploy.sh를 실행하도록 appspec.yml 설정 추가**  
    
> appspec.yml  

```yml
hooks:
  AfterInstall:
    - location: rundeploy.sh
      timeout: 180
```  
  

**rundeploy.sh 를 프로젝트의 yml들과 같은 위치에 생성해줍니다.**  
  
> rundeploy.sh

```shell
#!/bin/bash
/home/ubuntu/springboot/deploy.sh > /dev/null 2> /dev/null < /dev/null &
```   

* /dev/null 2> /dev/null < /dev/null 부분은 로그 같은 내용을 남기지 않도록 하는 처리라고 합니다.

* 저 같은 경우는 해당 부분 없이 백그라운드로만 실행하는 코드를 작성했을 때 AfterInstall 이벤트에서 다음같은 에러가 발생했었습니다.  
  
![2-15](https://user-images.githubusercontent.com/76927397/128662468-1be1c3df-2afb-4bd8-ad98-7d747549faa5.PNG)
  
<br>  
    
  
**모든 설정을 완료하였습니다. 커밋 푸쉬해줍니다.**  

* 현재 인스턴스에는 testcicd-0.0.2 버전이 실행되고 있는데 0.0.3으로 바꿔주고 푸쉬했을 때 자동으로 변경되나 확인해본 결과 배포가 정상적으로 되었습니다! 
  
```shell
ubuntu@instance-deployinstance:~/springboot$ ps -ef | grep testcicd
root      103551       1 36 05:17 ?        00:00:48 java -jar /home/ubuntu/springboot/jar/testcicd-0.0.3-SNAPSHOT.jar
ubuntu    103588   97350  0 05:19 pts/0    00:00:00 grep --color=auto testcicd
```


<br>  
<br>


<h3>끝~!</h3>  

* CodeDeploy 를 이용해 같은 AWS의 인스턴스가 아닌 다른 클라우드 또는 온프레미스 인스턴스에 배포 자동화 하기 성공!  
  
> 시작 전에는 어려워 보였지만 일부 설정만 추가하는 매우 간단한 과정이었습니다.(11번 만에 성공한건 비밀) 핵심은 역할과 사용자 권한 적절하게 설정하기, CodeDeploy 에 온프레미스 인스턴스 등록하기였다고 할 수 있습니다. 


* 이제 스프링부트 프로젝트를 깃허브 레포지토리 master 브랜치로 푸쉬하기만 하면 빌드부터 배포까지 자동으로 진행됩니다.  
  
* 새로운 배포 시 kill  로 기존 스프링부트 프로세스를 종료해주고 다시 실행시키기 때문에 배포하는 동안 서비스가 일시적으로 중단 된다는 점을 빼면 목표를 달성하였습니다!

* **길고 부족한 내용의 글 읽어주셔서 감사합니다!**
  
<br>  

----  
  

**Reference**  

* [TravisCI & AWS CodeDeploy로 배포 자동화 구축하기](https://jojoldu.tistory.com/265)  
  
* [AWS doc - Install the CodeDeploy agent for Ubuntu Server](Install the CodeDeploy agent for Ubuntu Server)  
  
* [AWS doc - Install AWS CLI on Linux](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2-linux.html)

* [AWS doc - Create a service role for CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/getting-started-create-service-role.html)

* [AWS doc - Creating a role to delegate permissions to an AWS service](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html)  
  
* [AWS doc - Use IAM-Arn to Register an on-premises instance](https://docs.aws.amazon.com/codedeploy/latest/userguide/register-on-premises-instance-iam-user-arn.html)

* [aws/aws-codedeploy-agent - High memory consumption](https://github.com/aws/aws-codedeploy-agent/issues/32) (나중에 읽어보기 위해 기록) 


 