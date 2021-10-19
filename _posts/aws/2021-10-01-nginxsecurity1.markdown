---
layout: post
title: Nginx에서 최소한의 보안처리하기 
date: 2021-10-01 13:58:12 +0900
category2: cloud/server
category: Study Note
tag: [nginx]
img: nginx.jpg
---
<br>  

  
## Nginx를 서버로 사용할 때 할 수 있는 간단한 보안설정 몇 가지에 대해 알아보자
  



<br>  

---


  
  
<br>



## 헤더 정보 노출을 방지하자


아무 설정을 하지 않았다면 nginx 설정에서 다음 코드는 주석처리 되어있을 것입니다. 주석을 반드시 해제해줍니다.

>/etc/nginx/nginx.conf

```shell
server_tokens off;
```

사이트에 대해 헤더 정보를 요청했을 때 server_tokens off 설정을 하지 않으면 웹서버 버전이나 서버 정보가 함께 노출됩니다.

<small>예) Server: nginx/1.10.1 (Ubuntu)</small>

서버정보를 노출시킨다면 공격자가 해당 정보를 바탕으로 더 편하게 공격을 할 수 있기때문에 과도한 정보를 제공해줄 필요가 없습니다.

![1](https://user-images.githubusercontent.com/76927397/137682225-190b62b2-f1b0-4a5e-96ff-a4798d5cfd18.PNG)

### PHP 웹서버라면 PHP 세팅정보도 숨기자

nginx 설정으로 php정보를 숨길 수 없다고합니다. 

php.ini 파일을 편집하여 Exposure_php 를 off로 설정하여 헤더 정보에 php 라는 정보 자체를 노출시키지 않게끔 숨겨줍니다.

php 를 사용해본적은 없지만 nginx 접속 로그를 통해 무작위 요청들에 대한 파라미터들을 보면 php 관련된 것들이 많았던 것이 기억나는데 
php 사이트가 보안에 취약하다던가 공격기법이 많은 듯 합니다.

```shell
# sudo cat access.log.1 | grep 'php'
2XX.XX.2X8.2X - - [**/**/2021:14:25:21 +0900] "GET /phpmyadmin4.8.5/index.php HTTP/1.1" 444 0 "-" ~~ 
1XX.1X3.1XX.X5 - - [**/**/2021:14:32:48 +0900] "GET /wp-login.php HTTP/1.1" 444 0 "-" ~~
```
access 로그를 보면 허구한날 이런식으로 찍혀있습니다. 대한민국 ip가 아니면 444를 리턴하도록 처리해두었었는데 머나먼 어느 다른 나라에서 시도했을 것입니다.


<br> 

## 특정 서버에러에 대해서 Redirect 처리를 하자

사용자의 어떤 요청에 대해 응답을 하는데 만약 인증되지 않았거나(401) 관리자페이지 같은 접근금지 리소스였다고(403) 가정해보면

공격자가 무작위로 요청을 했을 때 401 또는 403을 리턴한다면? 제 웹 서버에 리소스가 있다는 것을 알려주는 것이나 마찬가지가 됩니다.

이 오류 코드 발생에 대해서 404로 반드시 redirect 시켜줍시다.

nginx 경로에 404.html을 만들던 직접 애플리케이션에서 커스텀한 에러페이지던 redirect 시켜주기만 하면 될 것 같습니다.

> 내가만든 404로 이동 예시 /etc/nginx/sites-available/default

```shell
upstream back { 

... 

}
server{ 

...
...

error_page 401 404 403 =404 /404.html;
        location /404.html {
                internal;
                proxy_pass http://back/error/404.html;
        }
...
}

```

<br>

![2](https://user-images.githubusercontent.com/76927397/137686006-d07e0cd5-b3cb-4187-8c92-618d4b4ef551.PNG)

사실 이런 화면.. 즉 기본 nginx 에러페이지 자체를 안보여주는게 올바른 방법입니다.

오류 경로라던가 버전정보 같은 민감한 서버정보에 대한 부분들을 굳이 알려줄 수도 있게 되기 때문에 

일반적인 400~500대 에러나 발생 가능한 에러 코드에 대해 반드시 커스텀한 에러 페이지로 이동하게 끔 설정하는 것이 좋을 것 같습니다.


<br>


## 민감한 리소스 경로를 잘 보호하자

만약 웹 사이트를 운영하고 관리할 때 관리자만 접속할 수 있는 모니터링 리소스라던가 관리자 페이지가 있다면 

기본적으로 모든 접근에 대해 허용을 시키지 말아야 합니다.

그런 다음 관리자의 Local LAN 주소에 대해서만 허용을 시키면  관리자만 접속을 할 수 있게 될 것입니다.

> 예시

```shell
server{
...
    location /wp-admin {
        allow 192.168.1.0/24;
        deny all;
    }
...
}
```

403-404 redirect 설정이 따로 없다면 로컬 환경이 아닌 모든 경우에 /wp-admin 에 접근 시 403 오류를 뱉게 됩니다.

또한 이를 이용해서 deny x.x.x.x 이런식으로 설정해주어 악의적 접속을 계속해서 특정 ip또는 ip 대역범위로 시도한다면 직접 차단할 수도 있겠습니다.


<br>

## Request 비율을 제한하자


악의적으로 서버에 계속해서 빠르게 많은 요청을 시도해 서버를 과부하시켜서 응답을 불가능하게 만들 수 있습니다.

nginx에서는 일정 시간당 request 수를 제한시켜 요청 비율이 초과할 때 요청을 deny 시킬 수 있습니다.

다음과 같은 설정들로 요청 비율을 제한할 수 있습니다.

> 예시

```shell

limit_req_zone $binary_remote_addr zone=one:10m rate=30r/m;

server{
...

    location / {
        limit_req zone=one burst=3 nodelay;
        limit_req_status 429;
    }
...
}

``` 

* $binary_remote_addr : 키 - 하나의 ip에 대한 요청 비율을 제한합니다.
* $request_uri : 키 - 하나의 uri 에 대한 요청비율을 제한합니다.
* zone : 지정된 키(위 예시의 경우 요청자 ip주소)에 대한 요청 상태를 저장하기 위한 공유 메모리 영역

* burst - 지정한 rate 이상의 요청에 대해 큐에 저장할 횟수 - 초과할 경우에 접속을 제한합니다.
* nodelay - burst에 지정한 만큼은 제한한 비율 이상의 요청이라 하더라도 즉각적으로 응답을 해줍니다.

* limit_req_status - 접속 제한시 리턴할 에러상태로 따로 지정하지 않으면 503을 리턴합니다.









<br>

---
  

**Reference**  

* [5 tips for better NGINX security that any admin can handle](https://www.techrepublic.com/article/5-tips-for-better-nginx-security-that-any-admin-can-handle/)

* [nginx.org - Module ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_zone)
  