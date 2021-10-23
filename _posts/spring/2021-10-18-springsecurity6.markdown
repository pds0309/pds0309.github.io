---
layout: post
title: 스프링 시큐리티 - 인증
date: 2021-10-18 11:15:41 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (6)
  
인증의 개념과 스프링시큐리티에서의 인증 흐름을 공부해보자


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  

---


## Authentication(인증)

**니 누구야**

~~돈 받으러 왔을 때는 뭐 그것까지 알아야 할 필요는 없지만~~

**사용자의 인증정보를 저장하는 토큰 개념**
 
* id,password를 담고 인증 검증을 위해 전달되어 사용된다.

**인증에 성공하면 최종 인증 결과(객체 , 권한정보) 를 담고 SecurityContext에 저장되어 전역적으로 사용된다.**

> Authentication auth = SecurityContextHolder.getContext().getAuthentication()


**구조** 
 
* principal - (Object) 사용자 아이디 혹은 User 객체를 저장한다

* credential - 사용자 비밀번호

* authorities - 인증된 사용자의 권한 목록

* details - 인증 부가 정보

* Authenticated - 인증여부



<br> 

## SecurityContext

**인증객체가 저장되는 보관소** 

* ThreadLocal(스레드마다 고유하게 할당되는 저장소)에 저장되어 참조 가능하도록 설계됨
* 인증 완료되면 세션에 저장되어 애플리케이션 전반에 걸쳐 전역적인 참조가 가능함.

**SecurityContextHolder** 

* SecurityContext 객체 저장 

* MODE_THREADLOCAL : 기본설정  , 스레드당 SecurityContext 객체 할당. 스레드간 인증객체 공유하지 않음

* MODE_INHERITABLETHREADLOCAL : 메인스레드 , 자식스레드 동일한 SecurityContext 유지 , 서버측에서 비동기 처리 등을 할 때 설정 필요.

* MODE_GLOBAL : static으로 SecurityContext 저장

저장전략 변경설정

```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_XXX);
```


<br> 

**사용자 인증정보는 다음과 같은 방법으로 가져올 수 있다.** 

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

```java
//HttpSession session;
SecurityContext context = 
(SecurityContext)session.getAttribute(HttpSessionSecurityRepository.SPRING_SECURITY_CONTEXT_KEY);
Authentication auth = context.getAuthentication();
```


## 인증과정요약 with SecurityContext

(1) 사용자가 로그인 요구 (username + password) 를 한다. 

(2) UsernamePasswordAuthenticationFilter 가 정보를 받아 name,password를  추출해 Authentication 이라는 객체를 생성한다.

(3) Authentication 객체에 인증에 필요한 사용자에게서 얻은 Principal(유저아이디 등) , Credential(비밀번호) 등 정보를 저장한다.

(4) AuthenticationManager를 통해서 인증객체를 이용해 인증들을 한다.

(5) 모든 인증 성공시 매니저가 Authentication 객체를 만들어 Userdetails, **권한정보**, **인증여부**를 담는다.

(6) SecurityContextHolder의 SecurityContext에 인증에 성공한 인증 객체를 담는다. 
  최종적으로 ThreadLocal에 저장된다.

(7) 세션으로 저장되고 이를 전역적으로 사용한다.


<br>



---

## SecurityContextPersistentenceFilter

**SecurityContext 객체의 생성 , 저장, 조회하는 역할을 한다.** 
 
**익명사용자일 때**

* 새로운 SecurityContext 객체를 생성하고 Holder에 저장하고 이를 AnonymousAuthenticationFilter 에서 해당 토큰객체를 만들어 저장


**인증할 때** 

* 새로운 SecurityContext 객체를 생성하고  Holder에 저장하고 UsernamePasswordAuthenticationFilter 에서 인증 성공 후 해당 토큰객체를 만들어 저장
* 인증 완료되면 세션에 SecurityContext를 저장

**인증 후 요청 시** 

* Session에서 SecurityContext를 얻어와 Holder에 저장
* Authentication 객체가 존재한다면 인증을 유지한다.


**최종적으로 하는 공통작업** 

* SecurityContextHolder.clearContext() - SecurityContext를 삭제한다.
* 매 요청마다 SecurityContext 객체를 Holder에 저장하기 때문


<br> 

## AuthenticationManager

* AuthenticationProvider 목록에서 인증처리 요건에 맞는것을 찾아 인증처리를 위임한다(직접 인증을 하지는 않음)

* 구현체인 ProviderManager 를 통해 등록된 AuthenticationProvider들을 받아와 인증처리를 한다. 


## AuthenticationProvider

* 실질적으로 인증처리를 한다.

* AuthenticationManager 가 Provider를 선택해서 인증처리를 맡기고 Provider에서 모든 인증에 성공하면 인증객체를 Manager에게 전달한다.

* 인터페이스로서 직접 커스텀해서 사용한다.

* Filter->AuthenticationManager->AuthenticationProvider-인증->인증객체->AuthenticationManager->Filter->컨텍스트저장->세션생성 의 흐름이라고 할 수 있다.



## 인증흐름

**아이디 비밀번호 기반 Form 로그인방식에서 요청에서 응답까지 전반적인 인증 과정을 간단하게 살펴봅시다** 
 


![auth100](https://user-images.githubusercontent.com/76927397/138540536-fd1194ae-4793-423e-a166-e879713ed0be.PNG)






<br>

----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
