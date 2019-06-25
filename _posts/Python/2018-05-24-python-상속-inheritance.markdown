---
layout: "post"
title: "Python 상속 (Inheritance)"
categories:
 - Python
tags:
 - Python
 - 상속  
comments : true
date: "2018-05-24 20:01"
---

>기존의 구현해 놓은 클래스의 기능을 거의 대부분 가진 다른 기능을 추가적으로 가진 클래스가 필요할때
**상속**을 사용한다.이 때, 상속 되는 클래스를 **부모**(상위)클래스라고 하며, 상속을 받는 클래스는 **자식**(하위)클래스라고 한다.

상속을 받을때는 클래스의 정의 다음 괄호에 부모 클래스를 적어주면 된다.

```bash
class Restaurant(Shop):
    pass
```

>  **메서드 오브라이드**  
> 상속받은 클래스에서, 부모 클래스의 메서드와는 다른 동작을 하도록 할 수 있다. 이 경우 부모 클래스의 메서드를 덮어씌워서 사용하도록 하며, 이 방법을 메서드 오버라이드(method override)라고 한다.



<br><br>
다음을 상속 받는 클래스를 추가해서 상속받는 클래스 에서는  
show_info 호출시 상점정보 대신 식당정보를 print 해보자.  
<br>
주어진 클래스
```bash
class Shop:

    def __init__(self,name,_shop_type,address):
        self.name=name
        self._shop_type=_shop_type
        self.address=address
    def show_info(self):
        print(f"상점정보 ({self.name})")
        print(f" 유형: {self._shop_type}")
        print(f" 주소: {self.address}")
    def change_type(self):
        self._shop_type=input("바꾸고싶은 샵파입을 입력하시오: ")

    @property
    def shop_type(self):
        return self._shop_type

    @shop_type.setter
    def shop_type(self,new_name):
        self._shop_type=new_name
        print(f'새 상점 내용은 ({ self.__shop_type})')

```
추가한 자식 클래  

```bash
class Restaurant(Shop):
    def show_info(self):
        print(f"상점정보 ({self.name})")
        print(f" 유형: {self._shop_type}")
        print(f" 주소: {self.address}")
```
프라이벳으로 지정해서 상속받은 곳에서도 쓰지 못함.
자식에서 가져다 쓰기위해 관례적으로는 _ 한계만 쓴다(맹글링도 안된다.)
외부에서도 _으로 접근 가능하지만 외부에서 최대한 자제하라 는것.
상속받은곳에서 좀더 쓰기 위해서 이렇게 씀. 이게 protected 속성 갖는다고 생각하는것,
---->제정의 하려고 전체를 다시 다쓴것은 어리석어보인다.
부모클래스에서 하던일을 자식에서 가져다가 필요한 것만 붙여 쓰고 싶다.


<br>
<br>
그래서 등장한 것이 부모클래스의 메서드를 호출하는 super()메서드
<br>
<br>


###### 부모 클래스의 메서드를 호출 (super)
> 자식클래스의 메서드에서 부모 클래스에서 사용하는 메서드의 전체를 새로 쓰는것이 아닌, 부모 클래스의 메서드를 호출 후 해당 내용으로 새로운 작업을 해야 할 경우 super()메서드를 사용해서 부모 클래스의 메서드를 직접 호출할 수 있다.



```bash
class Restaurant(Shop):
    def __init__(self, name, shop_type, address, rating):#레이팅만 추가해 주고 싶음.
        super().__init__(name, shop_type, address) #부모에있던것 가져옴
        self.rating = rating
```
위 코드의 경우, super()메서드를 사용해서 부모의 __init__ 메서드를 호출한다.


<br>

한가지 문제를 생각해보자.  
새로추가하는 rating 변수에 대해서도 show_info()를 사용해서 출력하고 싶은데 만약 부모의 show_info메서드를 오버라이드 할때 super를 이용하게 되면 부모의 show_info 메서드에서 정보를 출력해 줬던 것에 대해서 변경을 못하기 때문에 문제가 밣생하다. 우리는 이미 출력된 그 문장중에서 상점정보의 상점을 빼고 식당을 넣고, rating변수도 출력하고 싶다.
---->  
<br>
해결방법  

1. info 프로퍼티를 정의하고
2. 리턴하는 내용은 아래의 show_info가 출력하는 문자열
3. 인스턴스 메서드를 만들고 show_info가 출력하던 문자열을리턴
4. 메서드 @property 데코러이터만 붙이면 완성.
5. subway.info 로 출력하면 됨
(subway.info라는 문자열을 다룰 수 있게 되면서 자식의 info, show_info에서 세부적인 재정의가 가능해진다.)



부모클래스 선언  
```bash
class Shop:

    def __init__(self,name,_shop_type,address):
        self.name=name
        self._shop_type=_shop_type
        self.address=address


    @property
    def info(self):
        return (
        f"상점정보: ({self.name}) \n"
        f"   유형: ({self.shop_type}) \n"
        f"   주소: ({self.address}) \n"   

        )


    def show_info(self):
        print(self.info)

    def change_type(self):
        self._shop_type=input("바꾸고싶은 샵파입을 입력하시오: ")

    @property
    def shop_type(self):
        return self._shop_type

    @shop_type.setter
    def shop_type(self,new_name):
        self._shop_type=new_name
        print(f'새 상점 내용은 ({ self.__shop_type})')
```

자식클래스 선언

```bash
class Restaurant(Shop):
    def __init__(self,name,_shop_type,address,rating):
        super().__init__(name,_shop_type,address)#self는 부모불를때 안넣어도된다.
        self.rating=rating


    @property
    def info(self):
        return super().info.replace('상점','식당')


    def show_info(self):
        super().show_info()#self는 부모불를때 안넣어도된다
        print( f"  레이팅: ({self.rating})")
```

사용
```bash
subway = Restaurant('서브위이', '페스트푸드', '성수역','100')
subway.show_info()
```

결과

```bash
  식당정보: (서브위이)
   유형: (페스트푸드)
   주소: (성수역)
   레이팅: (100)
```
