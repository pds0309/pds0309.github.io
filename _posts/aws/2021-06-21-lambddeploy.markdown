---
layout: post
title: Github Actions를 이용해 람다 서비스 배포 자동화를 해보자
date: 2021-06-21 19:31:05 +0900
category2: cloud
category: Study Note
tag: [aws,lambda,github,CI/CD]
img: githubactions.png 
---
<br>  

  
## Github Actions 를 이용해 코로나 정보 문자 메시지 전송 서비스를 관리해보자
  
<br>  

----  
  
> 해당 게시글의 람다 서비스를 바탕으로 작성하였습니다.   
읽지 않아도 현재 포스트의 실습은 가능합니다.

**1편 - [AWS Lambda로 만드는 초간단 코로나 정보 문자메시지 전송 서비스(1)](/lambdasms1/)** 
  

**2편 - [AWS Lambda로 만드는 초간단 코로나 정보 문자메시지 전송 서비스(2)](/lambdasms2/)**  

----
  

<br>  

### <span style='color:red'>Github Actions 란?</span>
  
* 깃허브에서 제공하는 CI/CD 툴이라고 합니다. 
* 도커 강의를 들으면서 TravisCI , 스프링 공부를 하면서 Jenkins 등에 대해서 들어본 적이 있었는데  
  이런 서비스들의 기능을 깃허브에서도 관리할 수 있다는 그런 느낌 같습니다.  

  
<br>  

### <span style='color:red'>CI / CD 란?</span>  
 
**CI(Continuous Integration)**  
 * 애플리케이션에 대한 코드 변경이 빌드, 테스트 되어 저장소에 통합되는 것을 말합니다.   
  
**CD(Continuous Deployment or Delivery)**
  * 중단없는 배포 , 사용가능한 환경으로 중단없이 자동으로 릴리즈 되는 것을 말합니다.  


 
  
  
<br>  
  
### <span style='color:red'>그래서</span>  
* 깃허브 레포지토리에서 람다 함수 코드를 관리하고 actions 를 사용하면 내 로컬환경에서 코드를 수정하고 깃허브에 커밋,푸쉬하면  
   실제 내 클라우드에서의 람다함수 코드도 알아서 변경이 반영되고 중단 없이 서비스 된다라는 것입니다.
  

<br>  



### <span style='color:red'>해봅시다</span>  
  
**(0) 준비물 체크리스트!**  
  
* AWS 계정이 있는가?  
* AWS 람다 함수를 생성 할 수 있는가?
* 깃허브 계정이 있는가?  
* 내 로컬에 node.js 설치가 되어있는가?  
  
<br>  

 
**배포 자동화 해야 할 람다 함수들입니다.**  
  
  
![3333](https://user-images.githubusercontent.com/76927397/124054901-e2fd5900-da5d-11eb-9301-56c1dacf3f79.PNG)
  
<br>  

**npm 버전**  


```linux  
C:\>npm -v
6.14.11
```  
  
**Github Actions 는 브랜치 별로 Action 을 설정할 수 있습니다. 레포지토리에 다음과 같이 브랜치를 나누고 진행하였고  
 현재 글에서는 info 브랜치로 CoronaInfo를 배포 자동화 하는 것으로 진행하였습니다.**  
 * CoronaSmsService - main  : 해당 게시글 작성 이전에 완료하였습니다.
 * CoronaInfo - info  
 * CoronaDynamo - dynamo  
  
  
<br>  
  

**(1) 깃허브 레포지토리를 만들고 로컬에 클론한 뒤 info 브랜치를 생성해주고 해당 브랜치로 이동해줍니다**  
  
```linux  
ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo
$ git clone https://github.com/pds0309/aws_lambda_simple_sms_service.git
...
...  

ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo
$ cd aws_lambda_simple_sms_service/
  
ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo/aws_lambda_simple_sms_service (main)
$ git remote -v
origin  https://github.com/pds0309/aws_lambda_simple_sms_service.git (fetch)
origin  https://github.com/pds0309/aws_lambda_simple_sms_service.git (push)

ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo/aws_lambda_simple_sms_service (main)
$ git branch
* main

ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo/aws_lambda_simple_sms_service (main)
$ git branch info

ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo/aws_lambda_simple_sms_service (main)
$ git checkout info
Switched to branch 'info'  
  
ehd03@DESKTOP-REGVU7O MINGW64 /c/cinfo/aws_lambda_simple_sms_service (info)
$ 
```   
 
<br>  

  
**(2) package.json 생성 및 의존 패키지 설치** 
  
```linux  
$ npm init -y  
$ npm install moment  
$ npm install xml-js  
```  
* 액션 워크플로우에서 npm ci 를 사용할 예정이기 때문에 package.json , package-lock.json 은 로컬에서 생성되어 있어야 합니다.   
 
* 물론 워크플로우 파일 자체에서 npm init -y 나 패키지 의존성 설치를 할 수 있지만 코드 수정 배포 자동화를 할 때마다 설치하는 것은 비효율적이라고 생각합니다.    

* 의존성 추가 및 제거가 필요하다면 로컬에서 해주고(즉 package.json, package-lock.json 또는 npm-shrinkwrap.json 는 로컬에서 생성한다!) npm ci로 검사하는게 올바른 방법이라 생각합니다.    

* **ref: [npm install 과 npm ci 의 차이점은 무엇입니까?](https://pythonq.com/so/npm/15391)** 
  
  
<br>  

**(3) index.js 생성**  
  
* 테스트를 위해 일단 다른 코드를 넣어봅시다.  
  
```javascript
exports.handler = async (event) => {
    const response = {
        statusCode: 200,
        body: JSON.stringify('hihihi'),
    };
    return response.body;
};   
```  
  
<br>  

**(4) 원격저장소에 푸쉬해줍니다.**  
  
```linux  
$ git add .  
$ git commit -m 'commit coronainfo'  
$ git push origin info  
```  
  
* 잘 되었군요  
  
![4444](https://user-images.githubusercontent.com/76927397/124058293-10e59c00-da64-11eb-90a2-a6a8925a9de9.PNG)

 
 
<br>  
**(6) 깃허브 레포지토리의 Settings -> Secrets 에 <span style='color:blue'>AWSAccessKeyID</span> 와 <span style='color:blue'>AWSSecretKey</span>를 등록합니다.**  
  
![66666](https://user-images.githubusercontent.com/76927397/124059269-c82ee280-da65-11eb-8f7c-53c3246292b1.PNG)
  
* 이게 무슨 소리고! 하신다면 **AWS IAM 콘솔**로 이동 -> 내 액세스 키에서 새 액세스 키를 발급받아 사용하시면 됩니다.   
 
* 비밀 키는 생성 시에만 확인 할 수 있기 때문에 발급 받고 다운로드 후 잘 보관해두시면 됩니다.
  
<br>  
  
  

**(7) 깃허브 레포지토리의 Actions 탭 클릭 후 <span style='color:blue'>Set up a workflow yourself</span> 클릭**  
  
* 하단에 Node.js 앱 Azure 에 배포하기 , Amazon ECS 에 배포하기 등 다양한 예시가 있지만 직접 만들어 봅시다.

![55555](https://user-images.githubusercontent.com/76927397/124058535-894c5d00-da64-11eb-9d92-8c80b98b7cf6.PNG)
  

**(8) yml 파일 작성 후 커밋해줍니다.**  
  
  
<span style='color:red'>default 브랜치를 info 로 바꿔서 작성 후 커밋하던가 직접 해당 브랜치에 폴더와 파일을 추가해줘도 됩니다. 파일은 프로젝트 폴더의 최상위에 .github/workflow/*.yml 로 위치시키면 됩니다.</span>
  
![77777](https://user-images.githubusercontent.com/76927397/124060201-80a95600-da67-11eb-984c-1040c7141d8e.PNG)
   
> info.yml  


```yml  
{% raw %}
name: lambda-CoronaInfo-CI  
# info 브랜치에서 push가 된 경우 실행합니다.
on:
  push:
    branches:
      - info

# 그룹의 역할입니다.
jobs:
  deploy_source:
    name: build and deploy lambda
    strategy:
      matrix:
        node-version: [14]
    runs-on: ubuntu-latest
    # Job 안에서 순차적으로 실행되는 프로세스 단위입니다.
    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node-version }}
      - name: Install dependencies, Build application and Zip dist folder contents
        run: |
          npm ci
          npm run build --if-present
        env:
          CI: true
      - name: zip
        uses: montudor/action-zip@v0.1.0
        with:
          args: zip -qq -r ./bundle.zip ./
      - name: default deploy
        uses: appleboy/lambda-action@master
        with:  
          # aws 키와 비밀키 , 리전 , 함수 이름입니다.
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws_region: ap-northeast-2
          function_name: CoronaInfo
          zip_file: bundle.zip
{% endraw %}
```

<br>  
  
* 워크플로우가 실행중임을 확인 할 수 있습니다.  
  
![88888](https://user-images.githubusercontent.com/76927397/124062363-93258e80-da6b-11eb-9c96-8a4581f30651.PNG)

<br>  
  
* 워크플로우의 상세한 과정을 확인할 수 있고 정상적으로 완료가 되었습니다. 이제 실제로 CoronaInfo 람다 함수도 변경되었는지 확인해볼까요 
 
![10101010](https://user-images.githubusercontent.com/76927397/124062545-d5e76680-da6b-11eb-88c9-dd246680e8b1.PNG)
  
<br>  
  
* 오! 깃허브에 푸시했던 그대로 CoronaInfo 람다 함수가 변경되어있습니다.  
  
![111111](https://user-images.githubusercontent.com/76927397/124063075-fa900e00-da6c-11eb-8cce-465ee18f6076.PNG)
  
<br>  
  
* 정상적인 서비스를 위해 다시 원래 코드로 바꿔줍시다. 그냥 레포지토리 자체에서 수정하고 커밋했습니다.
  
![99999](https://user-images.githubusercontent.com/76927397/124063291-7a1ddd00-da6d-11eb-8f00-b989f90ca4c9.PNG)
  
<br>  
 
 
* info 브랜치에서의 푸시를 감지하고 워크플로우가 실행이 됩니다. 문제 없이 완료된다면 수정한대로 다시 배포가 될 것입니다.  
  
![121212121](https://user-images.githubusercontent.com/76927397/124063414-c79a4a00-da6d-11eb-9dcd-5b0aa0ab12d8.PNG)
  
<br>  

  
* 자동으로 배포가 완료되었습니다.  
  
![1313131313](https://user-images.githubusercontent.com/76927397/124063498-f57f8e80-da6d-11eb-9261-b0f12fdfb5b4.PNG)
  
<br>  
<br>  

### <span style='color:red'>마치며</span>  
  
* **이전에 만들어 놓은 간단한 aws 서버리스 서비스를 배포 자동화 해보았습니다.**   

* **코드 변경이나 의존성 추가, 제거 등 작업을 aws 콘솔에서 직접 하다가 에러를 맞이하거나 코드를 날려먹을 일 없이 로컬에서 코드변경 , 의존성을 수정해주고 원격 저장소에 푸시만 해준다면  
  알아서 서비스 중단 없이 배포하고 워크플로우 파일에 추가한다면 원하는 테스트도 모두 수행할 수 있게 되었습니다.**  

* **솔직히 워크플로우 yml 파일의 모든 부분부분에 대해 정확히 알지도 못하지고 node.js를 배워본 적도 없지만 언어를 활용해보고 코드 배포 자동화의 큰 틀 정도에 노크는 해보지 않았나 싶어 긍정적인 실습시간이었다고 생각합니다** 
  
<br>  
  
 
  
#### 저장소  
  
[https://github.com/pds0309/aws_lambda_simple_sms_service](https://github.com/pds0309/aws_lambda_simple_sms_service) 
 
<br>  
  

  
#### Reference     
  
* yml파일 ``` uses: appleboy/lambda-action@master ```에서 알 수 있 듯 [appleboy/lambda-action](https://github.com/appleboy/lambda-action) 에서 정의된 람다코드 배포 action을 활용하였습니다.  
> 처음에 아 ~ 내 깃헙이름이랑 저장소에 브랜치 쓰면되는구나~~ㅎㅎ 하고 pds0309/aws_lambda_simple_sms_service@main 썻다가 action.yml 또는 도커파일이 없다는 에러를 마주하며 삽질을 했었습니다..  해당 부분에서 action설정 파일을 가져와 사용하겠다는 뜻이었습니다.
  
* [npm install 과 npm ci 의 차이점은 무엇입니까?](https://pythonq.com/so/npm/15391)  

* Hwasurr's Devlog 님의 [Github Actions를 이용해 CI/CD 파이프라인 구성하기 ](https://hwasurr.io/git-github/github-actions/)  
