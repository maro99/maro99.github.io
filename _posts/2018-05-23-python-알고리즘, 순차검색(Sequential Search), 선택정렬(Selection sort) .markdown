---
layout: "post"
title: "Python 함수 알고리즘, 순차검색(Sequential Search), 선택정렬(Selection sort)"
date: "2018-05-23 12:07"
---


>**순차검색(Sequential Search)**
문자열과 키 문자 1개를 받는 함수 구현  
while문을 이용, 문자열에서 키 문자가 존재하는 index위치를 검사 후 해당 index를 리턴  
찾지 못했을 경우 -1을 리턴


혼자해본것

```bash
def SeqSearch(str1, key_char1):
    i=0
    index=-1
    while i<len(str1):
        if str1[i]==key_char1:
            index=i
            break
        i=i+1
    return index
```

만약 찾으려는 문자 1개가 중복해서 존재 한다면?
```bash
def SeqSearch(str1, key_char1):
    i=0
    index=[-1]
    while i<len(str1):
        if str1[i]==key_char1:
            index.append(i)

        i=i+1
    return index[0] if len(index)==1 else index[1:]
```

for문 을 사용해보면?

```bash
def SeqSearch2(str1, key_char):
    i=0
    for str1_char in str1:
        i=i+1
        if str1_char ==key_char:
            return i
```

enumerate 사용해서 좀더 간단히 해본다.

```bash
def SeqSearch2(str1, key_char):
    for index ,str1_char in enumerate(str1):
        if str1_char ==key_char:
            return index
```

<br><br>

고찰.  

10글자 중에 한글자를 찾는다면?
평균 5글자면은 해당 문자 찾을 수 있다.
가장 빨리 찾을경우 1, 가장 늦게 찾으면 10

n개의 문자열 있을때
평균적으로 2/n 만큼 걸린다.
상수가 곱해지거나 나눠지는 경우는 치지 않아서 그 복잡도는 n 이라고 표현한다. (  O(N)) 이렇게.

이것을  빅오 표기법의라 한다.
n개의 수에 대해서 상수 배수 로 증가하면 n차
제곱 단위로 증가하면  n제곱,

아주 큰 수를 줬을때 어떤 그래프를 따라가느냐 이다.
1차는 주는만큼 걸리는 시간이 비래-------> 효율적인 알고리즘  
다차, 팩토리얼이 되서 주는만큼  걸리는 시간이 기하급수적으로 증가하면 ------> 비효율 적인 알고리즘.

대부분의 검색 알고리즘. log n 정도 로 되어애 적당하다.(미리 정렬해 놓고 검색해서 그정도가 가능하다한다.)




![빅오 그래프](images/BIGO_GRAPH.jpg)






<br><br><br>

>**선택정렬(Selection sort)**  
* [9, 1, 6, 8, 4, 3, 2, 0, 5, 7] 를 정렬한다.  
* 정렬과정  
 * 리스트 중 최소값을 검색  
 * 그 값을 맨 앞의 값과 교체  
 * 나머지 리스트에서 위의 과정을 반복  
* 해결방법  
 * 알고리즘 진행과정 그려보기  
 * 의사코드(Psuedo code) 작성  
 * 실제 코드 작성


의사 코드로 나타내면 다음과 같다.
```bash
for i = 0 to n:
    a[i]부터 a[n - 1]까지 차례로 비교하여 가장 작은 값이 a[j]에 있다고 하자.
    a[i]와 a[j]의 값을 서로 맞바꾼다.
```

<br><br>
내가 혼자해본것
```bash

def sort_func(list11):
    def minmin(list11):
        min_num=list11[0]
        for i in list11:
            if i<min_num:
                min_num=i
        return min_num

    list22=[]
    for i in range(len(list11)):
        min_num=minmin(list11)
        del list11[list11.index(min_num)]
        list22.append(min_num)
    return list22


```
리스트가 너무 난잡하게 많다.

<br><br>

강사님과 같이 해봄.리스트 한개만쓰고, 슬라이스를 사용해서.  
다음과 같은 순서로 했다.  
1. *가장먼저 루프 돌때 마다 리스트를 슬라이스 해서 앞의길이를 잘라 버리게함.*  
2. *남은 리스트에서 가장 작은 수를 뽑음.*  
3. *가장 작은 수에 있던 수를 리스트의 맨앞으로 옮기고 그 맨앞에 있던 수를 가장 작은 수가 있던 자리로 옮긴. 이때 리스트는 원본 리스트가 아니라 남은 리스트이다.*



일단은 나혼자서 강사님 방법대로 슬라이스 써서해봄.

```bash

def sort_func(list1):
    for i in range(len(list1)):
        print(list1)
        print(list1[i:])

        min_n=list1[i:][0]
        print('min_n_B:' , min_n)
        fin_index=0
        #이걸 안해줘서 오류가 생겼었다. 파이썬 특성상 for 루프를 나와도
        #루프를 카운터 하던 지역변수 fin_index가 살아있다. 그래서 만약 아래의 if문 안에 들어가지
        #어기지 않아서 fin index에 새로운 카운터에 해당하는 값이 할당되지 않으면 이전 루프에 있던
        #인덱스값이 맨아레의 리스트 [i+j] 값을 참조하는데 들어가서 문제를 일으킨다.
        #보통은 이런일이 일어나지 않지만 슬라이스한 리스트의 맨앞에 최소값이 있을때
        #최소값<최소값 이라서 if문 자체에 한번도 들어가지 않게되면서 발생한다.



        for j in range(len(list1[i:])):
            if list1[i:][j]<min_n:
                min_n=list1[i:][j]
                fin_index=j

        print('fin_index: ',fin_index)
        print('min_n_A :' , min_n)
        print('*'*20)
        list1[i],list1[i+fin_index]=list1[i+fin_index],list1[i]


    return(list1)

        #알게된 사실---->list1[i:][0],list1[i:][fin_index]=list1[i:][fin_index],list1[i:][0]
        #이렇게 하면 값 할당이 서로 안된다. 주는건되는데 받는게 안됨.

```

결과

```bash
[9, 1, 3, 2, 5, 6, 8, 7, 4]
[9, 1, 3, 2, 5, 6, 8, 7, 4]
min_n_B: 9
fin_index:  1
min_n_A : 1
********************
[1, 9, 3, 2, 5, 6, 8, 7, 4]
[9, 3, 2, 5, 6, 8, 7, 4]
min_n_B: 9
fin_index:  2
min_n_A : 2
********************
[1, 2, 3, 9, 5, 6, 8, 7, 4]
[3, 9, 5, 6, 8, 7, 4]
min_n_B: 3
fin_index:  0
min_n_A : 3
********************
[1, 2, 3, 9, 5, 6, 8, 7, 4]
[9, 5, 6, 8, 7, 4]
min_n_B: 9
fin_index:  5
min_n_A : 4
********************
[1, 2, 3, 4, 5, 6, 8, 7, 9]
[5, 6, 8, 7, 9]
min_n_B: 5
fin_index:  0
min_n_A : 5
********************
[1, 2, 3, 4, 5, 6, 8, 7, 9]
[6, 8, 7, 9]
min_n_B: 6
fin_index:  0
min_n_A : 6
********************
[1, 2, 3, 4, 5, 6, 8, 7, 9]
[8, 7, 9]
min_n_B: 8
fin_index:  1
min_n_A : 7
********************
[1, 2, 3, 4, 5, 6, 7, 8, 9]
[8, 9]
min_n_B: 8
fin_index:  0
min_n_A : 8
********************
[1, 2, 3, 4, 5, 6, 7, 8, 9]
[9]
min_n_B: 9
fin_index:  0
min_n_A : 9
********************


```




<br><br>

위키에서 처럼 범위를 멋있게 지정해서 해봄.

```bash
def sort_func(list1):

    for i in range(len(list1)):
        min_index=i
        for j in range(i,len(list1)):
            if list1[j]<list1[min_index]:
                min_index=j

        list1[i],list1[min_index]=list1[min_index],list1[i]

    return list1

```

<br><br>

강사님 한것 처럼 다시 깔끔하게 해봄.

```bash

oriori  ==  [[99,,11,,66,,88,,44,,33,,22,,00]]
 # min함수 쓰지말고 각 Loop의 리스트에서 가장 작은값을 출력# min함수 쓰
ori_length = len(ori)
for i in range(ori_length - 1):
    min_index = i
    for inner_index in range(i, ori_length):
        if ori[inner_index] < ori[min_index]:
            min_index = inner_index
    if min_index != i:
        ori[i], ori[min_index] = ori[min_index], ori[i]
print(ori)

```


<br><br>

고찰.  


이 선택 정렬이 시간이 얼마나 걸리나?
총 8개가 있다고 하자 [][][][][][][][]
총 7번의 정렬
첫번째 루프에선7번 비교~~~~1번 비교.

임의의 수 n 개의 수 있다고 할때
n-1~~~~~~1번 비교

총 (n-1+1)*n/2 번 비교.------------>n^2형
