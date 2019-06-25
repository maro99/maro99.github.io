---
layout: "post"
title: "Python class self parameter"
categories:
 - Python
tags:
 - Python
 - class
 - self   
comments : true   
date: "2018-05-24 18:43"
---

다음 클래스 선언 예제에서 self 의 역할에대해 살펴보자.

```bash
class Shop:
    def __init__(self, name):  //생성된 객체에서 자동으로 init 메서드 호출한다.
        self.name = name        //생성된 객체 자체의 name 을 클래스 생성시 입력한 name으로 바꾼다.

```

이렇게 선언한 클래스는 다음과 같이 사용할 수 있다.

```bash
lotteria = Shop('Lotteria')
Burgerking= Shop('Burgerking')

print(lotteria.name)
print(Burgerking.name)
```
다음과 같은 순서로 코드는 작동한다.

1. Shop 클래스가 정의되었는지 찾는다.
2. Shop 클래스형 객체를 메모리에 생성한다.
3. 생성한 객체의 초기화 메서드 __init__()을 호출한다.
4. name값을 저장하고, 만들어진 객체를 반환한다.
5. lotteria변수에 반환된 객체를 할당한다


여기서 self의 역할 을 살펴보려면 self 가 없을때를 가정해본다.
만약 self가 없다면 객체의 초기화 메서드를 호출에서 name값에 Lotteria, Burgerkig 등의 받은 스트링을 저장할때마다
같은 name이라는 변수에 저장하게 되면서 마지막으로 객체를 생성했을때 입력한 스트링이 name변수에 저장되게 될것이다.  

하지만 우리가 원하는 것은 각각의 객체마다 각각의 고유한 name변수를 가지는 것이다.
self는 생성되는 객체를 정확히는 객체의 주소를 나타내는 것이다.
이것을 아직 객체가 생성되지도 않은 상태에서 객체의 빵틀 격인 클래스 안에서 딱찝어 말하기가 힘들기 때문에 self라는것으로 생성될 객체를 나타내서 self.name 이런식으로 그 객체의 고유한 name변수를 특정 짓는 것이다.
