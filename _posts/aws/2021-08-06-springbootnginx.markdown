---
layout: post
title: Nginx를 이용한 Springboot 서비스 무중단 배포
date: 2021-08-06 16:27:34 +0900
category2: cloud/server
category: Study Note
tag: [travisCI,CI/CD,linux,spring]
img: nginx.jpg
---
<br>  

  
## Nginx를 이용해 Springboot 서비스를 중단 없이 배포해보자!
  


----  

**1편 - [Travis CI를 이용한 SpringBoot 서비스 빌드 자동화](/springbootcicd1/)**  

**2편 - [Travis CI, CodeDeploy를 이용한 온프레미스로의 배포 자동화](/springbootcicd2/)**  

---- 

<br>  

[이 글](https://jojoldu.tistory.com/267)을 참고하여 작성한 글입니다.
  
  

<h3>현재까지의 상황</h3>  

![2-1](https://user-images.githubusercontent.com/76927397/128670271-41bc909d-9ee4-4269-98c7-e76ea43137a0.PNG)
  
**Github - Travis CI - AWS S3 - CodeDeploy 를 연동해 스프링부트 서비스를 깃허브에 푸쉬했을 때 Travis CI를 통해 테스트/빌드 하여 S3에 저장된 빌드 결과물을 CodeDeploy를 통해 인스턴스로 배포 자동화 한다.**
  
<br>
  
  
<h3>문제점</h3>  
  
**새로운 배포가 완료되기 전 까지 기존에 배포되었던 프로세스가 종료된다(서비스가 중단된다.)**  

<br>  
  


----  
<br>  


<h3>시작하기 전</h3>  

**Nginx**

* 비동기 이벤트 기반구조의 웹서버 소프트웨어로 가벼움과 높은 성능을 목표로 한다.  
  
* Apache가 스레드 / 프로세스 기반으로 요청 1개에 스레드 1개가 처리되는 구조라면 Nginx는 Event-Driven 기반으로 여러 커넥션을 모두 EventHandler를 통해 비동기 방식으로 처리하여 먼저 처리되는 것부터 로직이 실행되게 하여 다수 연결에 효과적이다.  
  
* 활용  
  * HTTP Server

  * Reverse proxy server  - 클라이언트 요청을 애플리케이션 서버에 배분한다.
  
<br>  

**(Forward)Proxy**
  

![proxy](https://user-images.githubusercontent.com/76927397/128674550-cab188f8-4f51-4e38-ad3f-e8d74b3e65b0.png)
<small> img by wikipedia </small>  
  

* 프록시 서버는 클라이언트가 자신을 통해서 다른 네트워크 서비스에 간접적으로 접속할 수 있게 해 주는 컴퓨터 시스템이나 응용 프로그램을 가리킨다.
    
* 클라이언트가 서버에 서비스를 요청할 때 프록시 서버가 요청을 가로채 중간자로써 클라이언트를 대신해 웹 서버와 통신하여 결과를 제공한다.
    
* 클라이언트 요청 시 인터넷 보다 프록시 서버를 먼저 호출한다.

<br>  
  

**Reverse Proxy**  
  
![reverseproxy](https://user-images.githubusercontent.com/76927397/128674708-1fea7d44-44ca-451b-a4b9-29ea90a9a307.png)
<small> img by wikipedia </small>  
  

  
* 다수의 서버를 Proxy 서버 하단에 위치시키고 클라이언트 요청을 내부서버에 전달해주는 Proxy
 
* 클라이언트는 웹 서버의 주소가 아닌 Reverse Proxy로 설정된 주소로 요청을 하게 되고, Proxy 서버가 받아서  뒷단에 있는 웹 서버에게 다시 요청을 하는 방식으로 클라이언트는 진짜 웹 서버의 정보를 알 수가 없다.
  

>어렵다...
  

<br>  
  
  
  
<h3>구동 과정 2줄 요약</h3>  
  
  
* 사용자가 Nginx 주소로 접속한다. (80) 현재 정상 구동중인 스프링부트로 요청을 전달한다.  
  
* 새로운 배포 시 구동중이지 않은 스프링부트로 배포하고 배포가 정상 완료되면 사용자 접속 시 Nginx가 새로운 버전의 스프링부트를 요청하게끔 바꿔준다. 
  
  


----    
  
<br>  


<h3>인스턴스에 Nginx 설치</h3>  
  
> Ubuntu 20.04을 기준으로 진행하였습니다.
  
  
**Nginx 설치 및 실행**  
  
```shell
sudo apt install nginx
sudo service nginx start
```    

```shell
ubuntu@instance-deployinstance:/etc/nginx$ nginx -v
nginx version: nginx/1.18.0 (Ubuntu)
```  
  
  
* 80 포트를 열고 인스턴스 주소로 접속했을 때 Welcome to Nginx 페이지가 나오면 성공!  
  
<br>  

**리버스 프록시 테스트하기**  

* 기본 톰캣 (localhost:8080) 을 잘 바라보나 설정 후 테스트 합니다.  
   
> /etc/nginx/sites-available/default 수정

```shell
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /usr/share/nginx/html;
        index index.html index.htm index.nginx-debian.html;

        server_name localhost;

        location / {
                proxy_pass http://localhost:8080;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
        }
}
```  
  
* 수정 후 ```sudo service nginx restart``` 로 nginx를 재시작 시키고 인스턴스 주소로 접속해 스프링부트 앱 페이지가 나오면 성공!  
     
<br>  

  
<h3>Profile 추가하기</h3>

* 스프링부트 서비스에서 운영환경에 대한 설정값은 반드시 프로젝트 내부에 위치해서는 안됩니다.  
 
* DB 연결, 비밀번호 같은 **민감한** 정보가 담겨있는데 프로젝트 내부에 있다면 누구나 확인할 수 있겠지요  
  
* 따라서 우리가 올린 프로젝트 외부에(서버에) application-prod.properties 라는 운영전용 설정값이 있고 내부에는 application-test.properties 라는 개발,테스트 전용 설정값이 있었다고 가정하고 만들어봅시다.  

> 디렉터리생성 - jar 폴더에는 빌드 결과인 jar 파일을 복사하여 위치시킬 예정입니다.

```shell
sudo mkdir /home/ubuntu/springboot/nginx
sudo mkdir /home/ubuntu/springboot/nginx/jar
``` 
  
> /home/ubuntu/springboot/nginx/jar 폴더에 application-prod.properties 추가  
  

```shell
abc=prod
```

> 프로젝트 내부에 application-test.properties 추가

```shell
abc=test
```

> abc의 값이 로컬환경에서는 테스트,개발만을 위한 정보이고 운영환경에서는 민감한 정보라고 가정합니다. ㅎㅎ  
  
<small>&nbsp;예 ) 로컬에서는 메모리기반 h2 데이터베이스 정보 / 운영환경에서는 Mysql 데이터베이스 연결 정보

<br>  

**배포마다 nginx 로 스프링부트 서비스를 다른포트로 바라보게 하는 설정 추가**    
  
* /home/ubuntu/springboot/nginx/jar 에 위치시킵니다.
  
> application-s1.properties  
  
```shell
server.port=8081

#Springboot 2.4 버전 이상 기준
#배포에 따라 다른 포트를 바라보게 하더라도 결국 운영설정파일은 하나입니다.  
#어떤 포트를 바라보는지 상관없이 같은 설정파일을 import 해주어 사용할 수 있게 합니다.
spring.config.import=application-prod.properties
```  
  
> application-s2.properties

```shell
server.prot=8082

spring.config.import=application-prod.properties
```

<br>  

  
**로컬 환경에서 default profile test 로 설정**  
  
>TestcicdApplication.java

```java
@SpringBootApplication
public class TestcicdApplication {
    public static void main(String[] args) {
        if(System.getProperty("spring.profiles.default")==null){
            System.setProperty("spring.profiles.default" , "test");
        }
        SpringApplication.run(TestcicdApplication.class, args);
    }
}
```
<br>  
  
**ProfileController 추가**  
   
> ProfileController.java  

```java
@RequiredArgsConstructor
@RestController
public class ProfileController {
    private final Environment env;

    @Value("${abc}")
    private String prodVal;

    // 로컬에서는 test가 운영에서는 prod 가 조회되어야 한다.
    @GetMapping("/abc")
    public String getProdProfile(){
        return prodVal;
    }

    // 프로젝트가 실행중일 때 default profile 을 조회하는 API
    @GetMapping("/profile")
    public String getProfile () {
        return Arrays.stream(env.getDefaultProfiles())
                .findFirst()
                .orElse("");
    }
}
```    
  
* 테스트 코드에서 default profile 이 test로 되지 않으면 ProfileController 빈 생성시 abc 관련 의존성 주입이 안되기 때문에 테스트 코드에 해당 어노테이션을 추가합니다.  
  
```java
@ActiveProfiles("test")
```

<br>  
  
<h3>배포 스크립트 만들기</h3>
 
```shell
 sudo vim /home/ubuntu/springboot/nginx/deploy.sh
```  
  
> deploy.sh  

```shell
BUILD_PATH=$(ls $BASE_PATH/build/build/libs/*.jar)
JAR_NAME=$(ls $BUILD_PATH | grep 'testcicd' | tail -n 1 | xargs -0 -n 1 basename )
echo "> build 파일명: $JAR_NAME"

echo "> build 파일 복사"
DEPLOY_PATH=/home/ubuntu/springboot/nginx/jar/
sudo cp $BUILD_PATH $DEPLOY_PATH


RESPONSE_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/profile)

echo ">$RESPONSE_CODE response code"

# 아무것도 구동중이지  않을 때 우리가 프로젝트에서 만든 api로 조회하면 당연히 오류가 난다. 이를 방지!
if [ ${RESPONSE_CODE} -ge 400 ]
then
        echo "> There is no running app lets set s2"
        CURRENT_PROFILE=s2
else
        CURRENT_PROFILE=$(curl -s http://localhost/profile)
fi


echo ">$CURRENT_PROFILE current profile"

if [ $CURRENT_PROFILE == s1 ]
then
  IDLE_PROFILE=s2
  IDLE_PORT=8082
elif [ $CURRENT_PROFILE == s2 ]
then
  IDLE_PROFILE=s1
  IDLE_PORT=8081
else
  echo "> 일치하는 Profile이 없습니다. Profile: $CURRENT_PROFILE"
  echo "> s1을 할당합니다. IDLE_PROFILE: s1"
  IDLE_PROFILE=s1
  IDLE_PORT=8081
fi

IDLE_APPLICATION=$IDLE_PROFILE-testcicd.jar

sudo ln -Tfs $DEPLOY_PATH$JAR_NAME $DEPLOY_PATH$IDLE_PROFILE-testcicd.jar


echo "> $IDLE_PROFILE 에서 구동중인 애플리케이션 pid 확인"
IDLE_PID=$(pgrep -f $IDLE_APPLICATION)

if [ -z $IDLE_PID ]
then
  echo "> 현재 구동중인 애플리케이션이 없으므로 종료하지 않습니다."
else
  echo "> kill -15 $IDLE_PID"
  sudo kill -15 $IDLE_PID
  sleep 10
fi

echo "> $IDLE_PROFILE 배포"
echo "> Change Directory to $DEPLOY_PATH "
cd $DEPLOY_PATH
echo "> $IDLE_APPLICATION Deploying "

nohup java -jar $IDLE_APPLICATION --spring.profiles.default=$IDLE_PROFILE &
```
  
 
<br>  
  
<h3>Nginx 동적 프록시 설정</h3>  
  
**배포 성공 후 프로젝트가 실행 된 후 다른 Profile을 바라보도록 변경합니다.**  

  
> /etc/nginx/conf.d/service-url.inc 에 다음을 추가해줍니다.  

```shell
set $service_url http://127.0.0.1:8081;
```
  
  
  
> etc/nginx/sites-available/default 를 다음과 같이 수정해줍니다.
    


```shell  
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /usr/share/nginx/html;
        index index.html index.htm index.nginx-debian.html;
        server_name localhost;

        # 추가된 부분 : service-url.inc 에서 설정한 변수 사용 위해 import
        include /etc/nginx/conf.d/service-url.inc;
        # --------------------------

        location / {
	    # -----변경된 부분-------
                proxy_pass $service_url;
	    # -------------------------
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
        }
}
```
  
  
> Nginx 재시작!  
  
```shell
sudo service nginx restart
```
  
<br>  

  
**switch.sh 스크립트 만들기**  
  
* 배포 시점에 실행중이지 않은 Profile로 변경되도록 하기 위함입니다. deploy.sh 하단에 위치할 예정입니다. 쉘스크립트 파일들은 실행권한도 꼭 줍시다.
  
```shell
 sudo vim /home/ubuntu/springboot/nginx/switch.sh
``` 
   
> swtich.sh 
  
```shell
#!/bin/bash
echo "> 현재 구동중인 Port 확인"
CURRENT_PROFILE=$(curl -s http://localhost/profile)

# Idle Profile 찾기: s1이 사용중이면 s2가 Idle
if [ $CURRENT_PROFILE == s1 ]
then
  IDLE_PORT=8082
elif [ $CURRENT_PROFILE == s2 ]
then
  IDLE_PORT=8081
else
  echo "> 일치하는 Profile이 없습니다. Profile: $CURRENT_PROFILE"
  echo "> 8081을 할당합니다."
  IDLE_PORT=8081
fi

echo "> 전환할 Port: $IDLE_PORT"
echo "> Port 전환"
echo "set \$service_url http://127.0.0.1:${IDLE_PORT};" |sudo tee /etc/nginx/conf.d/service-url.inc

PROXY_PORT=$(curl -s http://localhost/profile)
echo "> Nginx Current Proxy Port: $PROXY_PORT"

echo "> Nginx Reload"
sudo service nginx reload
```

<br>  
  
> 다음과 같이 두 개의 Profile 에 대해 모두 실행시켜주고  

![2](https://user-images.githubusercontent.com/76927397/128835194-9c3857f0-6899-463e-8277-fdf6ef641a9f.PNG)
  
<br>  
  
> switch.sh 를 실행시켜주면 현재 물려있지 않은 Profile 로 전환되면서 포트도 변경됩니다.

![3](https://user-images.githubusercontent.com/76927397/128835324-4b7013f7-846d-453f-8685-d84fa25c4363.PNG)
 
<br>  
  
**deploy.sh 에 switch.sh 실행 코드 추가**  
  
> /home/ubuntu/springboot/nginx/deploy.sh 맨 밑에 다음 코드를 추가합니다.  
  
```shell
echo "> Profile Switching"
sleep 10
/home/ubuntu/springboot/nginx/switch.sh
```  

<br>  

<small>해치웠나,,??</small>   

> deploy.sh 를 실행하여 최종적으로 모든 과정이 정상적으로 수행되는지 확인합니다. 아까 switch.sh 테스트로 현재 s2를 물고있으니 deploy.sh를 실행 한 후 접속을 요청했을 때 다시 s1을 물어주면 테스트 성공이겠죠?
 
![4](https://user-images.githubusercontent.com/76927397/128836276-d650db5d-4869-4b47-a362-d67687026bd9.PNG)

<br>  
  
> 다행스럽게도 해치운 모습입니다.  

![5](https://user-images.githubusercontent.com/76927397/128836843-8b66b46b-8912-4548-a571-dccf896279fd.PNG)



<h3>실제 배포를 위해 프로젝트 설정 파일 경로 수정하기</h3>

* [2편](/springbootcicd2/)에서 만든 /home/ubuntu/springboot/deploy.sh 가 아닌 현재 글에서 만든 /home/ubuntu/springboot/nginx/deploy.sh를 실행시키게끔 경로만 수정해주시면 됩니다.  

> 프로젝트 내부의 rundeploy.sh 수정  
  
```shell
#!/bin/bash
/home/ubuntu/springboot/nginx/deploy.sh > /dev/null 2> /dev/null < /dev/null &
```
    
* 경로 같은 경우는 모두가 똑같이 그대로 할 수는 없으니 상황에 맞게 적절하고 정확하게 잘 변경해야 할 것입니다.(특히 경로 헷갈려서 쉘 스크립트 만들다가 헛짓거리를 많이 해서 echo로 다 찍어보면서 함)  
  
  
* 이제 프로젝트 내에 확인해볼 수 있을 만한 적절한 변경사항을 주고(빌드 버전 같은) 커밋 푸쉬를 해봅시다.  
  
* 빌드 중 배포 중 그 어떤 순간에도 인스턴스에 띄워져 있는 서비스가 중단이 안되나 광클? 도 해봅시다.  
  
* 배포가 잘 진행되는지에 대해 로그로도 확인해봅시다. 
  

> CodeDeploy-agent 상세 로그 위치 =>  ```cd /var/log/aws/codedeploy-agent```

  
> Springboot 로그  - nohup.out 확인하기  
    

----  
   
**~~번외 feat.Codedeploy~~**  
  
* ~~글 작성 전 테스트(삽질) 시 codedeploy-agent 사용 메모리가 계속 치솟았었는데 사용한 메모리 반환이 잘 안되는 문제가 있는지 싶습니다.~~  
  
```shell
ubuntu@instance-deployinstance:/var/log/aws/codedeploy-agent$ ps -eo user,pid,ppid,rss,size,vsize,pmem,pcpu,time,cmd --sort -rss
USER         PID    PPID   RSS  SIZE    VSZ %MEM %CPU     TIME CMD
root      113261  113259 58496 102664 184352  5.8 5.3 00:00:01 codedeploy-agent: InstanceAgent::Plugins::CodeDeployPlugin::CommandPoller of master 113259
```   
  
* ~~요 녀석이었는데 저는 서비스 재시작하니 해결이 되었는데 [해당 내용을 다룬 이슈](https://github.com/aws/aws-codedeploy-agent/issues/32) 같은 것들을 참고하면서 왜때문에 그런것이고 어떻게 해결하면 좋을지, 그냥 리스타트 해도 되는 건지 시간이 나면 알아보아야겠습니다.~~ 
  

----


<br>  
  
<h3>끝~!</h3>  
  
* Nginx..... 아직 어렵지만 어쨋든 목표인 Springboot 서비스를 중단 없이 배포하기에 성공하였습니다! 
  
  
* **부족하고 긴 글 읽어주셔서 감사합니다!**

  
   
  

<br>  

  

**Reference**  

* [NGINX란?](https://azderica.github.io/00-network-nginx/)

* [Proxy](https://ko.wikipedia.org/wiki/%ED%94%84%EB%A1%9D%EC%8B%9C_%EC%84%9C%EB%B2%84)

* [Forward Proxy 와 Reverse Proxy 차이](https://happymemoryies.tistory.com/13)

* [Nginx를 활용한 무중단 배포 구축하기](https://jojoldu.tistory.com/267)  

  
<br>  

----

**이전글 - [Travis CI, CodeDeploy를 이용한 온프레미스로의 배포 자동화](/springbootcicd2/)**  

----