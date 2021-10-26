---
layout: post
title: 스프링 시큐리티 - 인가
date: 2021-10-12 11:20:15 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (3)
  
스프링  시큐리티 인가에 대해 공부하고 사용자를 만들어서 적용해보자


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  


## 인가

**당신에게 무엇이 허가 되어있는지 자격을 증명하는 과정**

**스프링 시큐리티가 지원하는 권한 계층** 
 
웹 계층 - URL 요청에 따른 메뉴 혹은 화면 단위의 레벨 보안

서비스 계층 - 화면단위가 아닌 메소드 같은 기능 단위의 레벨 보안

도메인 계층 - 객체 단위의 레벨 보안


<br> 


## FilterSecurityInterceptor

HTTP 자원의 보안을 처리하는 필터로 권한처리를 AccessDecisionManager 에게 맡긴다.

마지막에 위치한 필터로 **인증된 사용자**에 대하여 특정 요청의 승인/거부 를 결정한다

인증객체 없이 보호자원에 접근할 경우 AuthentictaionException 발생시킨다.

인증 후 자원에 접근 가능한 권한이 존재하지 않을 경우 AccessDeniedException 발생시킨다.

즉 인증객체가 있는지, 있으면 인가 권한이 있는지에 대해 모두 체크한다


**FilterSecurityInterceptor 동작 과정** 
 
![author1](https://user-images.githubusercontent.com/76927397/138622604-bcaef398-e651-4871-8dd7-925b88757718.PNG)


(1) 사용자의 /user 접근 요청

(2) FilterSecurityInterceptor 가 **인증여부**를 먼저 체크!

(3) /user 리소스에 대해 권한이 필요한지 체크

(4) 권한이 필요 없다면 응답 / 특정 권한이 있어야 한다면 AccessDecisionManager로 전달

(5) AccessDecisionManager 와 Voter를 통해 사용자가 권한을 가지고 있는지 판단! 

(6) 해당 자원(/user) 에 접근이 가능한 사용자라면 인가 성공! 접근 불가능한 유저라면 인가 실패!

<br>

## AccessDecisionManager

인증,요청,권한 정보를 이용해 사용자의 자원접근 허용여부를 **최종결정**하는 주체

여러개의 Voter들을 가질 수 있고 Voter들로부터 접근허용,거부,보류에 해당하는 각 값들을 리턴받고 판단

최종 접근 거부 시 예외가 발생한다.

**접근결정 유형** 
 
* AffirmativeBased - 여러 Voter들 중 하나라도 접근 허가라면 허가한다.

* ConsensusBased - 다수결

* UnanimousBased - 만장일치

## AccessDecisionVoter

판단을 심사하는 것
  

**권한부여 과정에서 판단하는 자료** 

* Authentication - 인증정보  <small>User</small>

* FilterInvocation - 요청정보 <small>antMatcher("/user")</small>

* ConfigAttributes - 권한정보 <small>hasRole("USER")</small>

**결정방식**

* ACCESS_GRANTED(1)

* ACCESS_DENIED(-1)

* ACCESS_ABSTAIN(0) - voter가 해당 요청에 대해 결정을 내릴 수 없는 경우

<br>

 

## 인가 API 권한 설정방법

**선언적 방식**
* URL
* Method

**동적 방식** 
* DB 연동 프로그래밍



## URL 선언 방식 예시

* 설정 시 구체적인 경로가 먼저 오고 더 넓은 범위의 경로가 뒤에 오도록 해야한다.(순차적으로 처리한다)

```java
http
    .antMatcher("/shop/**")  // 여기 설정된 경로로만 보안기능이 작동합니다.
    .authorizeRequests()
        .antMatchers("/shop/login","/shop/users/**").permitAll() // login 해서 접근하는 요청에 대해 일치하면 users의 모든 리소스를  허용시킨다
        .antMatchers("/shop/mypage").hasRole("USER")  // mypage 에는 USER 롤이 있어야 접근할 수 있다.
        .ant ...
```

<br> 

**표현식 메서드**

* autheticated()   
* fullyAutheticated() - RememberMe 인증제외
* permitAll()
* denyAll()
* anonymous() - 익명사용자 접근 허용
* access(String) - SpEL 표현식
* hasRole(String) - 역할
* hasAuthority(String) - 권한
* hasIpAddress(String) - 주어진 IP로부터 요청만 허용

<br> 


**테스트 사용자 만들고 인가 구현하기**

```java
    // 테스트 사용자 만들기
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception { //noop = 암호화 안하고 평문사용
        auth.inMemoryAuthentication().withUser("user1").password("{noop}1111").roles("USER");
        auth.inMemoryAuthentication().withUser("sys").password("{noop}1111").roles("SYS");
        auth.inMemoryAuthentication().withUser("admin").password("{noop}1111").roles("ADMIN");
    }
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests() //모든요청에 대해 인가를 할 건데
            // 유저 페이지에는 USER롤이 있는 사용자 접근 허용시키고
           .antMatchers("/user").hasRole("USER")
           // /admin/pay 에는 ADMIN 롤이 있는 사용자  접근 허용
           .antMatchers("/admin/pay").hasRole("ADMIN")
           // 그 외 /admin 의 모든 페이지는 ADMIN이나 SYS 롤이 있으면 접근 허용
           .antMatchers("/admin/**").access("hasRole('ADMIN') or hasRole('SYS')")
           // 그 외의 모든 요청에 대해서는 인증이 필요하다.
           .anyRequest().authenticated();
        http
           .formLogin();
    }
```
 

* 이렇게 해서 테스트 해볼 경우 User로 로그인하고 admin 페이지로 접속 시도하면 403 에러가 나오며  
/admin/pay 에는 admin 사용자만 접속할 수 있고 다른 /admin 페이지에는 sys, admin 모두 접속할 수 있다. 
 
* 하지만 유저 페이지에도 admin , sys 가 접속할 수 없다. sys, admin 권한이 user보다 상위 권한이라는 설정을 하지 않았기 때문이다. 
향후 권한계층을 따로 구성해주어야 할 것이다.

* 또한 서비스 확장,변경 시 계속해서 권한설정을 수동으로 변경하거나 선언해줘야 하는 방식이라 비효율적인 방식이라고 볼 수 있다. 
    * 예) /admin/aaa 라는 기능을 추가했는데 이것도 ADMIN만 접근해야 한다면 설정에 한 줄 추가해주어야 한다. 



글의 내용은 계속해서 추가될 예정입니다.

----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
