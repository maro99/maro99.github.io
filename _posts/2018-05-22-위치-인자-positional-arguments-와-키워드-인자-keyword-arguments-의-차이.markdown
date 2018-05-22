---
layout: "post"
title: "위치 인자(Positional arguments) 와 키워드 인자(Keyword arguments)의 차이
"
date: "2018-05-22 15:54"
---





#### 위치 인자(Positional arguments)
&nbsp; 매개변수의 **순서대로** 인자를 전달하여  사용하는 경우

선언
<pre>
def student(name, age, gender):  
  return {'name': name, 'age': age, 'gender': gender}
</pre>

사용
<pre> 
student('hanyeong.lee', 30, 'male')  
  {'name': 'hanyeong.lee', 'age': 30, 'gender': 'male'}
</pre>




<br>
<br>


#### 키워드 인자(Keyword arguments)
&nbsp; 매개변수의 이름을 지정하여 **인자로 전달하여** 사용하는 경우

선언
<pre>
student(age=30, name='hanyeong.lee', gender='male')
  {'name': 'hanyeong.lee', 'age': 30, 'gender': 'male'}
</pre>

사용
<pre>
위치인자와 키워드인자를 동시에 쓴다면, 위치인자가 먼저 와야 한다
student(name='marl','27','male') 이렇게하면 호출 안된다. 한개가 위치인자면 그 뒤도 위치인자 써야한다.
</pre>
