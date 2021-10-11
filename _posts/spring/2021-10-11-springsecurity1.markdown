---
layout: post
title: 스프링 시큐리티 기본 및 Filter 이해하기
date: 2021-10-11 12:08:15 +0900
category2: spring
category: Study Note
tag: [springsecurity]
img: security.jpg 
---
<br>

<br>  





## Spring Boot 기반으로 개발하는 Spring Security (1)
  
스프링 시큐리티가 무엇인지 알아보고 간단하게 로그인 , 로그아웃 , 유저 정보 기억 기능을 구현하고 그 과정을 살펴보자


[인프런 강의: 스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
강의를 듣고 복습차원에서 기록하는 내용입니다.

<br>  



## 스프링 시큐리티란

* Spring 기반의 애플리케이션의 보안(인증과 권한, 인가 등)을 담당하는 스프링 하위 프레임워크.

* 인증 인가에 대한 것들을 Filter 흐름으로 처리(Dispatcher Servlet 으로 가기전에 먼저 요청을 받고 처리)  

* Credential 기반 인증방식 사용

**인증(Authentication)**

* 사용자가 본인이 맞는지 확인하는 것

**인가(Authorization)**

* 인증된 사용자가 요청한 자원에 접근할 권한이 있는지 확인하는 것

**접근주체(Principal)** 

* 자원에 접근하는 유저

**Credential(비밀번호)** 
 
* 자원에 접근하는 유저의 비밀번호


<br>  


**아키텍쳐** [ref](https://springbootdev.com/2017/08/23/spring-security-authentication-architecture/#more-54)  

![](https://chathurangat.files.wordpress.com/2017/08/blogpost-spring-security-architecture.png)

(1) 사용자의 요청 

(2) 사용자 자격 증명을 기반으로 AuthenticationToken 생성 

(3) 만들어진 토큰을 AuthenticationManager 로 전달 

(4) AuthenticationProvider(들) 목록으로 인증을 시도합니다 

(5)(6)(7) 일부 AuthenticationProvider는 사용자 이름을 기반으로 사용자 세부 정보를 검색하기 위해 UserDetailsService를 사용할 수 있습니다. 
  

(8) 인증에 성공 시 인증 객체를 반환하며 그렇지 않았을 경우 Authentication 예외를 발생시킵니다.

(9) 획득한 인증된 객체를 AuthenticationManager가 인증필터를 통해 다시 반환합니다.

(10) AuthenticationFilter는 향후 필터 사용을 위해 얻은 인증 개체를 SecurityContext에 저장합니다.


<br>  

## 스프링  시큐리티  의존성 추가하기
 
* Java8 , SpringBoot 2.5.5 , maven 으로 진행하였습니다. 

* 서버 기동 시 스프링  시큐리티  초기화 작업 및 보안설정이  이루어진다.
    * 모든 요청은  인증이 되어야 자원에  접근이 가능하다
    * 인증방식은 폼 로그인  방식  / httpbasic 로그인 방식 제공
    * 기본 로그인 페이지 제공


* 의존성 추가하기

```java
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```


* 기본으로 사용할 시 문제점
    * 계정  추가, 권한 추가 ,  DB 연동 필요하다.
    * 세부적으로  추가적 보안기능 필요하다.


<br>  


## 사용자 정의 보안 기능 구현하기


**(사용자정의)SecurityConfig -(상속)-> WebSecurityConfigurerAdapter -> HttpSecurity -> 인증, 인가 API**


* SecurityConfig 클래스를 만들고 WebSecurityConfigurerAdapter를 상속받고
configure 메소드를 오버라이딩 해 보안기능 설정을 커스텀한다.



**WebSecurityConfigurerAdapter 클래스 살펴보기**  
 
* 스프링 시큐리티 초기화 시 호출하게 된다.

> WebSecurityConfigurererAdapter 의 getHttp() 메소드 - Http Security 객체를 생성하고 기본 설정을 해줌

```java
protected final HttpSecurity getHttp() throws Exception {
    if (this.http != null) {
        return this.http;
    }
    AuthenticationEventPublisher eventPublisher = getAuthenticationEventPublisher();
    this.localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);
    AuthenticationManager authenticationManager = authenticationManager();
    this.authenticationBuilder.parentAuthenticationManager(authenticationManager);
    Map<Class<?>, Object> sharedObjects = createSharedObjects();
    this.http = new HttpSecurity(this.objectPostProcessor, this.authenticationBuilder, sharedObjects);
    if (!this.disableDefaults) {
        //default 로 설정되는 부분
        applyDefaultConfiguration(this.http);
        ClassLoader classLoader = this.context.getClassLoader();
        List<AbstractHttpConfigurer> defaultHttpConfigurers = SpringFactoriesLoader
	.loadFactories(AbstractHttpConfigurer.class, classLoader);
        for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
	this.http.apply(configurer);
        }
}
	// configure 메소드를 호출한다.
	configure(this.http);
	return this.http;
}

// default 로 다음과 같은 설정이 된다.
private void applyDefaultConfiguration(HttpSecurity http) throws Exception {
	http.csrf();
	http.addFilter(new WebAsyncManagerIntegrationFilter());
	http.exceptionHandling();
	http.headers();
	http.sessionManagement();
	http.securityContext();
	http.requestCache();
	http.anonymous();
	http.servletApi();
	http.apply(new DefaultLoginPageConfigurer<>());
	http.logout();
}
```
<br> 

**configure 메소드**

추가적 보안 설정을 하고 있다. 

기본적으로 어떤 요청이든 인증을 받도록 설정되어있고 인증방식은 formLogin , httpBasic 방식을 제공하겠다고 설정되어있다.

이 설정 때문에 스프링시큐리티 의존성 추가 후 루트 페이지에 접속해보면 로그인페이지가 자동으로 나오게 된다.

이 클래스를 상속받아 메소드를 오버라이딩 해서 재정의 해야 원하는 사용자 정의 보안 설정을 할 수 있다.

```java
protected void configure(HttpSecurity http) throws Exception {
	this.logger.debug("Using default configure(HttpSecurity). "
			+ "If subclassed this will potentially override subclass configure(HttpSecurity).");
	http.authorizeRequests((requests) -> requests.anyRequest().authenticated());
	http.formLogin();
	http.httpBasic();
}
```



<br> 

## 사용자 정의 보안 기능 구현하기 예시

* SecurityConfig 클래스 생성

```java
@Override
protected void configure(HttpSecurity http) throws Exception{
    //인가 정책
    http
        .authorizeRequests()
        .anyRequest().authenticated();
    //인증 정책
    http
        .formLogin()
        .loginPage("loginpage") // 사용자 정의 로그인 페이지
        .defaultSuccessUrl("/") // 로그인 성공 시 기본 페이지
        .failureUrl("/login") //실패 페이지
        .usernameParameter("userId") // 아이디 파라미터 - 폼 태그명
        .passwordParameter("passwd") // 패스워드 파라미터
        .loginProcessingUrl("/login_proc") // 폼 액션 url
        .successHandler(new AuthenticationSuccessHandler() { // 로그인 성공 후 핸들러
            @Override
            public void onAuthenticationSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
               System.out.println("AUT" + authentication.getName() + "님 어서오고");
                httpServletResponse.sendRedirect("/");
            }
        })
        .failureHandler(new AuthenticationFailureHandler() {
            @Override
            public void onAuthenticationFailure(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
                System.out.println("exception" + e.getMessage());
                httpServletResponse.sendRedirect("/login");
           }
        })
        .permitAll(); //인가 정책에서 어떤 요청에도 인증 없이는 접근이 불가능하게 설정했기 때문에 /loginPage 로는 모두 접속할 수 있도록 설정해준다.
}
```

<br> 

----

## Form Login

**인증 과정** 

(1) **클라이언트**의 자원 접근 시도

(2) 인증이 안되어있네? - 로그인페이지로 Redirect

(3) 사용자의 로그인 - 인증시도 - UsernamePasswordAuthenticationFilter를 통해 인증

(4) 성공시 세션 생성, 인증토큰 저장

(5) 클라이언트는 **세션**에 저장된 인증토큰으로 접근가능하고 인증이 유지된다.


![loginarc](https://user-images.githubusercontent.com/76927397/136755589-65cc24e9-a31c-45b9-81a1-dfe431672c27.PNG)




<br> 


**로그인 성공 시**  


* 로그인 페이지에서 기본유저의 id , password로 접속을 시도하면AbstractAuthenticationProcessingFilter 클래스에서 doFilter 메소드로 UsernamePasswordAuthenticationFilter를 불러와 여기서 이를 받아들이고 ProviderManager로 전달된다.


![auth1](https://user-images.githubusercontent.com/76927397/136742684-4166a19b-b128-4559-98ed-65de11746ad5.PNG)


<br> 

* ProviderManager 클래스에서 AuthenticationManager 를 거쳐 AuthenticationProvider를 거쳐 인증에 성공하면 이를 리턴한다.

![auth3](https://user-images.githubusercontent.com/76927397/136746077-99603378-700b-48e9-bafd-c5e8d45fb3df.PNG)

* 결과가 AbstractAuthenticationProcessingFilter로 전달되어 UsernamePasswordAuthenticationFilter 작업을 성공하게 됩니다.

![auth4](https://user-images.githubusercontent.com/76927397/136744416-9c6dfc35-0588-4fdf-9d67-c4931b840039.PNG)

* 정의한 successHandler 로 이동되고 스프링 시큐리티로 인해 생성된 AbstractAuthenticationProcessingFilter 다음의 필터들을 거치고 나면 성공적으로 로그인이 된 것을 확인할 수 있다.
 
![auth5](https://user-images.githubusercontent.com/76927397/136744574-384e80a6-a03e-4c71-b2c4-bfb60e903d0e.PNG)
 

![auth6](https://user-images.githubusercontent.com/76927397/136746642-56934d9d-43ea-4a67-b296-a69060abcc19.PNG)


<br> 


**로그인 실패 시**  

비밀번호를 다르게 하여 접속을 시도해보자  

* UsernamePasswordAuthenticationFilter를 통해 ProviderManager 로 온 뒤 여기서 AuthenticationProvider 를 거치고 난 뒤 예외를 발생시킵니다. 

![auth7](https://user-images.githubusercontent.com/76927397/136748237-8a2aaf39-9da5-4caf-a61c-2120fe406f61.PNG)


* AbstractAuthenticationProcessingFilter 에서 예외를 발생시켜 failHandler로 이동됩니다.

![auth8](https://user-images.githubusercontent.com/76927397/136748523-05c9bc2a-84e5-4057-8bb3-3f13cc65f285.PNG)

![auth9](https://user-images.githubusercontent.com/76927397/136748817-add6fe65-46ee-46ff-aa2b-a0ee389e1627.PNG)

<br> 
 


* 지금까지의 과정은 form login 에 대해 UsernamePasswordAuthenticationFilter 에 대해서만 확인한 것이었고 어떤 요청이든 모든 filter를 성공적으로 거쳐야 DispatcherServlet을 통해 요청에 대한 응답이 발생할 것입니다.

* FilterChainProxy 클래스를 통해서 구동하고 확인해보면 기본적으로 스프링시큐리티를 통해 10 여개의 필터를 거쳐야 됨을 확인할 수 있다. 

![auth10](https://user-images.githubusercontent.com/76927397/136750356-45bb920a-adfe-41c3-9f69-3ac174e61899.PNG)


<br>

---


## 로그아웃

**로그아웃 과정**

* 스프링시큐리티는 기본적으로는 POST 방식으로 logout 을 처리함

![logoutarc](https://user-images.githubusercontent.com/76927397/136755705-13d53ca6-09c6-49f0-971a-a83370812dc6.PNG)





> 로그아웃 설정

```java
http
    .logout()
    .logoutUrl("/logout") // 로그아웃 url
    .logoutSuccessUrl("/login") // 로그아웃 성공시 이동 페이지
    .addLogoutHandler(new LogoutHandler() {
        @Override
        public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
            HttpSession session = request.getSession();
            session.invalidate();
        }
    }) //핸들러
    .logoutSuccessHandler(new LogoutSuccessHandler() {
        @Override
        public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
            response.sendRedirect("/login");
        }
    }) // 성공 핸들러
    .deleteCookies("JSESSIONID", "remember-me"); // 로그아웃 후 쿠키 삭제
```


<br> 


## Remember Me

* 세션 종료 후 웹 브라우저 종료해도 애플리케이션이 사용자를 기억하는 기능

* Remember Me 쿠키에 대한 요청을 확인 한 후 토큰 기반 인증으로 유효성을 검사하고 검증되면 로그인 된다.



```java

@Autowired
private UserDetailsService userDetailsService;
...

http.rememberMe()
    .rememberMeParameter("remember") //기본 파라미터 이름은 remember-me
    .tokenValidiTySeconds(3600)  //default 14일
    .alwaysRemember(false) // true로 설정하면 항상 실행
    .userDetailsService(userDetailsService) // 리멤버 기능 수행 시 사용자 계정 조회처리 과정시 필요한 클래스
```

<br> 

**Remember Me 수행 과정**  
 
* SecurityContext 안에 인증을 받은 사용자는 항상 존재함

* RememberMeAtuhtenticationFilter 작동하는 조건
  * 사용자 세션이 만료(무효화) 되었을 때  
(securityContext 안에 인증 객체 없을 때 -> null 일 때 ) 요청한 클라이언트가 Remember-me 쿠키를 가지고 있어야 한다.

![rememberarc](https://user-images.githubusercontent.com/76927397/136762872-f1fc0d86-73c2-4cb4-bb56-5cefb2caf023.PNG)
 
<br> 


**Remember filter 구동 확인해보기** 

* 유저정보 기억에 체크하고 로그인 한 뒤 JSESSIONID 쿠키를 삭제합니다.

![log1](https://user-images.githubusercontent.com/76927397/136763135-fd02c181-b109-4a66-bef4-0e8fe5c4803f.PNG)


* RememberMeAuthencticationFilter 클래스에서 이미지의 위치에 SecurityContextHolder.getContext().getAuthentication() != null 필터를 포함시킨 브레이크포인트를 만들어줍니다. 
 
![remem1](https://user-images.githubusercontent.com/76927397/136767451-75004a10-6324-47a3-add3-d6b31ba5b943.PNG)


* 만약 여기에 ==null 을 필터해주고 JSESSIONID 쿠키를 삭제하지 않고 새로고침을 해보면 인증이 존재하기 때문에 해당 브레이크포인트로 넘어와 바로 인증을 성공하게 됩니다. 

* 위 이미지에서의 this.rememberMeServices.autoLogin 메서드에서 Step Info를 해보면 다음과 같이 AbstractRememberServices로 넘어와지고  
존재하는 Remember-me 쿠키를 검증하는 코드를 수행하는 것을 확인할 수 있습니다.

![remem3](https://user-images.githubusercontent.com/76927397/136768418-223e41f0-a3bf-407f-aa28-2977f0179bd8.PNG)
 

* 결과적으로는 ProviderManager를 통해 다음과 같은 결과가 리턴되고 새롭게 인증에 성공하여 확인해보면 JSESSIONID 쿠키가 다시 생성됨을 확인할 수 있습니다.

![remem2](https://user-images.githubusercontent.com/76927397/136766792-2a7040e4-4cb8-4818-b2b2-88454b2cc54f.PNG)


<br> 


  
----  

**Reference**  
 
* [스프링 시큐리티 - Spring Boot 기반으로 개발하는 Spring Security](https://www.inflearn.com/course/%EC%BD%94%EC%96%B4-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0) 
   
* [Spring Security : Authentication Architecture](https://springbootdev.com/2017/08/23/spring-security-authentication-architecture/#more-54)  
