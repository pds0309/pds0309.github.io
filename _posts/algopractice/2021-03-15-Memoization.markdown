---
layout: post
title: 메모이제이션 (Memoization)
date: 2021-03-15 11:25:00 +0900
category2: etc
category: Study Note
tag: [dp,algorithm]
---

## 메모이제이션이란?

* 반복되는 결과를 저장해두고 이후에 같은 결과가 다시 나올 때 재사용하는 것
* 이미 저장된 값이 있을 때 함수를 추가로 호출 할 필요 없음
-> 시간비용 절약 가능

<br>
### 예시 : 피보나치 수열
<br>
> 재귀함수

```java
static int fibo(int N) {
        if (N <= 1) {
            return N;
        } else {
            return fibo(N - 1) + fibo(N - 2);
        }
    }
```


```java
fibo(5);
5 4 3 2 1 0 1 2 1 0 3 2 1 0 1 
```
* N에 대한 가능한 모든 함수들을 탐색해야 해서 비효율적이다. <br>이미 연산했던 값을 반복한다.


<br>

> 메모이제이션

```java
static HashMap<Integer, Integer> hs = new HashMap<>();

    static int fiboMemoization(int N) {
        System.out.print(N + " ");
        if(hs.containsKey(N)){
            return hs.get(N);
        }
        if(N<=1){
            return N;
        }
        else{
            int res = fiboMemoization(N-1) + fiboMemoization(N-2);
            hs.put(N , res);
            return res;
        }
    }
```

```java
fiboMemoization(5);
5 4 3 2 1 0 1 2 3 
```
* 이미 연산해서 저장되었던 값을 중복해서 탐색하지 않는다.

<br><br>
```python
####### N 이 25 ######
재귀함수 걸린시간 :  0.0
메모 걸린시간 :  0.0
####### N 이 30 ######
재귀함수 걸린시간 :  3.0
메모 걸린시간 :  0.0
####### N 이 35 ######
재귀함수 걸린시간 :  26.0
메모 걸린시간 :  0.0
####### N 이 40 ######
재귀함수 걸린시간 :  299.0
메모 걸린시간 :  0.0
####### N 이 45 ######
재귀함수 걸린시간 :  3231.0
메모 걸린시간 :  0.0
```
* 성능차이가 확실하다.