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
