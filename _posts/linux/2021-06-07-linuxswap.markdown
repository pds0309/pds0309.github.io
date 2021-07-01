---
layout: post
title: (Ubuntu 20.04) Swap공간 생성하기
date: 2021-06-07 18:14:22 +0900
category2: linux
category: Study Note
tag: [linux,cloud]
---
<br>

<br>  




## 오라클 클라우드 Ubuntu 인스턴스에 Swap 파티션 공간을 만들기
<hr>  
 
<br>  



### <span style='color:red'>개요</span>  

  
  
**공부 목적으로 간단한 스프링부트 웹 서비스를 오라클 클라우드 리눅스에 띄워 놓고 있었는데..**  
**스프링 부트와 Rserve 프로세스 고작 몇 개 떠 있는데도 메모리가 부족했습니다.**  

>kern.log  

```linux 
Jun  7 06:08:42 instance-**** kernel: [3906635.277344] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,global_oom,task_memcg=/user.slice/user-1001.slice/session-1320.scope,task=java,pid=232693,uid=0
Jun  7 06:08:42 instance-**** kernel: [3906635.277392] Out of memory: Killed process 232693 (java) total-vm:2684284kB, anon-rss:320384kB, file-rss:0kB, shmem-rss:0kB, UID:0 pgtables:880kB oom_score_adj:0
Jun  7 06:08:43 instance-**** kernel: [3906635.391116] oom_reaper: reaped process 232693 (java), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB

```  

![](/assets/img/linux1/linux1-1.PNG)
  
* **오라클 클라우드 프리티어를 사용중인데 RAM 이 1GB 밖에 안됩니다.**    

* **오라클 클라우드 평생무료 인스턴스는 하나의 인스턴스에 RAM 1GB까지 밖에 허용이 되지 않아 스왑 파티션으로 하드 디스크 일부를 사용하기로 했습니다.** 
  
<br>  

### <span style='color:red'>스왑 파티션?</span>  
  
* 시스템 메모리가 부족할 경우 하드디스크 일부 공간을 활용하게 해주는 영역입니다.  
  
  
### <span style='color:red'>스왑 공간 생성하기</span>  
  
**[오라클 클라우드 Ubuntu 20.04 인스턴스 기본 설정하기](https://www.wsgvet.com/cloud/6?page=1) 를 참고했습니다.**  
* RAM이 2GB 미만이면 현재 RAM 의 두 배를 권장한다고 합니다.  
  
<br>  


<span style='color:blue'>   (1) 2GB 의 스왑공간 생성 및 권한 조정  </span>


```linux
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
```
<br>  

<span style='color:blue'>   (2) 스왑파일 생성  </span>	 


```linux
sudo mkswap /swapfile
```
> 다음과 같은 메시지가 출력됩니다.  

```linux
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
no label, UUID=71432f3c-95eb-4c80-af06-dc0af6848c1b
```
<br>  

<span style='color:blue'>  (3) 스왑 공간 사용  </span>

```linux
sudo swapon /swapfile
```

<br>  

<span style='color:blue'>   (4) 재부팅시 유지되게 세팅하기  </span>

*  /etc/fstab 파일에 해당 내용을 추가합니다  

```linux
/swapfile swap swap defaults 0 0
```
  
<br>  

<span style='color:blue'>   (5) 확인 </span>
```linux
ubuntu@instance-******:~/ff4$ free -h
              total        used        free      shared  buff/cache   available
Mem:          974Mi       625Mi        83Mi       1.0Mi       266Mi       200Mi
Swap:         2.0Gi       0.0Ki       2.0Gi
``` 

#### Reference
  
[오라클 클라우드 Ubuntu 20.04 인스턴스 기본 설정하기](https://www.wsgvet.com/cloud/6?page=1)  


#### 해당 내용을 다뤘던 이슈입니다.  
[백그라운드 실행 도중 강제 종료됨](https://github.com/pds0309/springboot-simple-fifaonline4-web/issues/112#issuecomment-864342888)