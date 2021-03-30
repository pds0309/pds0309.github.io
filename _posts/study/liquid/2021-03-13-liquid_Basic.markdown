---
layout: post
title: Liquid 기초 문법 (1)
date: 2021-03-13 09:40:15 +0900
category: github_blog
tag: study
mytag: [liquid]

---

# Liquid 란?

* Ruby로 작성된 템플릿 언어


## Comment
* 주석

```liquid
{% raw %}  
{% comment %}
	내용
{% endcomment %}
{% endraw %}
```
<br>
## Tag
* 논리와 제어흐름을 만든다.{% raw %}{% ~ %}{% endraw %}

>입력
  
```liquid
{% raw %}  
{% if object %}
  I am {{ object.name }} .
{% endif %}
{% endraw %}
```
  
<br>
>출력

```liquid
  I am pds .
```

<br>
## Filter

* 출력내용을 바꿔준다

>입력

```liquid
{% raw %}
{{ "pds/0309/users/index" | append: ".html" }}

{% assign abc = "aa,aa,bb,cc" | split: "," %}
{{abc | uniq | join: "  " }}
{% endraw %}  
```

>출력

```liquid

{{ "pds/0309/users/index" | append: ".html" }}
{% assign abc = "aa,aa,bb,cc" | split: "," %}
{{abc | uniq | join: "  " }}
```