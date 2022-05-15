---
title: "Python Dictionary"
date: 2022-05-15 23:00:00 +0900
categories: debugging
tags: python c
---
## 들어가며
'Hash table' 자료구조에 대해 공부할 때, python에서는 dictionary가 있기 때문에 별도의 hash table을 구현할 필요가 없다는 얘기를 많이 들었다.

key-value 쌍으로 이루어진 dictionary는 hashing 과정을 알아서 진행해주고, bucket의 크기도 필요할 때마다 알아서 넓혀주기 때문에 사용하기 정말 편할 뿐아니라 직관적이다.

다만 hash table을 너무 편하게 쓰다보니, hash table에 대해 설명을 부탁하면 dictionary 구현방식이 아닌 dict 그자체의 특징을 설명하는 것을 종종 보게된다.

본인 또한 개념적으로 설명할 때 혼란을 겪었었고, 실제로 어떻게 구현이 되어 있길래 dictionary가 hash table을 구현한 것이라 말할까 궁금증이 생겼다.

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

## 어떤 code를 debugging해야 하는가
```python
from sys import getsizeof


a = {}
print(getsizeof(a))  # 64
a['a'] = 1
print(getsizeof(a))  # 232
a['b'] = 2
print(getsizeof(a))  # 232
a['c'] = 3
print(getsizeof(a))  # 232
a['d'] = 4
print(getsizeof(a))  # 232
a['e'] = 5
print(getsizeof(a))  # 232
a['f'] = 6
print(getsizeof(a))  # 360
```
위 code를 통해 언제 dictionary의 크기가 변하는지 확인할 수 있었다. 본인은 dictonary의 크기가 변할 때의 code를 확인하기 위해 아래와 같은 code를 debugging 하였다.
```python
def foo():
    a = {'a':1, 'b': 2, 'c': 3, 'd': 4, 'e': 5}
    a['f'] = 6
```
함수로 만들면 아래와 같이 bytecode를 확인할 수 있다.
```python
>>> from dis import dis
>>> dis(foo)
#   2           0 LOAD_CONST               1 (1)
#               2 LOAD_CONST               2 (2)
#               4 LOAD_CONST               3 (3)
#               6 LOAD_CONST               4 (4)
#               8 LOAD_CONST               5 (5)
#              10 LOAD_CONST               6 (('a', 'b', 'c', 'd', 'e'))
#              12 BUILD_CONST_KEY_MAP      5
#              14 STORE_FAST               0 (a)

#   3          16 LOAD_CONST               7 (6)
#              18 LOAD_FAST                0 (a)
#              20 LOAD_CONST               8 ('f')
#              22 STORE_SUBSCR
#              24 LOAD_CONST               0 (None)
#              26 RETURN_VALUE
```
위의 bytecode를 통해 `a['f'] = 6`으로 dictionary에 값이 저장되는 부분은 `STORE_SUBSCR`을 확인해보면 되는 것을 확인할 수 있다.
```c
1848    switch (opcode) {
        ...
2365    case TARGET(STORE_SUBSCR): {
2366        PyObject *sub = TOP();
2367        PyObject *container = SECOND();
2368        PyObject *v = THIRD();
2369        int err;
2370        STACK_SHRINK(3);
2371        /* container[sub] = v */
2372        err = PyObject_SetItem(container, sub, v);
2373        Py_DECREF(v);
2374        Py_DECREF(container);
2375        Py_DECREF(sub);
2376        if (err != 0)
2377            goto error;
2378        DISPATCH();
        ...
```

## debugging
조금전 알게된 bytecode의 시작 부분에 breakpoint를 설정한다.
`./Python/ceval.c` file에서 bytecode를 검색하면 해당 bytecode의 case문을 확인할 수 있다.
몇 번째 줄인지 확인하면 해당 줄에 breakpoint를 설정해야한다.
이를 위해 다른 terminal 창을 띄워 현재 lldb로 실행중인 python을 잠시 멈추고 lldb로 빠져 나와야 한다.
```sh
ps  # 중지시켜야 할 process의 PID 확인
kill -5 $PID  # process에 SIGTRAP 전달
```

SIGTRAP signal을 받은 debugger는 실행중이던 python이 멈추고 lldb로 전환된다. 위에서 알게된 `STORE_SUBSCR`의 시작점에 breakpoint를 설정하고 다시 python interpreter를 실행한다.
```sh
(lldb) b ceval.c:2366
(lldb) c
```
이후 아까 정의했던 foo()함수를 호출하면 `STORE_SUBSCR`의 시작 부분에서 breakpoint가 작동한다.
```python
>>> foo()
```
```c
err = PyObject_SetItem(container, sub, v);  // ceval.c:2372
int res = m->mp_ass_subscript(o, key, value);  // abstract.c:210
return PyDict_SetItem((PyObject *)mp, v, w);  // dictobject.c:2225
return insertdict(mp, key, hash, value);  // dictobject.c:1623
```
위와 같은 코드를 따라가다 보면 찾아보려던 핵심 코드에 도달한다.
```c
1081    Py_ssize_t ix = mp->ma_keys->dk_lookup(mp, key, hash, &old_value);  
...
1098    if (ix == DKIX_EMPTY) {
1099        /* Insert into new slot. */
1100        assert(old_value == NULL);
1101        if (mp->ma_keys->dk_usable <= 0) {  // table에 저장할 수 있는 공간의 개수를 나타내는 dk_usable을 확인
1102            /* Need to resize. */
1103            if (insertion_resize(mp) < 0)  // 저장 가능 공간이 없을 경우 table의 size를 insertion_resize()를 통해 늘림
1104                goto Fail;
1105        }
```
```c
460     #define GROWTH_RATE(d) ((d)->ma_used*3)  // 사용중 공간의 2배만큼을 추가로 늘림
...
1060    return dictresize(mp, calculate_keysize(GROWTH_RATE(mp)));
...
1241    /* Allocate a new table. */
1242    mp->ma_keys = new_keys_object(newsize);  // 새 table 생성
1243    if (mp->ma_keys == NULL) {
1244        mp->ma_keys = oldkeys;
1245        return -1;
1246    }
1247    // New table must be large enough.
1248    assert(mp->ma_keys->dk_usable >= mp->ma_used);
1249    if (oldkeys->dk_lookup == lookdict)
1250        mp->ma_keys->dk_lookup = lookdict;
1251
1252    numentries = mp->ma_used;  // 
1253    oldentries = DK_ENTRIES(oldkeys);
1254    newentries = DK_ENTRIES(mp->ma_keys);
1255    oldvalues = mp->ma_values;
...
1277    else {  // combined table.
1278        if (oldkeys->dk_nentries == numentries) {
1279            memcpy(newentries, oldentries, numentries * sizeof(PyDictKeyEntry));  // 이전 table에서 크기가 증가된 새 table로 내용 복사
1280        }
...
1310    build_indices(mp->ma_keys, newentries, numentries);
1311    mp->ma_keys->dk_usable -= numentries;
1312    mp->ma_keys->dk_nentries = numentries;
```
위에서 볼 수 있듯이 table의 크기를 늘려야 할 때는 새로운 크기를 늘린 table을 생성하고, 기존 데이터를 복사하는 작업을 진행한다.
```c
135     #define PERTURB_SHIFT 5
...
299     #define DK_SIZE(dk) ((dk)->dk_size)
...
1191    /*
1192    Internal routine used by dictresize() to build a hashtable of entries.
1193    */
1194    static void
1195    build_indices(PyDictKeysObject *keys, PyDictKeyEntry *ep, Py_ssize_t n)
1196    {
1197        size_t mask = (size_t)DK_SIZE(keys) - 1;
1198        for (Py_ssize_t ix = 0; ix != n; ix++, ep++) {
1199            Py_hash_t hash = ep->me_hash;
1200            size_t i = hash & mask;
1201            for (size_t perturb = hash; dictkeys_get_index(keys, i) != DKIX_EMPTY;) {
1202                perturb >>= PERTURB_SHIFT;
1203                i = mask & (i*5 + perturb + 1);
1204            }
1205            dictkeys_set_index(keys, i, ix);
1206        }
1207   }
```
mask 값은 총 크기의 -1 한 값으로 지정된다. -1을 하는 이유는 2진법으로 치환하면 알기 쉽다.
예를 들어 dk_size = 16일 경우, 2진법으로 표현하면 10000(2)이다.
여기서 -1을 하게 되면, 01111(2)가 되고, 0000 ~ 1111까지(0 ~ 15) 총 16개의 숫자를 표현할 수 있게된다.
1200번 line을 확인해보면 hash값에 mask를 AND 연산하여 i(index)를 계산한다.
이후 1201번 line의 for문을 통해 dict[i]가 비어있는지 확인하고, 비어있지 않으면 i 값을 다시 계산하여 비어있는 dict[i]를 찾아 저장한다.
python에서는 hash table의 collision 해결방법으로 open addressing을 사용하고 있음을 확인할 수 있다.
```c
849     /* Specialized version for string-only keys */
850     static Py_ssize_t _Py_HOT_FUNCTION
851     lookdict_unicode(PyDictObject *mp, PyObject *key,
852                      Py_hash_t hash, PyObject **value_addr)
853     {
...
863         PyDictKeyEntry *ep0 = DK_ENTRIES(mp->ma_keys);
864         size_t mask = DK_MASK(mp->ma_keys);
865         size_t perturb = (size_t)hash;
866         size_t i = (size_t)hash & mask;
867
868         for (;;) {
869             Py_ssize_t ix = dictkeys_get_index(mp->ma_keys, i);
...
874             if (ix >= 0) {
875                 PyDictKeyEntry *ep = &ep0[ix];
...
878                 if (ep->me_key == key ||
879                         (ep->me_hash == hash && unicode_eq(ep->me_key, key))) {
880                     *value_addr = ep->me_value;
881                     return ix;
882                 }
883             }
884             perturb >>= PERTURB_SHIFT;
885             i = mask & (i*5 + perturb + 1);
886         }
887         Py_UNREACHABLE();
888     }
```
dict의 조회 시 사용되는 함수 중 하나인 lookdict_unicode의 878 ~ 879와 884 ~ 885 line을 확인해보면 저장된 key와 찾으려는 key가 동일한지 확인하고, 아닐 경우 저장할 때와 동일한 로직으로 index를 다시 계산하는 것을 볼 수 있다.

## References
- [CPython Github](https://github.com/python/cpython)
- [How To Step Through The CPython Interpreter](https://medium.com/@skabbass1/how-to-step-through-the-cpython-interpreter-2337da8a47ba)
- [Python Hash Tables: Understanding Dictionaries](https://thepythoncorner.com/posts/2020-08-21-hash-tables-understanding-dictionaries/)
- [More compact dictionaries with faster iteration](https://mail.python.org/pipermail/python-dev/2012-December/123028.html)
