---
layout: post
title: 스프링 시큐리티 CSRF 필터
date: 2021-10-12 13:01:11 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (4)
  
CSRF 와 스프링 시큐리티 CSRF 필터에 대해 공부해보자


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  


## CSRF(Cross-Site Request Forgery) 

**사이트간 요청 위조를 말한다.** 
 
* 사용자가 자신의 의지와는 무관하게 공격자가 의도한 행위를 특정 웹사이트에 요청하게 하는 공격.

**XSS vs CSRF** 

* 크로스 사이트 스크립팅은 웹 상에서 사용자 입력을 받는 공간에 악의적 스크립트를 삽입해  
의도하지 않은 결과를 나타내게 하거나 쿠키 같은 개인정보를 탈취하는 클라이언트 단에서 발생하는 문제이다.

* CSRF는 사용자가 특정 웹에 인증된 상태에서 공격자의 다른 링크 등을 통해 공격코드를 실행하면 
대상 서버는 인증된 사용자로 판단하고 응답에 성공하게 되는 서버 단에서 발생하는 문제이다.

~~스팸메일같은거 열지말라는데는 다 이유가 있다.~~

<br>

## CsrfFilter

* 모든 요청에 랜덤하게 생성된 토큰을 http파라미터로 요구
* 요청 시 전달되는 토큰 값과 서버에 있는 값 비교
* 토큰 불일치 시에 AccessDeniedException 발생

<br>

**client**

* POST ,PATCH ,PUT , DELETE 방식에서 동작

* input type='hidden' name="${_csrf.parameterName}" value="${_csrf.token}" 

**스프링시큐리티**

* http.csrf() - 기본으로 활성화 되어있다.

<br> 


## CsrfFilter 동작확인하기

**csrf 필터가  활성화 되어있을 때(기본) POST 방식으로 요청을 해봅니다. 헤더에는 어떤 정보도 없습니다** 

![csrf1](https://user-images.githubusercontent.com/76927397/137058300-388c49b0-2072-4307-bad4-073f46bfea66.PNG)


**104 라인에서 csrf 토큰을 확인하고 missingToken 이라면 서버에서 생성을 해줄 겁니다.**

**121 라인에서 사용자 요청에 존재하는 토큰을 추출해와 이것을 124 라인에서 서버의 토큰과 비교하는 작업을 합니다.**

> CsrfFilter 

![csrf2](https://user-images.githubusercontent.com/76927397/137058337-02819eef-1c2a-4fd0-83a9-db101a2c093a.PNG)

 
**arc에서 아무런 헤더 정보없이 요청을 했기 때문에 actualToken = null 이고 예외를 발생시킵니다.** 
 
![csrf3](https://user-images.githubusercontent.com/76927397/137058825-5a4bfd1d-6e7e-4c64-999b-6a2b6d7fb398.PNG)
![csrf4](https://user-images.githubusercontent.com/76927397/137058967-c8a891a6-42ff-44f7-8e73-1e05628b57dc.PNG)
 
<br> 

**서버에서 생성된 헤더정보를 추가하고 요청해보면** 

![csrf5](https://user-images.githubusercontent.com/76927397/137059796-bdb743a7-4a42-4cbd-9e45-f4c8424e75b7.PNG)

**예외 없이 정상적으로 응답해줍니다.** 

![csrf6](https://user-images.githubusercontent.com/76927397/137059829-5977d37a-5bef-4175-8fcd-b0ce411a1e3e.PNG)

![csrf7](https://user-images.githubusercontent.com/76927397/137060044-d8513730-32e9-4440-975c-1d7287e61969.PNG)
  
 
**Csrf 필터를 비활성화해봅시다** 
 
```java
http
    .csrf().disable();
```

**FilterChainProxy 를 보면 CsrfFilter 자체가 생성되지 않았음을 확인할 수 있습니다.**

![csrf8](https://user-images.githubusercontent.com/76927397/137060268-b400e28f-3b53-4745-b8b8-f1648e1fa0c8.PNG)

헤더 정보 없이 POST 요청을 해보면 당연히 정상적인 응답을 해줍니다.

Thymeleaf 같은 뷰 템플릿이나 스프링 form 태그는 기본적으로 POST 요청시 알아서 Csrf 토큰을 생성해줍니다.

<br> 

----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 

* [csrf](https://ko.wikipedia.org/wiki/%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%9A%94%EC%B2%AD_%EC%9C%84%EC%A1%B0)