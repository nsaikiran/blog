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
> a = [1,2,3] 

>  a.append(a)

> print(a)

Python's [copy](https://docs.python.org/3/library/copy.html) module highlights this problem. 

We can use `id()` to get unique identifier for each object and traverse all its contained objects and mark if we have visited them already or not.