---
layout: "post"
title: "python 속성 접근 지정자 (attribute acccess modifier)"
categories:
 - Python
tags:
 - Python
 - 속성접근지정자
date: "2018-05-24 19:09"
---

>**캡슐화**  
객체를 구현할 때, 사용자(다른 개발자) 가 반드시 알아야 할 데이터나 메서드를 제외한 부분을 은닉시켜 정해진 방법을 통해서만 객체를 조작할 수 있도록 하는 방식.

<br>
<br>


#### private
 속성 이름을 __으로 시작.  
 외부에서의 접근을 제한한다.

다음 주어진 예제의 shop_type 변경해서 private로 만들어보자.

변경전

```bash
class Shop:

    def __init__(self,name,shop_type,address):
        self.name=name
        self.shop_type=shop_type
        self.address=address

     def show_info(self):
        print(f"상점정보 ({self.name})")
        print(f" 유형: {self.shop_type}")
        print(f" 주소: {self.address}")

     def change_type(self):
        self.shop_type=input("바꾸고싶은 샵파입을 입력하시오: ")

    def change_type2(self,new_shop_type):
        self.shop_type=new_shop_type

```

이상태에선 객체를 만든 후에 그객체에  
lotteria.shoptype= '푸줏간' 이런식으로 접근해도 쉽게 바꿀 수 있다.


변경후

```bash
class Shop:

    def __init__(self,name,shop_type,address):  
        self.name=name
        self.__shop_type=shop_type
        self.address=address

     def show_info(self):
        print(f"상점정보 ({self.name})")
        print(f" 유형: {self.__shop_type}")
        print(f" 주소: {self.address}")

     def change_type(self):
        self.__shop_type=input("바꾸고싶은 샵파입을 입력하시오: ")

     def change_type2(self,new_shop_type):
        self.__shop_type=new_shop_type


```
일단 객체를 선언해보자.

```bash
marumaru= Shop('marumaru','cartoonshop','답십리')
marumaru.show_info()
marumaru.change_type()
marumaru.show_info()
```
실행결과
```bash
상점정보 (marumaru)
  유형: cartoonshop
  주소: 답십리
바꾸고싶은 샵파입을 입력하시오: '게임방'
상점정보 (marumaru)
  유형: '게임방'
  주소: 답십리
```
이후 다음과 같이 접근을 시도해보면접근이 안된다.
```bash
marumaru.shop_type
 marumaru.__shop_type  #둘다 접근이 안된다.
```
결과 -->에러
```bash
AttributeError: 'Shop' object has no attribute '__shop_type'
```

<br> <br>

dir(marumaru)  #이렇게 디렉토리를 조회해보면

```bash
['_Shop__shop_type',  이렇게 어트리부트 되어있다.
 '__class__',
 '__delattr__',
 '__dict__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__le__',
 '__lt__',
 '__module__',
 '__ne__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 '__weakref__',
 'address',
 'change_type',
 'change_type2',
 'name',
 'show_info']
```

파이썬에서는 이렇게 언어상에서 막은게 아니라 이름을 바꿔서 저런식으로 뒤섞어 놓은 것이다. 이것을 *네임 멩글링(name mangling)*이라고 한다.
저기에 접근하면 바꿀 수 있긴 하다.
marumar.Shop__shopname  ='asasfsaf
 파이썬에서는 사용자의 정중함에 의존하기 때문에 강제하지 않는부분이다.
