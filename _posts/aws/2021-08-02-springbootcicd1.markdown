---
layout: post
title: Travis CI를 이용한 SpringBoot 서비스 빌드 자동화
date: 2021-08-02 11:25:05 +0900
category2: cloud
category: Study Note
tag: [aws,travisCI,CI/CD,spring]
img: travis.png 
---
<br>  

  
## Travis CI를 이용해 SpringBoot 서비스 빌드 자동화를 해보자  
  
 
  
<br>  



<h3>최종 목표</h3>

이미지

<br>  
  
----

<h3>현재 글에서의 목표 - CI</h3> 

**<span style='color:blue'>Github - Travis CI - AWS S3</span> 를 연동해
  간단한 스프링부트 서비스를 깃허브에 푸쉬했을 때 Travis CI를 통해 테스트/빌드 하고 빌드 결과 파일을 S3로 전달한다.**  
  
----

<br>  


<h3>시작하기 전</h3>  

**CI(지속적 통합)**

* 버전 관리 시스템에 Push 시 자동으로 테스트와 빌드가 수행되어 안정적으로 배포 파일을 만드는 과정을 말합니다.  
  

**CD(지속적 배포)**

* CI 결과를 자동으로 운영 서버에 무중단 배포까지 진행되는 과정을 말합니다.  
  
  
**Travis CI**

* 깃허브에서 제공하는 무료 CI 서비스이다.  
  

**AWS S3(Simple Storage Service)**

* AWS 에서 제공하는 정적 파일 등을 관리할 수 있는 저장소이다.    

----  
    
<h3>시작합니다!</h3>  
  
<h3>Springboot 프로젝트를 깃허브 저장소에 푸쉬한다.</h3>
  
* 어렵지 않으니 생략! 
  
* 저는  
   * Framework - SpringBoot 2.5.3
   * Build Tool - gradle 7.1.1

<br>  
   

<h3>Travis CI와 Github 연동하기</h3>  
  
* Travis CI 를 Github로 로그인한 후 프로필에서 알맞는 Github 저장소를 활성화 시킵니다.    
  
* 저장소에 푸쉬된 프로젝트의 build.gradle 과 같은 위치에 .travis.yml 을 추가하고 커밋 푸쉬해줍니다.
  
>.travis.yml  

```yml

language: java
jdk:
  - openjdk8

# 마스터 브랜치에 Push 될 때 수행
branches:
  only:
    - master

# Gradle 의존성을 받고 디렉터리에 캐시하여 다음 배포때 사용
cache:
  directories:
    - '$HOME/.m2/repository'
    - '$HOME/.gradle'

# Push 되었을 때 수행할 명령어
script: "./gradlew clean build"

# 실행 완료 후 결과 알람 - Slack도 된다고 한다.
notifications:
  email:
    recipients:
      - ehd0309@naver.com
```    
  
<br>  
  
----
<h3>gradlew?</h3>  

**Gradle Wrapper란?**
>Gradle 빌드에 권장되는 사용 방법은 Gradle Wrapper를 사용하는 것이다. Wrapper는 미리 선언된 버전의 Gradle을 호출하고, 필요한 경우 미리 다운로드한다. Java 프로젝트를 CI 환경에서 빌드할 때 CI 환경을 프로젝트 빌드 환경과 매번 맞춰줄 필요가 없는 이유가 바로 Gradle Wrapper를 사용하기 때문이다. 즉, 환경에 종속되지 않고 프로젝트를 빌드할 수 있는데 이런 점이 Gradle이 가진 강력한 특징중 하나이다.
  
**gradlew?**  
> 유닉스용 wrapper 실행 스크립트이다. 컴파일, 빌드 등을 하는 경우 사용한다.  

ref : [Gradle Wrapper란?](https://jungseob86.tistory.com/21)    

----  

**Travis CI 저장소 페이지에서 로그와 함께 빌드 성공을 알려줍니다.**  
  
![3](https://user-images.githubusercontent.com/76927397/127966890-f94d5f5c-50de-4fdf-913b-e000343afc87.PNG)
    

<br>  
  
  
**만약 로그에서 다음 에러를 나타내며 실패했다면**  
  
 
```shell
 $ ./gradlew assemble
/home/travis/.travis/functions: line 350: ./gradlew: Permission denied 
...
```  

<br>  
  
  
**.travis.yml 에 해당 코드를 추가해줍니다.**  
  
```yml
before_install:
  - chmod +x gradlew
```   

<br>  
  
* 자~ Github와 Travis CI의 연동으로 이제 내가 작성,수정하는 Springboot 프로젝트를 커밋 푸쉬하면 자동으로 테스트/빌드를 진행해줍니다.  
* 배포 자동화를 위한 빌드 결과물을 저장할 AWS S3 버킷을 만들고 연동하여 저장해는것 를! 해보겠습니다.  
  
<br>  
  
<h3>AWS S3 버킷 생성</h3>  
  
* 빌드된 jar 파일을 보관할 S3 버킷을 생성해봅시다.  
  
* S3 콘솔에서 S3 버킷을 생성합니다. 새 ACL 액세스 차단만 해제해줍니다.  
  
  
![4](https://user-images.githubusercontent.com/76927397/127968481-e71fb9b3-f2b4-4c0d-a352-1b0ccdfca3e3.PNG)
  
![6](https://user-images.githubusercontent.com/76927397/127968512-d3b19e5b-c750-4270-98e1-0844f98e3a3f.PNG)
 

<br>  
  
<h3>Travis CI 연결 전용 AWS 사용자 생성 및 연결</h3>  
  
* Travis CI에서(외부에서) AWS를 사용하기 위해서는 AWS에 대한 접근 권한(액세스 키 , 비밀 키)이 있어야 합니다. 
  
* 모든 권한을 가진 루트 사용자 대신   
S3와 나중에 사용하게 될 CodeDeploy 에 대한 권한만 가진 새로운 사용자를 만들어 해당 키를 통해 Travis CI와 연동해보겠습니다
  
* AWS IAM 콘솔에서 사용자를 프로그래밍 방식 액세스 유형으로 생성합니다.  
  
![1](https://user-images.githubusercontent.com/76927397/127969101-97adb297-91ed-4d76-ba5f-9fcb268ce68d.PNG)
  
<br>  
  
* AmazonS3FullAccess , AWSCodeDeployFullAccess 에 대한 권한을 추가해줍니다.   
 
* 사용자 생성에 성공하면 액세스 키와 비밀 키가 생성되는데 비밀 키는 생성 시에만 확인 가능하니 csv로 다운로드해서 잘 보관해둡니다.    
   
![2](https://user-images.githubusercontent.com/76927397/127969317-121d2a50-fa11-4b42-8d4f-b1b831933eff.PNG)

<br>  
  
* 다시 Travis CI 저장소 페이지로 와서 우측 상단의 More options => Settings 를 클릭!     

* Environment Variables 항목에 방금 사용자를 생성하면서 얻은 AWS_ACCESS_KEY와 AWS_SECRET_KEY 를 등록합니다.   
  
> .travis.yml 파일은 깃허브에 그대로 푸쉬되어 누구에게나 보여질텐데  파일에 사용자 정보가 그대로 노출되면 안되겠죠!  
  
![5](https://user-images.githubusercontent.com/76927397/127969786-04cf71b2-fbc2-49ec-a0fe-9763fea00f31.PNG)

<br>  
  
  
* .travis.yml 에 S3 배포용 코드를 추가하고 커밋 푸쉬해줍니다.    
  
>.travis.yml  
  
```yml

before_deploy:
  - zip -r springboot-test-cicd *
  - mkdir -p deploytest
  - mv springboot-test-cicd.zip deploytest/springboot-test-cicd.zip

deploy:
  - provider: s3
    access_key_id: $AWS_ACCESS_KEY 
    secret_access_key: $AWS_SECRET_KEY
    bucket: springboot-deploy-test-bucket # 생성했던 S3 버킷 이름
    region: ap-northeast-2
    skip_cleanup: true
    acl: public_read
    local_dir: deploytest #before_deploy 에서 만든 디렉터리 이름
    wait-until-deployed: true
    on:
      repo: pds0309/springboot-test-cicd #깃허브 레포지토리 주소
      branch: master
```    
  
<br>  
  
* 빌드가 성공하면 S3의 지정한 버킷에 빌드 결과물이 저장된 것을 확인할 수 있습니다. 
  
  
![7](https://user-images.githubusercontent.com/76927397/127970230-e9950a67-12d2-443d-aa09-cbb6d18fcee2.PNG)
![8](https://user-images.githubusercontent.com/76927397/127970255-e963cc8e-51fe-48be-9e4f-be099336b6e2.PNG)
  
  
<h3> CI 끝! </h3>  
  
* 스프링부트 서비스를 테스트/빌드 자동화하고 S3 버킷에 저장하여 배포 자동화를 위한 준비를 끝마쳤습니다.    
  
* 다음 글에서 S3에 저장된 빌드된 jar 파일을 AWS CodeDeploy 를 이용하여 내 인스턴스(서버)로 전달하여 배포 자동화를 해보도록 하겠습니다.  
    
* 저는 AWS의 EC2 가 아닌 Oracle Cloud 의 인스턴스로 배포 자동화를 해 볼 생각입니다.
  
* **부족한 글 읽어주셔서 감사합니다.**  
  
 
<br>  

----  
    

**Reference**  
  
* [TravisCI & AWS CodeDeploy로 배포 자동화 구축하기](https://jojoldu.tistory.com/265)  
  
* [Gradle Wrapper란?](https://jungseob86.tistory.com/21)  
  


