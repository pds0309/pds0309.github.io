---
layout: post
title: (Jekyll Blog) GA4 적용하기 & 구글 검색엔진 등록하기
date: 2021-08-20 17:21:25 +0900
category2: blog
category: Study Note
tag: [jekyll ,ga4]
img: ga4.png

---
<br>  

## **Jekyll Github 블로그에 Google Analytics(GA4) 를 적용하고 Google Search로 검색엔진에 등록해보자**

----
  
<br> 
  
**참고 : GA에 대한 배경지식 없이 가능합니다.**  
 

Github 블로그를 만들어 공부한 내용들을 기록한 지 시간이 꽤 지났습니다.  
  
비록 아직 누추?하고 별 거 없는 글들 뿐인 부족한 블로그이지만 다양하고 많은 내용들을 더 공부할 수 있는 동기부여도 될 것 같고

 다른 사람들이 볼 수 있게끔 하여 저와 같은 초보자라면 같이 공유하고 개선하면서 발전하고 
 
고수분들의 일침, 비판, 조언 등을 통해 더 빨리 강해질 수 있지 않을까 싶어 드디어 ~~문호를~~ 개방하게 되었습니다!

 

**블로그 템플릿 - [flexible-jekyll](https://jekyllthemes.io/theme/flexible-jekyll)**

<br>  


----

<h3>블로그에 GA4 적용하기</h3>

**(1) Google Analytics 사이트에서 계정을 생성하고 <em>웹&nbsp;</em>  항목으로 블로그 주소를 입력하여 스트림을 생성합니다.**  
  
   
<br>  


**(2) GA 콘솔의 좌측 사이드바 하단의 관리(톱니바퀴) -> 데이터 스트림 -> 등록한 블로그 스트림을 클릭!**  
  
* 구 버전인 유니버설 애널리틱스를 사용할 경우(사용했을 경우) 측정 ID가 UA-000 형식이지만 새로 생성하는 우리는 GA4를 사용하기 때문에 G-000 형식입니다. 
  
![ga0](https://user-images.githubusercontent.com/76927397/130308660-f221cce7-5e64-4a60-ae27-68d47d850bce.PNG)

  
![ga0-1](https://user-images.githubusercontent.com/76927397/130308635-24ff1ba8-2b70-49e6-82b0-8e5533bfb7c1.PNG)

<br>  
  
**(3) Jekyll 블로그 _config.yml 에 측정 ID 추가하기**  

>_config.yml
  
```yml
analytics: # Google Analytics
  provider: "google-gtag"
  google:
    g_id: "G-PJT01GDQYD"
```  
  
<br>  
  
**(4) 블로그 head 태그 안에 해당 스크립트를 복사합니다**  
  

![ga0-2](https://user-images.githubusercontent.com/76927397/130308846-4d12a173-86da-4bd3-831e-bca2c95dd61d.PNG)

* 콘솔에서 데이터 스트림 -> 새로운 온페이지 태그 추가 -> 전체 사이트 태그(gtag.js) ~~ 를 클릭하면 해당 코드를 확인할 수 있습니다.  
  
* 저는 _config.yml 에 등록한 측정 ID를 사용하게끔 코드를 변경하고 _include 폴더에 analytics.html 을 만들고 블로그 기본페이지로 include 시켰습니다.  
  
> analytics.html

```html  
{% raw %}
<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id={{ site.analytics.google.g_id }}"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', '{{ site.analytics.google.g_id }}');
</script>
{% endraw %}
```

<br>  

**(5) 크롬 개발자도구 콘솔에서 ```gtag``` 를 입력해 다음과 같이 나오면 적용완료!**  
  
![ga-3](https://user-images.githubusercontent.com/76927397/130309048-346a50c7-ccff-4471-9e61-d474573f917b.PNG)
  
<br>  
  
**(6) 원격 저장소에 푸쉬하고 블로그에 접속해보고 GA 콘솔에서 실시간 보고서를 확인해봅니다**  
  
![ga](https://user-images.githubusercontent.com/76927397/130309186-c26aa5dc-2ef9-4280-b493-7af5d8a24d0e.PNG)


<br>  
  
----  
  
<h3>Google Search Console 을 이용해 구글 검색엔진에 블로그 등록하기</h3>

 
**(1) Google Search Console에서 URL 접두어에 자기 블로그 주소를 입력후 계속!**

![ga2](https://user-images.githubusercontent.com/76927397/130309420-23854d9a-c268-4e10-ba08-fd5706e552f2.PNG)


**(2) html 파일을 다운로드 한 후 블로그 루트 디렉터리에 위치시킵니다.**  
  
* 앞서 ga 를 등록하면서 <head> 태그에 추적 스크립트를 삽입했다면 Google 애널리틱스를 이용해 소유권을 확인하고 sitemap.xml을 제출하면 된다고 하나 저는 직접 번거롭게 확인시켜보겠습니다.   
  
![ga3](https://user-images.githubusercontent.com/76927397/130309481-389a2839-dd3c-4169-a42e-4623d46f4b92.PNG)

 
<br>  
  
**(3) 루트 디렉터리에 sitemap.xml 파일을 추가합니다.**  
  
> sitemap.xml은 웹사이트 내 모든 페이지의 목록을 나열한 파일로 책의 목차와 같은 역할을 합니다. 사이트맵을 제출하면 일반적인 크롤링 과정에서 쉽게 발견되지 않는 웹페이지도 문제없이 크롤링되고 색인될 수 있게 해줍니다. 

>sitemap.xml  

```xml
{% raw %}
---
---
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
    {% for post in site.posts %}
    <url>
        <loc>{{ site.url }}{{ post.url | remove: 'index.html' }}</loc>
    </url>
    {% endfor %}

    {% for page in site.pages %}
    {% if page.layout != nil %}
    {% if page.layout != 'feed' %}
    <url>
        <loc>{{ site.url }}{{ page.url | remove: 'index.html' }}</loc>
    </url>
    {% endif %}
    {% endif %}
    {% endfor %}
</urlset>
{% endraw %}
```  
    
<br>  
  
**(4) 루트 디렉터리에 robots.txt 파일을 추가합니다.**

> 내 웹사이트를 검색엔진 로봇들이 웹 크롤링 할 수 있게 하여 노출시켜주는 역할을 한다고 합니다. 설정을 통해 특정 검색엔진, 특정 컨텐츠에 대한 접근에 대해 지정할 수 있습니다.

>robots.txt 

```txt
User-agent: *
Allow: /

Sitemap: https://YOURBLOG/sitemap.xml
```

* 템플릿마다 다르겠지만 로컬에서 _site 디렉터리에 robot.txt ,sitemap.xml 이 존재한다면 지우고 진행합니다. 지우지 않으면 충돌이 발생합니다.


<br>  

**(5) 내 블로그 _config.yml 에서 ```url``` 이 내 블로그 주소로 되어있는지 확인하고 푸쉬하기!**  

> _config.yml  

```yml
url: "https://pds0309.github.io"
```  
  
<br>  
  


---  
  
**Reference**  
  
* [Robots.txt와 Sitemap.xml 알아보기](https://www.twinword.co.kr/blog/basic-technical-seo/)

* [Sitemaps for Jekyll sites](https://joelglovier.com/writing/sitemaps-for-jekyll-sites)



  
  
