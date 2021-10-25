---
title: '2021 Software Engineer Interview'
date: 2021-10-01 12:46:28
urlname:
tags: 
  - Interview
---

## About This Article

* Python 3
* For someone who wants to use Python for interviews/leetcode
* For someone who knows a bit of Python and familiar with Java

## Language

```python
a, b, c = True, False, None

if b:
  print("hello1")
elif b is None:
  print("hello2")
else:
  print("hello3")

while a:
  print("a")
  a = False
print(6/3) # 2.0
print(6//3) # 2
```

## String

```python
message = "hello"
print(len(message))
```

## Iterate array, for loop

```python
nums = [10, 20, 30]
nums.append(40)

for num in nums:
  print(num)

for index, num in enumerate(nums):
  print(index, num)

for i in range(0, 5):
  print(i) # 0, 1, 2, 3, 4

for i in range(0, 5, 2):
  print(i) # 0, 2, 4
```

## HashMap

```python
room_num = {'john': 100, 'tom': 200}
room_num['john'] = 201

print(room_num['tom'])
print(room_num.keys())

if 'john' in room_num:
  print('john in room')

for key in room_num:
  print(key)

for key, value in room_num.items():
  print key, value
```

## Set

```python
num_set = set()
num_set.add(1)
num_set.add(2)

print(2 in num_set)
print(3 in num_set)
```

## Queue

```python
from collections import deque

nums = deque()
nums.append(1)
nums.append(2)

print(nums.popleft())
print(nums.popleft())
```

## Stack

```python
stack = []

stack.append('a')
stack.append('b')
stack.append('c')

print(stack.pop())
print(stack.pop())
```

## PriorityQueue

## Binary Search

## Breath First Search

## Depth First Search
