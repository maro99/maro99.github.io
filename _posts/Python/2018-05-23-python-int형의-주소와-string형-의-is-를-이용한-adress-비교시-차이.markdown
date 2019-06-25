---
layout: "post"
title: "Python int형의 주소와 string형 의 is 를 이용한 adress 비교시 차이"
categories:
 - Python
tags:
 - Python
comments : true
date: "2018-05-23 12:37"
---


<br>
문자열의 경우
```bash
a1='123123123123'
b1='123123123123'
print(a1 is b1)
print( 'a1 adress:',id(a1))
print( 'b1 adress:',id(b1))

import sys
sys.getsizeof(a1)
```
결과
```bash
True
a1 adress: 140227742309936
b1 adress: 140227742309936
61
```
문자열의 경우 그 길이가 길어도 그것이 같다면 주소가 같다.
<br>



<br>
작은 int의 경우
```bash
a2= 12
b2= 12
print(a2 is b2)
print( 'a2 adress:',id(a2))
print( 'b2 adress:',id(b2))

import sys
sys.getsizeof(a2)
```
결과
```bash
True
a2 adress: 93884359181440
b2 adress: 93884359181440
28
```
int의 경우 작을때는 그값만 같으면 주소도 같다.
<br>





<br>
 큰 int의 경우
```bash
a3= 123123123123
b3= 123123123123
print(a3 is b3)
print( 'a3 adress:',id(a3))
print( 'b3 adress:',id(b3))

import sys
sys.getsizeof(a3)
```
결과
```bash
False
a3 adress: 140227742331568
b3 adress: 140227742331280
32
```
<br>
