---
layout: "post"
title: "Python get/set 속성"
categories:
 - Python
tags:
 - Python
 - private     
 - getter   
 - setter 
date: "2018-05-24 19:22"
---

파이썬에서는 정확히 말해서는 private를 지원하지는 않지만 비슷하게 동작하는 private 지정자 __ 가존재한다.
그렇기 때문에 다른언어에서 getter,setter 를 사용하듯 private속성에 접근하기 위해 getter와 sette 메서드를 사용한다.
파이썬에서는 다음과 같이 해당 기능을 property를 사용해 구현한다.


<br>
<br>


```bash
@property
def name(self):
    return self.__name

@name.setter
def name(self, new_name):
    self.__name = new_name
    print('Set new name ({})'.format(self.__name))
setter
```
이때 메서드의 이름은 변수(속성)의 이름과 동일하게 하는것이 관례이다.  
setter의 name과 propery 앞에 name앞에 @ 가 붙었는데 이를 decorator 라고하며 뭔가 꾸며주는 함수를 의미한다.



<br>


예제를 보자.

```bash
class Shop:

    def __init__(self,name,__shop_type,address):
        self.name=name
        self.__shop_type=__shop_type
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

위와같은 상황에서 private인 __ shop_type 에 접근하기 위해서
보통의 언어는 다음과 같이 setter 와 getter를 선언해서 쓸 것이다.


```bash
#getter방식
    def get_shop_type(self):
        return self.__shop_type

#setter 방식
    def set_shop_type(self,new_shop_type):
        shop_type_list=['래스토랑','패스트푸드']   #어떤 필터 기능을 추가 할 수 있다.

        if new_shop_type in shop_type_list:
            self.__shop_type=new_shop_type
        else:
            print(f'입력하신 유형({new_shop_type})은 사용할 수 없습니다.')
            self.__shop_type=new_shop_type
```

(클래스 내부에서는 private 변수에 접근 가능하니 클래스에 하위의 메소드를 만들어서 변수를 내놓고 만드는 단순한 기능을 하게 한것.
세터의 경우는 거기서 원하지 않는 값으로 바꾸는것을 필터링 해주는 것을 추가한것.)

<br>
파이썬에서는 다음과 같이 선언해서 쓴다.

```bash
@property
def shop_type(self):
    return self.__shop_type

@shop_type.setter
def shop_type(self,new_name):
    self.__shop_type=new_name
    print(f'새 상점 내용은 ({ self.__shop_type})')
```

첫번째 getter/setter는 다음과 같이 사용해봤다.  
사용
```bash
#일반적인 언어처럼 게터 세터 선언한것 사용해보면
marumaru= Shop('marumaru','cartoonshop','답십리')
marumaru.get_shop_type()
marumaru.set_shop_type('오락실')
marumaru.get_shop_type()


```
결과
```bash
Set new name (marumaru)
입력하신 유형(오락실)은 사용할 수 없습니다.
Out[20]:
'오락실'

```

두번째 파이썬에서의 보통의  getter/setter는 다음과 같이 사용해봤다.  
사용
```bash
marumaru= Shop('marumaru','cartoonshop','답십리')
marumaru.shop_type
marumaru.shop_type='super fast food'
#샵타입이라는 것을 마치 속성 처럼 쓸 수 있게됨.실제로는 내부에서함수가 동작해서~리턴해준것

```
결과
```bash
Set new name (marumaru)
Out[21]:
'cartoonshop'
새 상점 내용은 (super fast food)
```
