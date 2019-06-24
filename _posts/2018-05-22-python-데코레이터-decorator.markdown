---
layout: "post"
title: "Python 데코레이터(Decorator)"
categories:
 - Python
tags:
 - Python
 - decorator
date: "2018-05-22 16:52"
---

>데코레이터 (decorator)  
함수를 받아 다른 함수를 반환하는 함수. 예를 들면, 기존에 존재하던 함수를 바꾸지 않고 전달된 인자를 보기위한 디버깅 print함수를 추가한다던가 하는 기능을 할 수 있다.


<br>

각각 하나의 기능들을 가지는 함수가 다음과 같이 있다고 하자.
```bash
def square(x):
    return x**2

def double(x):
    return x*2

def half(x):
    return x/2
```

<br>

이 함수 들에 우리는 공통적으로 받은 arguemnt들을 출력하는 프린트문을 다음과같이 넣고싶다.
```bash
def square(x):
    print(x)
    return x**2

def double(x):
    print(x)
    return x*2

def half(x):
    print(x)
    return x/2
```
하지만 이렇게 완벽하게 똑같은 동작을 하는 구문들을 반복해서 프로그램 내에 작성하는 것은 프로그래머가 해서는 안되는 행위이다.
그래서 데코레이터라는 함수를 만들어서 동일한 기능들을 대처하겠다.

<br><br>

다음과 같은 세부사항으로 함수를 만들었다.

*1.이 함수는 original_function을 매개변수로 받아서  
2.내부의 decorated_function을 리턴해준다.  
3.decorated_function은 모든 가변인자를 전달받을 수 있으며  
4.모든 가변인자 목록을 출력후 original_function에 가변인자들을 모두 전달해 호출한 결과를 리턴한다.  
print_args --> 함수를 받아 그 함수를 포함한 다른 함수를 리턴해주는 decorator 함수*


&nbsp;&nbsp;선언
```bash
def print_args(original_function):                          #데코레이터
     # decorated_funtion->original_function을 한번 감싼 함수   #데코레이터 함수.
    def decorated_function(*args, **kwargs):
        print(f'args: {args}')
        print(f'kargs: {kwargs}')
        return original_function(*args, **kwargs)

    return decorated_function

```

&nbsp;&nbsp;사용
```bash
decorated_square = print_args(square)
decorated_square(5)
```
&nbsp;&nbsp;결과  
args :{ 5. }  
kwargs: {}  
out:25


<br>

간편하게 이렇게도 사용한다.
square 함수가 자동으로 데코레이터 함스로 바뀜
square에 데코레이트 한 적 없지만 그것과 같은 결과남.

&nbsp;&nbsp;선언
```bash
@print_args      #이걸로 감쌌다고 보면됨.

def square(x):
    return x*3
```

&nbsp;&nbsp;사용.
```bash
 square(3)
```
