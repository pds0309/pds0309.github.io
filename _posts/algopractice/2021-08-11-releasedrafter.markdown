---
layout: post
title: Release Drafter로 깃허브 프로젝트 릴리즈 자동화하기
date: 2021-08-11 10:44:25 +0900
category2: etc
category: Study Note
tag: [github,github-actions,버전관리]
img: githubactions.png 
---
<br>  

  
## Github Actions Release Drafter 를 사용해 깃허브 프로젝트 버전 관리를 해보자  
<br>  

----  
  

<br>   
  
  

  
<h3>시작하기전</h3>
  
**Github Actions**  
  
* Github에서 공식적으로 제공하는 소프트웨어 workflow를 사용자 정의하고 자동화 해주는 CI/CD 도구이다.
  
* Github 레포지토리에서의 빌드, 테스트 , 릴리즈 , 배포 등을 이벤트를 기반으로 하여 원하는 workflow를 만들 수 있다. 


**Release**  
 
* 릴리스(release)는 소프트웨어 배포 생명 주기에서 컴퓨터 소프트웨어의 배포를 의미, 프로그램  성능과 출시 시기를 표시하는 숫자이다.
 
> 게임이나 오픈소스 라이브러리, 운영체제  등에서 본 적이 있습니다.   
예)  **클라이언트 v1.2.356 릴리즈**  
 
* 예시의 표기에 대해서 스스로 이해를 쉽게 하기 위해 **게임으로 예**를 들자면   
  
----  

> v(1).(2).(3) 일때
  
&nbsp;&nbsp;(1) 메이저 패치 - 00월드 대격변!, 리마스터   

&nbsp;&nbsp;(2) 예정된 공식적 패치 - PVP 밸런스 개선 , 새로운 생활 컨텐츠 추가!  

&nbsp;&nbsp;(3) 자잘한 마이너 패치 - 버그픽스, 그래픽 수정 등 사소한 패치  
  
----  
 
* release drafter로 깃헙 프로젝트를 위와 같은 방식으로 관리할 수 있는데 [release-drafter 문서](https://github.com/release-drafter/release-drafter)에 따르면 default로 다음과 같이 버저닝 할 수 있습니다.
  
> MAJOR.MINOR.PATCH 
  
* 상태 코드, 핫픽스에 대한 개념도 있지만 일단 이정도로만 알아보도록 합시다!   
  
>핫픽스는 게임으로 예를 또 들자면 '아 점검끝나고부터 이거 안되는데요?' 같은 것을 긴급히 수정하는 것이라고 할 수 있다.      
  
----  

<small>시작해봅시다</small>  

<small>Github [release-drafter/release-drafter](https://github.com/release-drafter/release-drafter) 문서를 참고해 최소한의 활용방법으로 아주 쉽게 다뤄보고자 합니다.
<br>  
  
<h3>yml 파일 추가</h3>
   

**리포지토리 최상단에 .github/workflows/release-drafter.yml 을 추가합니다.**  
  
>release-drafter.yml

```yml
{% raw %}
name: Release Drafter
on:
  push:
    # 해당 브랜치에 푸쉬될 때 action 이 실행된다.
    branches:
      - master
jobs:
  update_release_draft:
    runs-on: ubuntu-latest
    steps:
      - uses: release-drafter/release-drafter@v5
        # (Optional)
        # workflow 에서 여러 구성이 필요할 때 config-name 에 오버라이드 해줍니다.
        # 반드시 .github/ 에 위치해야합니다.
        with:
          config-name: release-drafter-config.yml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
{% endraw %}
``` 
 
* GITHUB_TOKEN 은 필수입니다.  
* 없다면 Developer Settings->Personal access tokens 에서 만들어줍시다. (workflow 액세스 허용에 체크!)

![33](https://user-images.githubusercontent.com/76927397/129142828-48472330-68be-4620-b5f6-53519a0994bf.PNG)

   
 
<br>   

**config-name 에 설정한 release-drafter-config.yml 을 .github/ 에 추가합니다.**  
 
* release draft 구조와 요구되는 레이블을 명시해줍니다.  
  
> release-drafter-config.yml

```yml
name-template: 'v$RESOLVED_VERSION'
tag-template: 'v$RESOLVED_VERSION'
categories:
  - title: 'Features'
    label: 'enhancement'
    label: 'feature'
  - title: 'Fixes'
    label: 'fix'
    label: 'bug'
  - title: 'Etcetera'
    label: 'documentation'
change-template: '- $TITLE @$AUTHOR (#$NUMBER)'
change-title-escapes: '\<*_&' # You can add # and @ to disable mentions, and add ` to disable code blocks.
version-resolver:
  major:
    labels:
      - 'major'
  minor:
    labels:
      - 'minor'
  patch:
    labels:
      - 'patch'
  default: patch
template: |
  ## 변경사항

  $CHANGES
``` 
 
**주요 설정 옵션**  
  
>name-template , tag-template 
  
* 각각 템플릿 이름과 태그로 MAJOR.MINOR.PATCH 중 어떤 버전을 올릴 것인가를 정의합니다.  
  
* v$NEXT_PATCH_VERSION,v$NEXT_MINOR_VERSION,v$NEXT_MAJOR_VERSION
 
* v$RESOLVED_VERSION - ```version-resolver``` 옵션에 정의한 레이블에 따라 버전을 올릴 수 있습니다. 

>categories 

* merge된 PR의 각 사항들을 릴리즈 노트에 보여줄 때 설정된 레이블에 대한 타이틀로 분류해줍니다.  
  
>version-resolver
  
* 해당 옵션을 설정하고 레이블을 설정하면 설정된 브랜치로 PR 시 붙인 레이블 기반으로 버전을 올려줍니다.

>change-template 

* PR의 내용이 릴리즈 노트에 보여질 때의 형식을 지정해줍니다.  
* $NUMBER - PR 넘버
* $TITLE - PR 제목
* $AUTHOR - PR한 사람
* $BODY - PR 내용
* $URL - PR url

  
>template**(필수옵션)**
  
* 릴리즈 노트 전체 형식을 정의합니다. 

* $CHANGES - 머지된 PR들의 리스트 , categories에 의해 분류되지 않은 PR은 맨 위에 위치합니다.
* $PREVIOUS_TAG	- 이전 버전에 대한 태그
* $CONTRIBUTORS - 프로젝트 현 릴리즈 기여자 목록
  
<br>  
  
* yml 을 모두 마스터브랜치에 추가해주고 action이 성공하고 나서 Releases 를 확인하면 github-actions 가 released 했다고 draft 릴리즈가 하나 생깁니다.  
  
* 버전 이름과 태그를 설정해준뒤 Publish 해줍시다.
 
![66](https://user-images.githubusercontent.com/76927397/129142530-1044ec5a-5b62-4df4-a0ba-dd432ba59285.PNG)
  
<br>  
  
  

----  
  
<h3>테스트하기</h3>  
  
* 이제 마스터 브랜치에 하나의 릴리즈를 위해 다른 브랜치들에서 작업되어 합쳐진 PR이 merge 되면 위의 설정에 따라 깃허브 릴리즈에 릴리즈 기록이 작성될겁니다.

* 그 전에 config.yml 에서 활용할 레이블들을 기존에 없는 것을 만들어서 진행한다면 적절한 이름과 이쁜 색깔로 레이블을 만들어줍시다.
  
**레이블 만들기**  
  
![11](https://user-images.githubusercontent.com/76927397/129135894-00bcf41b-0cc3-4a5f-bdda-cc8f562e436c.PNG)
  
<br>  
 

   
**기능 추가, 개선으로 새로운 릴리즈 출시 해보기!**  
  
* develop 브랜치를 만들고 feature1,feature2 브랜치를 분기한다. 현재 마스터 브랜치에는 텍스트파일1이  있다. (기능1)

* feature1 브랜치에서 텍스트파일2 추가(기능2 추가라고 가정)

* feature2 브랜치에서 텍스트파일1 의 오타를 수정(기능1의 개선이라고 가정)

* develop 브랜치로 (2)(3) 작업 PR 후 develop 브랜치에서 release 브랜치를 분기 해 master 로 PR한다.  
  
* 릴리즈가 제대로 작성되는지 확인한다.



<br>  
  
**기능2를 추가하고 develop 브랜치로 레이블을 설정하여 PR하기**   
  
* 이와 같이 기능1 개선도 (fix)레이블을 붙여 진행시켰습니다.

![22](https://user-images.githubusercontent.com/76927397/129137464-a3b6544a-a788-40b9-87e1-c6cd62c25c58.PNG)
  
<br>  
  
**develop 브랜치로 각 feature의 작업을 모두 merge 해주고 develop 으로부터 release 브랜치를 분기하여 마스터 브랜치로 PR후 merge!**  
  
![55](https://user-images.githubusercontent.com/76927397/129143495-f56b1db3-fbc4-4696-9a2d-cc730a279ce7.PNG)

   
<br>  

<h3>결과</h3>
  
**Actions 에서 release-drafter가 성공했는지 확인할 수 있습니다.**  

![77](https://user-images.githubusercontent.com/76927397/129143975-aaf579cc-82d0-4108-ace4-ac70dc3cb9de.PNG)
  
  
<br>  



**정의한 설정 대로 릴리즈 작성에 성공하였습니다.**  
 
* Edit 를 클릭 해 Publish Release 를 해주면 바로 릴리즈 됩니다.  
  
![88](https://user-images.githubusercontent.com/76927397/129148964-ef9eae52-9a12-424a-850f-32c448d00b83.PNG)
  
  
<br>  
  
<h3>마치며</h3>  
  
* github actions 의 release drafter 를 사용해 최소한의 프로젝트 릴리즈 관리 자동화를 해보았습니다.  
 
* 특정 레이블의 PR은 릴리즈 생성에서 배제하기, 파일명,브랜치명을 통해 자동으로 레이블 붙이기 등 다양한 기능들이 많아  
필요 시 잘 활용하면 상당히 편할 것 같다는 생각이 들었습니다.  
  
* **부족한 글 읽어주셔서 감사합니다.!**
  
  
<br>  



**Reference**     
  
* [Github Docs - Managing releases in a repository](https://docs.github.com/en/github/administering-a-repository/releasing-projects-on-github/managing-releases-in-a-repository)
  
* [GitHub actions example for automatic release drafts and changelog.md creation](https://johanneskonings.dev/github/2021/02/28/github_automatic_releases_and-changelog/)

* [What are Github Actions](https://dev.to/github/what-are-github-actions-3pml)

