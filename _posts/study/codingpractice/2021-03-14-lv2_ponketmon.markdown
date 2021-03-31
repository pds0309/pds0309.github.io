---
layout: post
title: 프로그래머스-lv2 폰켓몬 (java)
date: 2021-03-14 09:17:45 +0900
category: 111CP
tag: study
mytag: [프로그래머스, 기타]
---

## [프로그래머스] level 2 - 폰켓몬

---
### [문제설명링크   ](https://programmers.co.kr/learn/courses/30/lessons/1845)https://programmers.co.kr/learn/courses/30/lessons/1845
<br>
### 요약
* 홍 박사님이 N마리의 몹 중 N/2 마리를 주는데 최대한 다양한 종류를 선택할 때 몹의 종류번호 수를 구하라.

<br>
### 제한사항
* nums는 폰켓몬의 종류 번호가 담긴 1차원 배열입니다.
* nums의 길이(N)는 1 이상 10,000 이하의 자연수이며, 항상 짝수로 주어집니다.
* 폰켓몬의 종류 번호는 1 이상 200,000 이하의 자연수로 나타냅니다.
* 가장 많은 종류의 폰켓몬을 선택하는 방법이 여러 가지인 경우에도, 선택할 수 있는 폰켓몬 종류 개수의 최댓값 하나만 return 하면 됩니다.
<br>


<br>
### 입출력 예시

| Nums |result|
|--------------------|---|
|[3,1,2,3] |2|
|[3,3,3,2,2,4]|3|
|[3,3,3,2,2,2]|2|



<br>

### 나의 풀이

* 종류의 개수를 구하는 것이니 우선 중복을 제거한다.
* 중복을 제거한 배열의 길이와 내가 가질 수 있는 몹(N/2)의 수를 비교한다.
<br>

<br>
<br>
### 자바 풀이 코드
```java
import java.util.Arrays;
class Solution {
    public int solution(int[] nums) {
        int answer = 0;
        // ans = 주어진 배열 nums 에서 중복을 제거(distinct()) 한 후 카운트 한 값
        int ans = (int)Arrays.stream(nums).distinct().count();
        answer = ans;
        // 내가 가질 수 있는 몹의 수보다 종류가 많을 경우 
        answer = ans > nums.length/2 ? nums.length/2 : answer;
        return answer;
    }
}
```

#### 체감 난이도 :  하
* 레벨 2인 것에 비해 푼 사람도 적고 문제도 길어서 어려울거라 생각했는데 다들 읽기 귀찮아서 그랬던 것이었다.
