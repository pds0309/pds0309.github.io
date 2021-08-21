---
layout: post
title: TravisCI Encrypt로 .gitignore 된 파일을 사용해보자
date: 2021-08-17 13:41:55 +0900
category2: etc
category: Study Note
tag: [github,travisCI,spring]
img: travis.png
---
<br>  

  
## TravisCI에서 .gitignore로 숨겨둔 중요한 설정파일을 사용하는 방법
<br>  

----  
  

<br>   
  
  

  
<h3>문제상황</h3>
  
* Github 저장소의 스프링부트 프로젝트에 오픈 API 키(피파온라인 , 네이버 파파고) 정보를 가지는 application-key.properties을 gitignore 처리한 상태였고 이 프로젝트를 테스트/빌드 자동화 하기 위해 TravisCI를 사용하려 하였습니다. 

* 로컬 또는 운영 환경에서는 key properties 파일이 존재하기 때문에 테스트, 실행 시 당연히 키 정보를 잘 가져오지만.. TravisCI Job Log를 보면..

```shell
git clone --depth=50 --branch=master https://github.com/pds0309/springboot-simple-fifaonline4-web.git pds0309/springboot-simple-fifaonline4-web
Cloning into 'pds0309/springboot-simple-fifaonline4-web'...
$ cd pds0309/springboot-simple-fifaonline4-web
$ git checkout -qf ....
``` 

<br>  

 
* 내 원격저장소에는 당연히 .gitigore 된 파일은 존재하지 않는데 결국 설정해둔 브랜치로부터 TravisCI가 클론해서 작업을 진행하기 때문에 테스트 과정에서 key profile 을 가져오지 못했고 FileNotFoundException으로 인해 BeanDefinitionStoreException 이 발생하였고 테스트를 실패했습니다. 
  
![111](https://user-images.githubusercontent.com/76927397/129649306-d6071106-d734-4307-a9f3-b26b6f3a37a0.PNG)
   

<br>  
  
* key profile 을 노출시킬수는 없는 상황! 어떻게 해결할 것인지 찾아보다가 Travis의 encrypt-file 을 이용해 중요 파일 또는 압축물을 암호화 하고 작업 시작 전 복호화하여 사용할 수 있다는 것을 알게되었습니다.

  
----  

<small>시작해봅시다 아주아주 왕 간단합니다.</small>  


  
<h3>로컬에서 travis로 로그인하고 작업해줍니다.</h3>
   

**travis 설치 및 로그인**  
  
```shell
gem install travis
```

```shell
C:\Users\ehd03>travis -version
1.10.0
```


```shell
#Github username 및 password로 로그인 되지 않는다면 --github-token YOURTOKEN 으로 로그인합니다.
travis login --com
```

**조건**  
  
* 암호화 할 파일의 저장소가 Travis CI에 세팅되어 있어야 한다.
* 1.7.0 이상의 Travis CI 이 설치되어야 한다.

<br>  

**파일 암호화**  

```shell
# --add -> 복호화에 필요한 key , iv 정보를 자동으로 생성해주고 .yml에도 설정에 추가됩니다.
travis encrypt-file --com [암호화할 파일 이름] --add
```

> 이렇게 나오면 성공! 암호화 전의 파일은 반드시 .gitignore 에 추가해줍니다.

```shell
$ travis encrypt-file YOURFILE.txt
encrypting YOURFILE.txt for GITHUBUSER/REPOSITORY
storing result as YOURFILE.txt.enc
storing secure env variables for decryption
```

* 암호화 된 파일은 저장소의 루트에 위치하게 합시다. 만약 내가 사용해야할 설정 파일이 특정 디렉터리에 있어야 한다면 다음과 같이 경로를 설정해서 암호화 하면 암호화된 파일은 루트위치에, 복호화는 내가 지정한 위치로 됩니다.  

> 예 ) application-key.properties 파일이 src/main/resources/ 에 위치해야 한다.  

```shell
C:\YOURREPOSITORYROOTDIRECTORY> travis encrypt-file --com src/main/resources/application-key.properties --add
```  

<br>  
 
**.travis.yml 에 다음과 같은 코드가 추가되어 있을 것입니다.**  
  
```yml
before_install:
  - openssl aes-256-cbc -K $encrypted_***_key -iv $encrypted_***_iv
    -in application-key.properties.enc -out src/main/resources/application-key.properties -d
```

<br>  

**travisCI 해당 저장소 페이지에서 Settings의 Environment Variables를 보면 key , iv 값이 알아서 저장되어있습니다**  
   
![333](https://user-images.githubusercontent.com/76927397/129651750-21ed256d-7710-4334-880c-aa4999ac4927.PNG)
  
<br>  
  
**이제 커밋푸쉬하면 중요한 설정파일을 노출시키지 않고도 TravisCI 작업에서는 사용할 수 있게 되었습니다**

<br>  

<h3>끝! + 여러 문제들</h3>  
  
**문제1. .gitignore 했는데 원격 저장소에 계속 나와요!**  

* 암호화 전의 파일은 반드시 .gitignore 에 포함되어야 하고 암호화 파일은 반드시 프로젝트에 존재해야합니다.    
* 이미 푸쉬해둔 파일은 로컬에서 gitignore 해도 원격 저장소에 계속 버전관리 되기 때문에 그대로 저장되어있습니다. 캐시를 지워주고 진행하면됩니다.

  
```shell  
 git rm --cached [지울파일]
```

<br>  
  

**문제2. 하라는대로 했는데 TravisCI 작업 Log 에서 Bad Decrypt 가 떠요!**  
  
```shell
bad decrypt
140564500219544:error:0606506D:digital envelope routines:EVP_DecryptFinal_ex:wrong final block length:evp_enc.c:518:
```
 
* 저 같은 경우는 테스트를 위해 다른 리포지토리에서 같은 이름의 파일로 테스트를 하고 본 프로젝트 리포지토리에 적용을 시켰을 때 다음 에러를 마주했었습니다.  
* 같은 github token 을 사용해서 그런가? 하고 새로 토큰을 발급받아 사용했는데도 그랬었는데 다른 리포지토리라 하더라도 같은 이름의 파일을 암호화 할 시 key,iv 이름이 같게 발급되기 때문에 충돌로 발생하는 에러로 추정됩니다. 
 
**해결방법**  
  
* encrypt 시 repository 설정하기 - 해보진 않았지만 [참고](https://travis-ci.community/t/encrypting-files-bad-decrypt/11202)
  
* encrypt 시 key,iv 파일 이름 임의로 설정하기 - 아직 제 수준에 이해가 안되는 코드였는데 링크조차 기억이... 찾는대로 올리도록 하겠습니다.  
  
* 근본적으로 같은 이름의 파일 암호화를 안하기 - 파일을 다른 이름으로 압축하고 암호화 한 뒤 복호화 후 압축해제를 하는 방법이면 될 것 같습니다.

<br>  

**문제3. 여러 파일을 암호화 해야 해요!**  
  
* 위에서 언급한 것 처럼 압축하여 암호화를 진행하시면 될 것 같습니다.

>압축 후 암호화 예시  
  
```shell
tar -cvf 압축파일이름.tar 암호화할파일1 암호화할파일2 
travis encrypt-file --com 압축파일이름.tar --add
```  
  
>TravisCI에서 사용을 위한 복호화 후 압축해제 예시
  
```shell
before_install:
    - openssl ....~~  
    - tar xvf 압축파일이름.tar
```  

* 즉 압축->암호화->복호화->압축해제 의 순서가 되겠네요.
  

<br>  

---  
 

**Reference**     
  
* [TravisCI docs - Encrypting Files](https://docs.travis-ci.com/user/encrypting-files/)
  
* [TravisCI Community - Encrypting files “bad decrypt"](https://travis-ci.community/t/encrypting-files-bad-decrypt/11202)


