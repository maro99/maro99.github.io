---
layout: "post"
title: "global키워드와 인자(argument)전달의 차이"
categories:
 - Python
tags:
 - Python
 - global
date: "2018-05-22 16:38"
---

#### 인자로 전달한 경우

```bash
global_level = 100
def level_add(value):
    value += 30
    print(value)

level_add(global_level)
print(global_level)
```
---->결과  
130  
100

global_level이라는 변수가  
어떤 메모리 공간에서 100이라는 값을 참조하고 있다.  
level_add 라는 함수가 value라는 parameter 가지는데 여기에 global_level 전달했다.
이 value가 global_level과 같은곳을 가리킨다.
value에 +=30하자 메모리에 130이라는 값이 생기고 100과의 연결이 끊기며 value만 130을 가리킨다.  
이 함수가 끝난 뒤에는 130을 참조한 곳이 사라지고 100은 계속 남아있는다.



<br><br>
#### global키워드를 사용한 경우

```bash
global_level = 100
def level_add():
    global global_level
    global_level += 30
    print(global_level)

level_add()
print(global_level)
```

결과--->  
130  
130

인자로 전달한 경우, 같은 객체를 가리키는 글로벌 변수 global_level과 매개변수 value가 존재한다.  
이 때, 매개변수인 value의 값을 변경하는 것은 global_level에는 영향을 주지 않는다.  
global키워드의 경우 둘은 같은 변수이다.
