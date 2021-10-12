---
layout: post
title: 스프링 시큐리티 기본 및 Filter 이해하기 2
date: 2021-10-12 09:11:15 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (2)
  
세션 제어와 예외처리 필터 , 캐시 필터에 대해 알아보자


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  


## 동시 세션 제어

**최대 세션 허용  개수 초과 시 스프링 시큐리티에서의 제어 전략** 

1. 이전 사용자 세션을 만료시킨다.
 
2. 현재 사용자(새로 접속하는)의 인증을 실패시킨다.


**설정하기**
  
```java
http.sessionManagement()
    .maximumSession(1) // 최대 허용 가능 세션 수 , -1 무제한
    .maxSessionsPreventsLogin(true) // 동시 로그인 차단함. false: 기존세션 만료시킴
    .invalidSessionUrl() //세션 유효하지 않을 경우 이동할 페이지
    .expriedUrl()  // 세션만료시 이동 페이지 invalidSessionUrl 설정시 그게 우선임
```


<br>

## 세션 고정 보호

**세션 Fixation 공격** 
 
악의적 공격자가 접속하여 발급받은 JSESSIONID 쿠키를 XSS 등 웹의 취약점을 통해 다른 사용자에게 주입하고 그 사용자가 그 상태에서 로그인을 성공하면? 
 
모니터링을 통해 쿠키를 통해 사용자가 인증에 성공하여 세션을 얻었다는 것을 공격자가 알게되면  탈취하여 그대로 사용자의 정보를 사용할 수 있게 됩니다. 

**해결법**

* 인증할 때 마다 새로운 세션을 생성시킨다 == 새로운 쿠키를 발급시킨다. => 공격자가 주입한 쿠키는 더이상 유효하지 않다.
* 스프링시큐리티에서 기본으로 설정되어있다.

**설정하기** 
 
```java
http.sessionManageMent()
    .sessionFixation().changeSessionId() // 기본값 - 인증성공하면 세션아이디만 변경시킨다.
				// none , migrateSession , newSession
```

## 세션정책

**sessionManageMent().sessionCreateionPolicy(VAL)**

* SessionCreationPolicy.Always : 항상 세션을 생성
* SessionCreationPolicy.If_Required : 필요시 생성(기본값)
* SessionCreationPolicy.Never : 생성은 하지않지만 있으면 사용
* SessionCreationPolicy.Stateless : 있어도 사용안함 - 쿠키세션방식이 아닌 다른 인증방식일 때 사용(JWT)


<br> 

## SessionManagementFilter , ConcurrentSessionFilter 동작 과정  - 최대세션 1개로 가정

<img src ="https://cdn.inflearn.com/public/files/courses/324591/dda4eeff-9e4c-454f-a446-ebb13b3d077b/arch2.jpg" />


**User1 님의 로그인 시도** 

(1) UsernamePasswordAuthenticationFilter - 인증처리 : 최초 유저가 로그인을 한다.

(2) ConcurrentSessionControlAuthStrategy 클래스 호출 - 동시세션제어 : 인증시도하는 사용자의 세션이 0개네! => 통과

(3) ChangeSessionIdAuthStrategy 클래스 호출 - 세션 고정 보호 처리 : changeSessionId()?  새롭게 세션 쿠키 발급!

(4) RegisterSessionAuthStrategy 클래스 호출 - 세션 등록!  현재 user1에 대해 연결된 세션 1개!

(5) 인증 성공!

**User2 님의 로그인 시도** 

(6) ConcurrentSessionControlAuthStrategy 클래스 호출 - 4885 너지? 아까그놈이네

(7.1) 인증실패! 아까 들어왔잖아

(7.2) 인증성공! - user1 세션 만료시킴

(8.2) ChangeSessionIdAuthStrategy 클래스 호출 - 새롭게 세션 쿠키 발급!

(9.2) RegisterSessionAuthStrategy 클래스 호출 - user2 세션 등록


**User1 님의 방문 시도**

(10.2) ConcurrentSessionFilter - 멈춰! 너 세션 만료됨. 로그아웃~

 
<br> 

--- 
 
## ExceptionTranslationFilter

* FilterSecurityInterceptor 라는 예외를 발생시키는 스프링 시큐리티 보안필터 마지막에 위치한 필터 앞에 위치하여 호출시킨다.


**AuthentictaionException** 

* 인증 예외처리

* AuthenticationEntryPoint 호출
  * 로그인 페이지 이동, 오류 코드 전달 등
  * 구현하여 원하는 처리 가능

* 인증 예외 발생 전 요청 정보를 저장
  * RequestCache : 사용자의 요청을 세션에 저장하고 캐싱 - 인증 성공하면 그 요청에 대한 응답을 함
      -> SavedRequest - 실제 사용자 요청의 request 파라미터, 헤더 값을 저장


**AccessDeniedException** 

* 인가 예외처리
	

**동작 원리 시나리오1 - 익명(인증이 안된) 사용자의 요청 시도** 
 

![excep1](https://user-images.githubusercontent.com/76927397/136888429-127a29c5-bef9-4cf1-87c2-2a764922a4b2.PNG)


<br>


**동작 원리 시나리오2 - 인증이 되었으나 권한이 없는 요청 시도**
 
 
![excep2](https://user-images.githubusercontent.com/76927397/136888303-c19d72be-2d2e-4edc-abb9-ce6de4469b7e.PNG)
 

<br> 
 

**설정하기** 
 

* 인증에 실패시 /login 페이지로, 인가에 실패 시 /denied 페이지로 이동하도록 함.
* 이 때 login 페이지 같은경우도 메서드를 오버라이드 해서 커스텀했기 때문에 직접 구현해줘야 함.(기본 스프링시큐리티 로그인 페이지가 아님) 

```java
http
    .exceptionHandling()
    // 인증 - 인터페이스 구현해서 커스텀 가능
    .authenticationEntryPoint(new AuthenticationEntryPoint() {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        response.sendRedirect("/login");
        }
    })
    // 인가 - 인터페이스 구현해서 커스텀 가능
    .accessDeniedHandler(new AccessDeniedHandler() {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        response.sendRedirect("/denied");
        }
    });
```
 
<br> 

* 인증에 성공했을 경우 세션에 저장되었던 요청 정보로 응답해주기 추가

```java
http
    .formLogin()
    // 인증 성공 시 세션에 저장된 요청 정보를 꺼내와 사용자의 그 때 요청에 대한 응답을 하는 처리를 한다.
    .successHandler(new AuthenticationSuccessHandler() {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        RequestCache requestCache = new HttpSessionRequestCache();
        SavedRequest savedRequest =  requestCache.getRequest(request,response);
        String url = savedRequest.getRedirectUrl();
        response.sendRedirect(url);
        }    
    });
```
 
<br> 
 
* /login은 인증에 실패했을 때 , /denied 는 인가에 실패했을 때의 응답이다.  
로그인 페이지의 경우 인증을 받지않아도 접근할 수 있도록 설정해주어야 한다.

```java
http.authorizeRequests() 
    // 로그인 페이지는 모두 접근 허용
    .antMatchers("/login").permitAll()
    ...
    .antMatchers("/user").hasRole("USER")
    ...
```
  


----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
