---
layout: "post"
title: "python 제네레이터 (generator) "
categories:
 - Python
tags:
 - Python
 - generator
date: "2018-05-22 17:16"
---


>제네레이터는 함수는 파이썬의 **시퀀스 데이터를 생성하는데 사용** 된다. 실제 시퀀스 데이터와 다른 점은, 시퀀스 전체를 가지고 있는 것이 아니라 시퀀스 데이터를 생성하기 위한 어떠한 루틴만을 가지고 있는 것이다. 이 방식을 택했을 때의 장점은, 전체 크기만큼의 메모리를 가지고 있는 시퀀스 데이터와는 달리 **메모리를 적게 사용** 할 수 있다. 제네레이터는 마지막으로 호출한 위치(항목)에 을 기억하고 있으며, 한 번 순회할 때 마다 그 다음 값을 반환한다. 제네레이터는 함수를 통해서 만들어지며, 함수 **내부의 반복문에서 yield키워드를 사용** 하면 제네레이터가 된다.

<br><br>
우리가 흔히 썼던 range에대해서 먼저 살펴보자.  
range 문법을쓸때  
result =range(10)  
print(result) 하면 그결과 안나오고
type(range) 해보면 range나왔다.  
result= range(10000000000000000000000000000000)  
이뜻은 range가 영부터~까지를 가져서 메모리에 남긴게 아니라 ~까지 갈수있다는 로직을 남긴것이다.
그걸실제로 메모리에 모두 저장하면 컴퓨터에 과부하가 걸린다.
(*반대로 리스트데이터는 모든 값을 다 할당해 가지고 싶다면 실제값 가지고싶으면 리스트 값으로 변환하면 된다.
list(result)*)

--------------->**range는 제너레이터와 하는 역할이 정확히 같다**.


<br><br>

데이터를 모두 가지고 있을필요 없이 순회할때마다 값을 가져다 쓰는 range와 같은 역할을 하는 제너레이터 만들어 보겠다.

<br>

&nbsp;&nbsp;&nbsp; 선언

```bash
def range_gen(num):
   i = 0
   while i < num:  #i보다 작은동안.
     yield i
     i += 1

```
&nbsp;&nbsp;&nbsp; 사용

```bash
>>> gen = range_gen(10)
>>> gen           ---->type(gen)------->generator  for문으로 레인지처럼 출력해봐도 좋다.
<generator object range_gen at 0x10b682168>
>>> type(gen)
<class 'generator'>
>>> gen.__next__()          #포문 안에서도 결국 이것이 호출되는것.
0
>>> gen.__next__()
1
>>> gen.__next__()
2
>>> gen.__next__()
3
>>> gen.__next__()
4
>>> gen.__next__()
5
>>> gen.__next__()
6
>>> gen.__next__()
7
>>> gen.__next__()
8
>>> gen.__next__()
9
>>> gen.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

함수 내부에 yield키워드가 사용되어 제네레이터 함수가 되었으며, 함수를 실행하면 제네레이터 객체를 반환한다.
yield부분에서 멈춘 제네레이터 객체를 순회하기 위해서는 __next__() 함수를 실행해준다.



<br><br><br>
generator로 구간을 생성해 보면? 숫자 두개 (1,2) 이런식으로 받아서.

&nbsp;&nbsp;&nbsp; 선언
```bash
def range_gen(start_num,end_num):
    =start_num
    while i< end_num:
        yield i
        i+=1
```
&nbsp;&nbsp;&nbsp; 사용

```bash
for i in range_gen(10,20):
    print (i)
```
&nbsp;&nbsp;&nbsp; 결과

```bash
10
11
12
13
14
15
16
17
18
19
```


<br>

이번에는 스텝도 표현해보면?


&nbsp;&nbsp;&nbsp; 선언
```bash
def range_gen(start_num,end_num,step):
i=start_num
while i< end_num:
yield i
i+=step
```


<br>

빽스텝도 가능하게 하기


&nbsp;&nbsp;&nbsp; 선언
```bash
def range_gen(start_num,end_num,step):
    i=start_num
    if step< 0:
        while i> end_num:
            yield i
            i+=step
    elif step > 0:
            while i<end_num:
            yield i
            i+=step
```



<br>

입력시 초기값 줘서 스타트 넘, 엔드넘 등 입력 안해도 되도록.


&nbsp;&nbsp;&nbsp; 선언
```bash
def range_gen(start_num=0,end_num=0,step=1):
    i=start_num
    if step< 0:
        while i> end_num:
            yield i
            i+=step
    elif step > 0:
        while i<end_num:
            yield i
            i+=step
```
&nbsp;&nbsp;&nbsp; 사용

```bash
for i in range_gen(end_num=11):
    print(i)
```
키워드 인자 저렇게 안주고 보통 레인지 쓸때처럼 값을( , 11, ) 이런식으로 주고 싶은데 잘 모르겠다...
