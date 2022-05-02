---
title: "Python Dictionary"
date: 2022-05-02 21:00:00 +0900
categories: debugging
tags: python c
---
## 들어가며
'Hash table' 자료구조에 대해 공부할 때, python에서는 dictionary가 있기 때문에 별도의 hash table을 구현할 필요가 없다는 얘기를 많이 들었다.

key-value 쌍으로 이루어진 dictionary는 hashing 과정을 알아서 진행해주고, bucket의 크기도 필요할 때마다 알아서 넓혀주기 때문에 사용하기 정말 편할 뿐아니라 직관적이다.

다만 hash table을 너무 편하게 쓰다보니, hash table에 대해 설명을 부탁하면 dictionary의 특징을 설명하는 것을 종종 보게된다.

나 또한 개념적으로 설명할 때 혼란을 겪었었고, 실제로 어떻게 구현이 되어 있길래 dictionary가 hash table을 구현한 것이라 말할까 궁금증이 생겼다.

## debugging 준비
먼저 [python.org](https://www.python.org) 또는 github에서 분석할 source code를 받는다.
```sh
# github에서 cpython source 받기
git clone https://github.com/python/cpython.git

# folder 이동
cd cpython

# 3.10 version으로 checkout
git checkout 3.10
```
아래 명령어로 debugging용 `python.exe` file을 생성한다.
```sh
./configure --with-pydebug
make
```

본인은 mac을 사용하기 때문에 `gdb` 대신 `lldb`를 사용해 debugging을 진행하였다.
```sh
# 위에서 build한 python.exe를 lldb parameter로 넘겨줌
lldb python.exe

# lldb에서 python.exe 실행
(lldb) r
```

## 
```python
from sys import getsizeof


```


## 


## References
- [CPython Github](https://github.com/python/cpython)
- [How To Step Through The CPython Interpreter](https://medium.com/@skabbass1/how-to-step-through-the-cpython-interpreter-2337da8a47ba)
- [Python Hash Tables: Understanding Dictionaries](https://thepythoncorner.com/posts/2020-08-21-hash-tables-understanding-dictionaries/)
