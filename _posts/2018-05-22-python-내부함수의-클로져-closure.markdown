---
layout: "post"
title: "내부함수의 클로져(Closure)"
categories:
 - Python
tags:
 - Python  
 - closure  
 - 파이썬 문법
date: "2018-05-22 16:45"
---

>클로져 (Closure)  
함수가 정의된 환경을 말하며, 파이썬 파일이 여러개일 경우 각 파일은 하나의 모듈역할을 하고, 각 모듈은 독립적인 환경을 가진다.  
독립된 환경은 각자의 영역을 전역 영역으로 사용한다.



#### 내부함수의 클로져

```bash
level = 0
def outer():
   level = 50

   def inner():
     nonlocal level
     level += 3
     print(level)
   return inner   #inner() 함수의 호출 결과가 아니라 inner라는 함수 자체를 리턴함.

 func = outer() # 이결과는 어떤 객체인가? 함수이다.
```
이후 func() 호출할때마다 3씩 50 에서 증가한다.  
밖의 레벨 값을 참조하고 있어서.  
그런데 우리는 outer() 실행 결과 인 inner 함수만 리턴했다.  
밖의 레벨과는 관계업다. 그 순간 함수를 실행해서 nonlocal
받아온게 아니라 함수만 준것.
하지만 이 함수가 동작하는 모든 영역을 다 포함 시키는 것이 내부함수의 클로져.
그리고 이것은 독립적이다.

<br><br>

```bash
func11=outer()
func22=outer()

func11()   #----->53
func11()   #------>56
func22()   #---->53출력된다. 독립적이다. 새로운 주소 할당되서
```

위의 경우, outer함수는 inner함수를 반환하여 결과를 func전역변수에 할당한다.  
inner함수의 호출 결과가 아닌 함수 자체를 반환하기 때문에, func변수는 실행할 수 있는 함수객체이다.  
이 때 inner함수가 사용하는 level변수는 nonlocal키워드를 사용했기 때문에 outer에 새로 정의된 지역변수 level을 사용한다.  
반복적으로 func함수를 실행하면 inner의 외부(outer)에 존재하는 level변수의 값을 증가시키고 print시키기 때문에, 값은 계속해서 증가한다.
