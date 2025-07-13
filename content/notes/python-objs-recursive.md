---
layout: post
title:  "Pythons objects can be recursive"
description: "Pythons objects can be recursive"
categories: ["tech"]
date: 2024-05-11 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

Python objects can be recursive! for example, to a list you can add itself as an element.

> NOTE: In an interview I was asked to serialize a given python object. The object can be any valid type in python, for ex: list, dict, tuple, set etc and it can also be a primitive type. In that I was asked to identify if do you prevent infinite recursion if the object is added to itself somewhere inside.

For example,
```python
a = [1,2,3] 
a.append(a)
print(a)
```

Python's [copy](https://docs.python.org/3/library/copy.html) module highlights this problem. 
To find recursive objects and prevent processing them, we can keep track of the id's of the objects. Python objects will have unique id.

```python
def has_recursion(obj, seen=None):
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return True
    seen.add(obj_id)

    if isinstance(obj, dict):
        for k, v in obj.items():
            if has_recursion(k, seen) or has_recursion(v, seen):
                return True
    elif isinstance(obj, (list, tuple, set)):
        for item in obj:
            if has_recursion(item, seen):
                return True

    seen.remove(obj_id)
    return False

# Example
a = []
a.append(a)

print(has_recursion(a))  # True

b = [1, 2, 3, [4, 5]]
print(has_recursion(b))  # False
```

We can't use `==` because it checks for elements in the objects to be equal which is not we wanted, we wanted if two objects are referring to the same memory location.

```python
a = [1, 2]
b = [1, 2]
print(a == b)  # True, same *values*
```

```python
a = []
a.append(a)

print(a == a)  # Fine
print(a == [a])  # Also fine, same structure

b = [a]
# If you compare a and b deeply, equality would loop forever.
```