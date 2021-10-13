---
layout: post
title: 스프링 시큐리티 필터가 동작하기까지 1
date: 2021-10-13 12:05:41 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (5)
  
스프링 시큐리티 아키텍쳐 이해하기 - DelegatingFilterProxy , FilterChaingProxy


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  





## DelegatingFilterProxy

![prox1](https://user-images.githubusercontent.com/76927397/137062906-2a200921-92be-404e-a607-f291ebc13b27.PNG)


**ServletFilter**란 클라이언트 요청에 대해 응답하기 전 서블릿 호출 전후에서 인증,로깅  같은 작성된 기능을 수행하는 was에서 동작하는 필터이다.

스프링은 사용자 요청에 대해 응답할 때 서블릿컨테이너의 DispatcherServlet가 먼저 요청을 받아 스프링컨테이너로부터 응답에 맞는 PageController를 작동시켜 응답하게 되는데 

서블릿 필터에서는 당연히 스프링 기반 기술이나 빈을 주입하여 사용할 수 없습니다. 
 
스프링 시큐리티 필터들은 스프링 빈으로 생성된 것들로 요청이 DispatcherServlet 으로 전달되기 이전에 처리되어야 하지만 

서블릿 필터에서 이를 주입하여 사용할 수 없습니다.  

이 때 특정한 이름을 가진 스프링 빈을 찾아 그 빈에게 요청(보안처리)을 위임해주는 서블릿 필터가 **DelegatingFilterProxy**입니다.

실제로 보안처리를 하지는 않지만 SpringSecurityFilterChain 이라는 이름으로 생성된 빈을 ApplicationContext 로부터 찾아 필터를 대신 실행하게 됩니다.

<br> 

## FilterChainProxy

**SpringSecurityFilterChain의 이름으로 생성되는 필터 빈**

DelegatingFilterProxy 로부터 요청을 위임받아 실제로 보안처리를 합니다.

**스프링 시큐리티 초기화 시 생성되는 필터들을 관리하고 제어한다.** 

* 스프링 시큐리티가 기본적으로 생성하는 필터

* 설정클래스에서 api 추가시 생성되는 필터

* 사용자의 요청을 필터 순서대로 호출해서 전달. 

**마지막까지 모든 필터에 인증/인가 예외없으면 보안 통과->서블릿 자원으로 접근 허용** 


<br>

## 동작과정 - 초기화


**SecurityFilterAutoConfiguration 클래스**

SpringSecurityFilterChain 이라는 이름으로 필터 빈을 등록하고 있다. 

![del1](https://user-images.githubusercontent.com/76927397/137074628-a1108b88-d487-4f07-a48c-4321f169e801.png)


**DelegatingFilterProxy 클래스**

생성자에 targetBeanName(SpringSecurityFilterChain) , ApplicationContext 객체를 전해줘 생성하고 있다. 스프링에서도 같은이름으로 FilterChainProxy를 생성한다.

![del2](https://user-images.githubusercontent.com/76927397/137075309-48f2c7bd-22a7-455c-a93b-2605ae7f55e1.PNG)


<br>

## 동작과정 - 요청 시

**응답 전 먼저 DelegatingFilterProxy 클래스의 doFilter()가 요청을 받습니다.** 

doFilter 에서 initDelegate() 메서드로 ApplicationContext로 부터 필터를 찾아 invokeDelegate() 메서드를 통해 요청을 위임합니다.

![del4](https://user-images.githubusercontent.com/76927397/137079625-aed4cd82-f98c-410f-9e95-3a580f8f4914.PNG)


invokeDelegate() 메서드를 보면 delegate 의 doFilter(). 즉 FilterChainProxy 에게 필터 작업을 요청하고 있는 것을 알 수 있습니다.

![del5](https://user-images.githubusercontent.com/76927397/137080274-0c4a2307-2345-4095-9995-02239432a1d2.PNG)
![del6](https://user-images.githubusercontent.com/76927397/137080310-2e415f3e-d5e5-4430-87a1-4d4aad9819dd.PNG)



FilterChainProxy에서 10 여 개의 필터 작업을 모두 거치고 예외가 나지 않았다면 결국 DispatcheServlet 에서 일련의 과정을 거쳐 요청에 맞는 PageController를 구동시켜 응답을 해줄 것입니다.

![del7](https://user-images.githubusercontent.com/76927397/137082055-b969471a-b9e5-458b-9634-327a6220f019.PNG)





----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 

* [csrf](https://ko.wikipedia.org/wiki/%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%9A%94%EC%B2%AD_%EC%9C%84%EC%A1%B0)