---
layout: post
title: Spring Annotation @Component vs @Bean
date: 2021-04-22 17:11:26 +0900
category2: spring
category: Study Note
tag: [spring]
img: spring.png 
---
<br>

<br>  




##  @Component 와 @Bean 의 차이를 간단하게 공부해보았습니다.
  
<br>  

  
#### **<span style='color:red'>@Component 와 @Bean</span>**
  
**스프링 컨테이너에 빈으로 등록하도록 하는 어노테이션입니다.**  
  
##### 빈이란?
  * Spring IOC 컨테이너가 관리하는 자바객체를 말합니다. 


#### **<span style='color:red'>@Component</span>**
**개발자가 직접 작성한 클래스를 빈으로 등록시켜주는 어노테이션입니다.**  
  
> 예시  

```java  

@Component(value="myexam1")
@RequiredArgsConstructor
public class ExamClass {
    private final RestTemplate restTemplate;
    private final HttpEntity<String> requestEntity;

    public String getExam(){
        ResponseEntity<String> responseEntity = restTemplate.exchange("url", HttpMethod.GET, requestEntity, String.class);
        String response = responseEntity.getBody();
        return "{data: " + response + "}";
    }
}
```  
  
* **빈으로 등록할 클래스에 @Component를 사용하면 됩니다**
* **빈의 id를 따로 설정하지 않으면 클래스 이름으로 등록이 됩니다.(camelCase)**  
  
    
> Component.java  

```java
package org.springframework.stereotype;
...
...
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface Component {
...
	String value() default "";
}
```
  
* @Target 을 보면 클래스에만 선언 할 수 있음을 알 수 있습니다. (class, interface, enum)
  
<br>  


#### **<span style='color:red'>@Bean</span>**

**개발자가 제어 불가능한 외부라이브러리를 빈으로 만드는 어노테이션입니다.**    
  
> 예시  

```java  
// @Configuration :  컨테이너에게 1개 이상의 Bean을 등록하고 있는 클래스임을 명시해주는 것
@Configuration
@EnableAsync
public class AsyncThreadConfiguration {
    @Bean(name="myexam2")
    public Executor asyncThreadTaskExecutor() {
...
...
        return new ThreadPoolTaskExecutor();
    }
}
```  
* **객체를 반환하는 메소드를 만들고 메소드에 @Bean 을 붙여주면 됩니다.**  
* **빈의 id를 따로 설정하지 않으면 메소드 이름으로 등록이 됩니다.(camelCase)**  
    
<br>  
  

> Bean.java  

```java
package org.springframework.context.annotation;
...
...
// @Target : 어노테이션을 작성할 위치입니다.
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {

	@AliasFor("name")
	String[] value() default {};
...
...

```  
* @Target을 보면 메소드에 선언해야함을 알 수 있습니다.
  
  
<br>  


#### Reference
  
[Spring - @Bean 어노테이션과 @Component 어노테이션(DI) - 2](https://galid1.tistory.com/494?category=769011)