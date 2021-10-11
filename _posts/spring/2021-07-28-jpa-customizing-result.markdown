---
layout: post
title: JPA 특정 컬럼 조회 feat.ConverterNot FoundException 
date: 2021-07-28 10:08:15 +0900
category2: spring
category: Study Note
tag: [spring,jpa]
img: spring.png 
---
<br>

<br>  




## [Springboot] JPA를 활용해 원하는 컬럼만 조회 해보자!
  
<br>  



* JPA로 요청에 알맞는 쿼리를 만들 때 정의한 엔티티 테이블의 일부 컬럼만 반환한다던가 집계함수, 조인 등을 사용해 다양한 결과를 출력해야 할 일이 있습니다.  
  

* 간단한 서비스를 만들면서 실제로 겪고 공부하여 해결한 것을 바탕으로 쉬운 예시를 통해 기록해보고자 합니다. 
  
  
<br>  

  
<h3>요청 예시) 랭크 구간 별 유저 수를 조회하여 랭크 이름과 해당 랭크의 유저 수 현황을 뷰에 그래프로 보여주고자 한다.</h3>
  

>Users 테이블  

|-----------|---------------|------------|---------|
|USER_NAME|USER_ID|USER_LEVEL|HIGH_RANK|
|(PK)VARCHAR|VARCHAR|INTEGER|(FK)INTEGER|
 

>Rank 테이블 

|-----------|---------------|
|DIVISION_ID|DIVISION_NAME|
|(PK)INTEGER|VARCHAR|

  
**두 테이블은 1:N 관계이고 요구사항을 처리하는 쿼리는 다음과 같습니다.**  
  
```sql  
SELECT R.DIVISION_NAME AS DIVISION , COUNT(U.USER_ID) AS CNT
FROM USERS U
JOIN RANK R
ON U.HIGH_RANK = R.DIVISION_ID
GROUP BY R.DIVISION_NAME
ORDER BY 1 DESC;
```  

<br>  

	 
* 결과 모양이 달라 기존에 정의된 Users 인스턴스에는 결과를 로드 할 수 없습니다.    

* Object[] 타입으로 반환하게 하면 결과에 액세스 할 수는 있지만 지저분하고 오류 발생이 쉬운 코드가 됩니다. - [ref](https://www.baeldung.com/jpa-queries-custom-result-with-aggregation-functions)
  
> While we can still access the results in the general-purpose Object[] returned in the list, doing so will result in messy, error-prone code.

<br>    


<h3>에러 상황</h3>  
----  
  

**적절한 Alias도 주었겠다. WrapperClass를 따로 정의 해 결과를 얻어보기로 하였습니다.**  
  
>UserCntDivisionVo
  
```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class UserCntDivisionVo {
    private String division;
    private Long cnt;
}
```


>UserRepository
  
```java
public interface UserRepository extends JpaRepository<User, String> {
    @Query(value="select r.divisionName as division, count(u.userId) as cnt " +
            "from Users u join Rank r on u.rank.divisionId = r.divisionId " +
            "group by r.divisionName " +
            "order by 1 desc ")
    List<UserCntDivisionVo> findUserNumByDivisionError();
}
```
 
**조회 결과 ConverterNotFoundException 예외가 발생합니다.**  
 
* JPA 에서 쿼리 결과에 대해 반환한 타입이 제가 정의한 클래스 타입으로 변환되지 못한 것입니다.  
  
```java
org.springframework.core.convert.ConverterNotFoundException: 
 No converter found capable of converting from type [org.springframework.data.jpa.repository.query.AbstractJpaQuery$TupleConverter$TupleBackedMap] to type [com.xyz.ff.UserCntDivisionVo]
```
  

<br>  
  


<h3>해결하기</h3>  
----  
  
**1. Custom Class With JPQL**  

* JPQL(Java Persistence Query Language) 란?  
 
>JPA 의 일부로 정의된 플랫폼 독립적 객체 지향 쿼리이다.  
DB의 테이블이 아닌 JPA 엔티티에 대해 동작한다.(쿼리에 테이블 컬럼이름이 아니라 엔티티에서 표현되는 컬럼 이름을 사용해야함.)
  
* 객체 지향 방식으로 결과를 사용자가 정의할 수 있습니다.   
SELECT 절의 컬럼을 POJO에 바인딩하는데 지정된 클래스에는 프로젝션 된 속성과 정확히 일치하는 생성자가 있어야 합니다. 
  

>UserRepository
  
```java
public interface UserRepository extends JpaRepository<User, String> {
    @Query(value = "SELECT new com.xyz.ff.UserCntDivisionVo(r.divisionName,count(u.userId)) " +
            "FROM Users u " +
            "Join Rank r ON u.rank.divisionId = r.divisionId " +
            "GROUP BY r.divisionName " +
            "ORDER BY 1 DESC")
    List<UserCntDivisionVo> findUserNumByDivisionJpql();
}
``` 

<br>  

**2. Spring JPA Projection**
  
* 인터페이스 기반 프로젝션을 통해 정의한 인터페이스를 기준으로 Repository 반환 값에 매핑해줍니다.  
     

>UserCntDivisionInterface 
  
* 프로젝션 될 컬럼이름과 일치하게 인터페이스에 getter 메소드를 추가해야 합니다.
 
```java
public interface UserCntDivisionInterface {
    String getDivision();
    Long getCnt();
}
```  
  
>UserRepository

```java
public interface UserRepository extends JpaRepository<User, String> {
    @Query(value="select r.divisionName as division, count(u.userId) as cnt " +
            "from Users u join Rank r on u.rank.divisionId = r.divisionId " +
            "group by r.divisionName " +
            "order by 1 desc ")
    List<UserCntDivisionInterface> findUserNumByDivisionInterface();
}
```  

<br>  

  
<h3>결과 테스트</h3>  
----  
  
> TestCustomProjection

```java
@ActiveProfiles("test")
@SpringBootTest
public class TestCustomProjection {

    static List<UserCntDivisionInterface> listInterface;
    static List<UserCntDivisionVo> listJpql;

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("[에러 상황 - 에러 시 통과]")
    void errorCustomTest() {
        Exception e = assertThrows(Exception.class, () ->
                userRepository.findUserNumByDivisionError());
        System.out.println("에러메시지 : " + e.getMessage());
    }

    @Test
    @DisplayName("[JPQL]")
    void jpqlCustomTest() {
        assertDoesNotThrow(() ->
                userRepository.findUserNumByDivisionJpql());
        System.out.println("JPQL 정상");
        listJpql = userRepository.findUserNumByDivisionJpql();
        assertNotNull(listJpql);
    }

    @Test
    @DisplayName("[Spring JPA Projection]")
    void interfaceCustomTest() {
        assertDoesNotThrow(() ->
                userRepository.findUserNumByDivisionInterface());
        System.out.println("JPA 프로젝션 정상");
        listInterface = userRepository.findUserNumByDivisionInterface();
        assertNotNull(listInterface);
    }

    @AfterAll
    public static void testBoth() {
        assertEquals(listJpql.size(), listInterface.size());
        for (int i = 0; i < listJpql.size(); i++) {
            assertEquals(listJpql.get(i).getCnt(), listInterface.get(i).getCnt());
            System.out.println(listJpql.get(i).getDivision()+" " + listJpql.get(i).getCnt());
        }
    }
}
```
<br>  
  
**결과**  
  
![07291](https://user-images.githubusercontent.com/76927397/127434323-8d4f3537-33ec-4f90-91e1-0772681c2bea.PNG)
  
  
-----
 
<br>  
  
**Reference**  
 
* [Customizing the Result of JPA Queries with Aggregation Functions](https://www.baeldung.com/jpa-queries-custom-result-with-aggregation-functions)  
  
  
* [Spring Data JPA Projections](https://www.baeldung.com/spring-data-jpa-projections)
  
