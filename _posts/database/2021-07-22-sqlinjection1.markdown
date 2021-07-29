---
layout: post
title: SQL Injection 겉핥기 feat. JPA
date: 2021-07-22 09:25:45 +0900
category2: database
category: Study Note
tag: [sql,jpa,spring]
img: databaseimg.png 
---
<br>

<br>  




##  SQL Injection
  
<br>  




  
<h3 style='color:red'>SQL 인젝션이란?? </h3> 
 
SQL 인젝션은 백앤드 데이터베이스를 조작해 금지된 정보에 접근하는 공격 유형입니다.  
  
민감한 정보에 접근하여 확인하거나 이를 조작해 피해를 끼칠 수 있는 **매우 심각한** 공격입니다.  
  
<br>  


<h3 style='color:red'>SQL 인젝션 예시 </h3> 

**<span style='color:blue'>Retrieving hidden data (숨겨진 데이터 검색)</span>**

&nbsp;SQL문 수정으로 숨겨진 데이터를 조회합니다.  
  
**<span style='color:blue'>Subverting application logic(애플리케이션 로직 파괴)</span>**
  
&nbsp;SQL문 수정으로 애플리케이션 실행 로직을 방해합니다.  
  
**<span style='color:blue'>Blind SQL injection</span>**
  
&nbsp;오류 메시지가 보이지 않을 때 쿼리의 True/False 동작으로 데이터베이스 구조를 파악하는 방법입니다.    

**<span style='color:blue'>Union SQL injection</span>**
  
&nbsp;기존 쿼리에 악성쿼리를 합집합하여 정보를 획득하는 방법입니다.  
  
**<span style='color:blue'>Examining the database</span>**
  
&nbsp;데이터베이스 버전, 구조 같은 정보를 얻을 수 있습니다.  
  
  
<br>  
  

  
<h3 style='color:red'>블라인드 인젝션을 체험해봅시다. </h3>

**연습 사이트**  
[http://testphp.vulnweb.com/](http://testphp.vulnweb.com/)

아무사이트에서나 시도하는 것 == 놀이라 보기엔 이건 범죄
  
연습용 사이트와 자기가 만든 사이트에서 시도해보도록 하자.  

<br>  


> http://testphp.vulnweb.com/listproducts.php?cat=1
  
**주소를 보면 listproducts 페이지에서 카테고리 1번째를 보여주겠다는 것을 알 수 있습니다.**
  
product 에 대한 테이블에서 카테고리가 1인 부분을 조회해라! 라는 쿼리를 가지고 있겠죠  
  
> 
  
![07221](https://user-images.githubusercontent.com/76927397/126589347-c6507799-af5c-4f63-8f0b-3c7a890cd102.PNG)

<br>  
  
**<span style='color:red'>SQL 인젝션이 가능한지 판별해봅시다.   
사이트의 쿼리가 where cat = 1 로 끝난다고 가정하고 조건을 넣어봅시다.</span>**
  
> and 1=1 
  
![07222](https://user-images.githubusercontent.com/76927397/126589574-4a06865c-2b4b-47e6-8fb3-c07f1958b814.PNG)
  
**조회에 성공합니다(TRUE)**  
  
1=1 은 항상 참이기 때문에  ```WHERE CAT = 1 AND 1=1;``` 로 조회했다는 사실을 알 수 있습니다.
  
<br>  
  
> and 1=0  
  
![07223](https://user-images.githubusercontent.com/76927397/126589633-72975c05-8767-4543-9254-247a80e2c3e4.PNG)
  
**조회에 실패합니다(FALSE)**

SQL 공격에 취약한 사이트임을 확인했습니다. 게시글 정렬 순서가 마음에 안드네요. 제가 바꿔보겠습니다.  
 
<br>  

> order by 2
  
![07224](https://user-images.githubusercontent.com/76927397/126590114-49dd99fb-dd90-4e6f-a87c-f6d42dc04456.PNG)
  
해당 페이지의 2번째 컬럼이 무엇인지는 모르겠지만 게시글 정렬순서가 바뀐것을 알 수 있습니다.  

order by 11 까지 정상적으로 출력되고 12에서 출력되지 않는 것으로 해당 쿼리에서 11개의 컬럼을 사용한다는 것을 알 수 있었습니다.  
  
<br>  
  

**<span style='color:red'> 이제 union SQL 인젝션을 시도해보겠습니다.</span>**  
  
> union select 1,1,1,1,1,1,1,1,1,1,1,version()  
  
![07225](https://user-images.githubusercontent.com/76927397/126590447-f241ee6d-db55-462b-83af-3367c074ba0b.PNG)
  

union sql 인젝션 결과로 해당 사이트에서 mysql 을 사용한다는 정보와 더불어 mysql 버전까지 알 수 있었습니다.   
 
MySQL 을 사용한다는 사실을 알았으니 information_schema.table 로 현재 데이터베이스의 테이블 목록들도 확인해보았습니다.  
  
![07226](https://user-images.githubusercontent.com/76927397/126591301-97af47bf-3184-419c-bed1-931c842ea179.PNG)
  
이런 방법으로 원하는 테이블의 컬럼들도 확인할 수 있게 되고, 사용자들에 대한 테이블일 가능성이 높은 users 테이블의 정보를 얻게된다면..?  
  
공격자가 저 처럼 글 정렬순서나 바꾸고 앉아있으면 다행이겠지만 중요정보를 탈취해 악용하게 된다면 정말 큰 문제가 될 것 같습니다.  
  
<br>  
 
<br>  


  
  
<h3 style='color:red'>내 사이트에서 블라인드 SQL 인젝션 해보기 </h3>
  
**아직 만들어가고 있는 피파온라인 open-api 조회 사이트에서 SQL 인젝션을 해보겠습니다.  
회원 및 로그인 기능은 없고 만들 필요를 느끼지 못하고 있어 간단하게 선수 조회 페이지에서 해볼까합니다.**  
  
**<span style='color:blue'>목표</span>**  
  
&nbsp;SQL 인젝션으로 선수검색 페이지에서 유저 테이블의 유저를 모두 조회해보자
  
<br>  
  
**선수 조회 쿼리입니다. 실제 사용하는 쿼리는 조인,정렬 등으로 복잡해 원활한 공격? 을 위해 간단하게 변경해보았습니다.**   

```java
    @Query(nativeQuery = true,value = "select * from PLAYER  where PLAYER_NAME = :name")
    Page<Player> findByPlayerName(String name, Pageable pageable);
```  
  
**정상적인 차범근 선수 조회**  
  
![07227](https://user-images.githubusercontent.com/76927397/126593600-6d17d818-5d14-41e0-8fab-7980db7c1f7b.PNG)
  
  
<br>  

**자 이제 SQL 인젝션이 가능한지 판별해 봅시..**  
  
![07228](https://user-images.githubusercontent.com/76927397/126594291-8e4a507c-8d9d-48d6-81ba-e89cc2ca6fac.PNG)

  
<br>  

**왜 안될까요? 쿼리 로그를 확인해봅시다.**  
  
* 스프링 부트에서 다음과 같은 설정으로 sql 쿼리 로그를 확인할 수 있습니다.
```properties
logging.level.org.hibernate.SQL=debug
logging.level.org.hibernate.type.descriptor.sql=trace
```  
  
![07229](https://user-images.githubusercontent.com/76927397/126594788-5600a986-5e0b-4c74-8dc4-936254210ef7.png)
  
**SELECT * FROM PLAYER WHERE PLAYER_NAME = "차범근" AND 1=1; 이라는 결과를 원했지만** 

**PLAYER_NAME = ? 에서 ?에 해당되는 입력값이 "차범근 AND 1=1" 로 들어가 있는 것을 알 수 있습니다.** 
  
즉 **'차범근 and 1=1'** 이라는 선수를 조회한 꼴이 된 것입니다.
  
<br>  
  
  
<h3 style='color:red'>JPA 와 SQL Injection</h3>  
  
  
**<span style='color:red'>JPA 에서는 파라미터 바인딩으로 SQL 인젝션에 대한 걱정이 크게 없다</span>**  
       
&nbsp;위의 자바코드에서 사용한 ```:name``` 은 이름 기반 파라미터 바인딩이고 JPA에서 제공하는 쿼리메소드를 사용할 때도 적용됩니다.  
   
예를 들면 ```findById()``` 는 ```where x.id = ?1``` 와 같은 형태가 됩니다.
  

&nbsp;또한 [adam-bien](https://www.adam-bien.com/roller/abien/entry/preventing_injection_in_jpa_query) 이 분의 말에 따르면 SQL,QL 인젝션은 JPA 네임드 쿼리를 통해 효과적으로 방지할 수 있고 
  
거의 모든 경우의 90%를 처리할 수 있다고 합니다.  
  
<br>  
    

**<span style='color:red'>직접 쿼리를 스트링으로 EntityManager로 주입하는 것은 위험할 수 있다.</span>**  
  
**직접 확인해보기 위해 레포지토리를 만들고 name 을 입력받아 문자열 쿼리를 통해 EntityManager 로 전달하게 하였습니다.**  
 
   
```java
@RequiredArgsConstructor
@Repository
public class MyRepository {
    private final EntityManager em;
    public List<Player> findPlayer(String name){
        String sql = "SELECT * FROM PLAYER WHERE PLAYER_NAME='"+name+"';";
        List<Player> playerList = em.createNativeQuery(sql)
                .getResultList();
        return playerList;
    }
}
```  
<br>  
  
**JSON 데이터로 조회 결과를 얻을 수 있게 컨트롤러를 만들어주었습니다.**  
  
```java
    @GetMapping("/playersearch2")
    @ResponseBody
    public List<Player> getPlayerList2(@RequestParam String pname){
        return playerService.getPlayerList(pname);
    }
```  
  
<br>  
  
  
**<span style='color:red'>pname=차범근 (정상조회 결과)</span>**  

![072210](https://user-images.githubusercontent.com/76927397/126612359-41bf4d8d-62f7-40de-88e6-84b9a26eb062.PNG)


  
<br>  

**<span style='color:red'>{% raw %} pname=차범근' and 1=1-\- {% endraw %}</span>**  
&nbsp;아까와는 다르게 결과가 나옵니다.  
1=1 조건을 넣으면 결과가 나오고 1=0 조건을 넣으면 결과가 나오지 않습니다.  
쿼리 로그를 보면 입력한 값 그대로 쿼리에 추가되어 조회된 것을 알 수 있습니다.  
  
![072211](https://user-images.githubusercontent.com/76927397/126612291-6c66fe88-ae9b-4ea0-9620-0ab88d1000ee.PNG)


```java
{% raw %}
2021-07-19 16:52:48.83 DEBUG 4212 --- [nio-8080-exec-8] org.hibernate.SQL                        : 
    select
        * 
    from
        PLAYER  
    where
        PLAYER_NAME='차범근' 
        and 1=1--';
{% endraw %}
```
  
<br>  
  
**취약점을 이용해 테이블 목록을 모두 탈취하였다고 가정하고 Users 테이블의 모든 선수를 조회해보겠습니다.**  
  
**<span style='color:red'>pname=차범근` union select 1, user_name , user_level from users</span>**  
  
![072212](https://user-images.githubusercontent.com/76927397/126612192-e9cb95dd-502f-4015-887b-b8cb7b5e37ba.PNG)


  
```java
{% raw %}

  2021-07-19 17:10:48.986 DEBUG 3284 --- [nio-8080-exec-2] org.hibernate.SQL                        : 
    select
        * 
    from
        PLAYER  
    where
        PLAYER_NAME='차범근' 
    union
    select
        1,
        user_name,
        user_level 
    from
        users--';
{% endraw %}
```
**이런 취약점을 잘 활용하면 공격자 마음대로 테이블을 다 날려버릴 수도 있겠군요.**  
  
<br>  

----  
&nbsp;**SQL Injection**의 개념과 종류에 대해 간단하게 알아보고 실제로 해보고,   
스프링의 JPA에서는 어떨까 알아보기 위해 자신의 웹 사이트 공격도 마다하지 않는  
**SQL Injection**의 겉을 핥아보는 시간을 가져보았습니다.  

가볍게 알아보고 공부하려고 하였으나 알아보면 알아볼 수록 깊은 내용이 많고 어렵게 느껴졌습니다.  
현재 저의 수준으로는 JPA 쿼리메소드 커스터마이징이 어려워 쿼리를 직접 작성해야 할 때 파라미터 바인딩을 잘 하자 라는 결론을 내려볼 수 있겠습니다.  
  
  
길고 부족한 글 읽어주셔서 감사합니다.  

----

### References

* [What is SQL Injection? How to Prevent SQL Injection](https://dzone.com/articles/what-is-sql-injection-how-to-prevent-sql-injection)
* [adam-bien.com - Preventing Injection in JPA Query language](https://www.adam-bien.com/roller/abien/entry/preventing_injection_in_jpa_query)
* [Stackoverflow - Avoid SQL Injection in Spring JPA](https://stackoverflow.com/questions/63997976/avoid-sql-injection-in-spring-jpa)
* [Stackoverflow - Are SQL injection attacks possible in JPA?](https://stackoverflow.com/questions/3441193/are-sql-injection-attacks-possible-in-jpa)

