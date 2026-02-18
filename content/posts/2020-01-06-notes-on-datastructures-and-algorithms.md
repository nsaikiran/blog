---
layout: post
title:  "Notes on data structures and algorithms"
description: "My notes on fundamentals of data strucures and algorithms"
categories: ["cs"]
tags: ["data-structures", "algorithms", "data structure"]
date: 2020-01-06 19:45:31 +0530
author: "Sai kiran"
url: /cs/2020/01/06/notes-on-datastructures-and-algorithms.html
aliases:
  - /cs/2020/01/06/notes-on-datastructures-and-algorithms.html
---

## Introduction
<!-- (why DS and Algorithms) Rename this?-->
Computers are ubiquitous. They help us solving so many problems and for many people they are already integral part of daily life. We expect them to be fast and use hardware resources effectively. We can achieve these goals through carefully crafted software.

Computers are dumb and they have only two fundamental capabilities: following instructions precisely and storing data. They don't actually solve the problems, we have to  solve the problem and provide them with steps of solution. Those steps of solution is called an algoritm. As the algorithm is written for computer, we should have a good understanding of how computers work.

As we want to have an efficient solution/algorithm, we are interested in time complexity and space complexity of the solution.

Unless the solution is just a mathematical expression (For example, calculate sum of consecutive natural numbers upto given number), it operates on the data that is specified in(or derived from) the problem. For the sake of understanding let us assume that the data is a group of objects, then the basic operations on the group will be: add more objects, delete objects, updating an object, simple exploration/traversal of objects and more importantly search for an object. As computer's memory is a gaint array of bytes, we need to first solve those operations(on data) as needed by the solution. _Data Structure_ is a concept where we study various ways to organize and operate on the data so that we've an idea of time complexity or space complexity of various operations we will need. As solution to data operations is an important part of the solution itself, the _data structure_ we choose affects time and space complexities of solution. 

<!-- Hence, figueringout the data structure will be the first step of solution. (Data structure is part of the solution) -->

We can produce many solutions/algorithms to a problem, but we want the best of those in terms of runtime and space complexity. To do that, we need to _compare_ those solutions. Using a real computer to calculate runtime and space complexity for each is a very tedious task, and all computers don't have same capabilities. Hece, to simplify the complexity calculation, to make a common ground for benchmarking algorithms we choose a theoritical computer &mdash; _the RAM model of computation_.

<!-- (because the real compters are complex. Exlain trait of problem with respect to solving using computers.) -->

## Random Access Machine

RAM is a simpliefied computer: it has single procesor, no cache, each basic instruction take constant time.
Get to know more about RAM [here](https://www8.cs.umu.se/kurser/TDBA77/VT06/algorithms/BOOK/BOOK/NODE12.HTM#SECTION02131000000000000000) and [here](https://www.cse.cuhk.edu.hk/~taoyf/course/comp3506/lec/ram.pdf).

But I'm also listing properties here:

- Each simple operation (+, *, -, =, if, call) takes exactly 1 time step.
- Loops and subroutines are not considered simple operations. Instead, they are the composition of many single-step operations. It makes no sense for _sort_ to be a single-step operation, since sorting 1,000,000 items will take much longer than sorting 10 items. The time it takes to run through a loop or execute a subprogram depends upon the number of loop iterations or the specific nature of the subprogram.  
- Each memory access takes exactly one time step, and we have as much memory as we need. The RAM model takes no notice of whether an item is in cache or on the disk, which simplifies the analysis.

### Accounting on RAM

We calculate time and space complexity of various algorithms for a problem and compare them to pick a _relatively better_ algorithm. Those calculations are formulated interms of number of constant-time(low-level) operations needed on RAM(for some amount of data).

TODO: worst-case, average-case and best case analysis somewhere

### Asymptotic analysis

Sometimes we are interested in solving the problem assuming that we are working on _large amount of input data_, because these days computers are operating on huge volumes of data.
If that is the case, we use _asymptotic analysis_ to simplify the complexity calculation of an algorithm. Once we've formulated the complexities, as part of _asymptotic analysis_ we simplify the formulation assuming that the input is very large, i.e. close to infinify. With this, our process of selecting a relatively better algorithm will be easier.

The analysis made for large inputs might not be suitable for small inputs (For ex: the insertion sort is better for small inputs than merge sort which asymptoticallly better than insertion sort).

Refer: [Big-O Notation](https://www.cs.cmu.edu/~clo/www/CMU/DataStructures/Lessons//lesson9_1.htm)

## Data structures

As discussed, solutions that just need numerical computation may not any data structure per se. For example, calculating GCD, Check if given number is prime number or not? etc. But many other solutions do need data structures. The memory of RAM, is a gaint array of memory locations which can be _randomly_ accessed. We store/organize the data in it and operate.
<!-- (More info?).  -->

### Fundamental ways to organize data in memory

Fundamentally we can store the data contigously (as an array of objects) or non-contigous/linked (linked objects).

#### Array

As the objects stored as an array are stored contigously, randomly accessing an object based on its index takes constant time. But insert/delete operations will take more work. We also need to know the no. of objects upfront to be able to allocate the required memory. 

In an array, objects are stored contigously. As an array objects are of same type and are stored contigously, is very easy to index into an array -- you can always calculate the position of object if you know the index of the object in that array. But inserting into an array would be a costlier operation, because to make room for the extra object. Array is suitable if we the data is of fixed size known at the time of allocation and we perform indexing operation often.

#### Linked objects

But if we want to be able to insert/delete objects often or we don't know the size of data upfront. We can go for Linked objects, where objects are linked to each other. In this objects may not be stored contigously. As the physical location of the objects are not evident, we can't index into Linked objects as quickly we did in an array. Each object stores position of object(s) that can be reached. Eamples: Single linked list, double-linked list.

Where do we place the data structure: either stack area or heap area depends on the life time of the data(function scope or program scope) and the size of data as well(stack will be limited).

Read:

- [What’s a Linked List, Anyway? [Part 1]](https://medium.com/basecs/whats-a-linked-list-anyway-part-1-d8b7e6508b9d)

Operations on linked-list:

- Can't index, linear traversal is required. Even if the linked list is sorted finding an object will take linear time.
- Binary seach won't work, linked list can be augmented to be tranformed into a skip list to get same effect of binary search.
- Modified mergesort suits best for sorting.
- Using fast and slow pointer, we can find, middle of linked list, k th last element etc.

### Importance of relations

Now, imagine we've an array of integers and our task is to check if a given integer exists in our array. Here, we need to find the given integer in the array. It costs us time that is propotional to the size of the array. (Consider we do this opeartion very often) But how can we reduce this? We sort the array. Interestingly when we sort the array in ascending order, we've established correlation between locality of the integer with its value &mdash; an integer is located after integers that are less than this. Using this correlaation we perform binary search. The same principle applies to binary search trees where all keys that are lesser will be stored in its left side. We know the concept of _Height Balanced Binary Search Trees_, which provide us find operation in logarithmic of input even in worst case. But we also spend some extra time to balance the tree, right after a change is done on the tree(which may be ignored if changes to the tree is lesser compared to read/find operations). 

I would highly recommend watching [Sean Parent "Better Code: Data Structures"](https://www.youtube.com/watch?v=sWgDk-o-6ZE), which helped me concretise this idea.

Other examples

- _Hashing_: where we bring correlation between representaton of object and its location. 
- _Heap invariant_: In heap, parent and children are related: greater or lesser.

<!-- (Describe?) -->

TODO: Try to give more examples.

## Abstract data types

For solving the problem, we first need to decide the operations on objects. The theoretical definition of required operations is called _an Abstract Data Type_. Type of the data describes operations allowed on the data. Because ADT don't have implementation, it is called as _abstract_. For a given ADT, we try to implement data structures that supports those operations; we compare them and pick the one with a reasonable amount of complexity.
Some common ADTs that may be incorporated into the solution are [Dynamic array, Stack, Queue](https://web.stanford.edu/class/archive/cs/cs106b/cs106b.1186/lectures/05-Stacks_Queues/5-Stacks_Queues.pdf), [Circuar queue][Circular-queue-implementation], Priority queue, Graph, Min-Max-heap, Map(of a key-value pair), and [Union-Find](https://sedgewick.io/wp-content/uploads/2022/04/Algs01-UnionFind.pdf) etc. It is very rare that you'll have to implement ADT yourself. You may have to implement ADT yourself only when you feel the availble implementaion is not suitable for your use case or you've not found any implemention that suits your need.
Sometimes, well implemented ADTs may be built-into the programming language you work on or can be used from a library. It is worth knowing various properties of the _readily availble implementsions_ before using them in your particular case.

### List

`list` is an example of dynamic array or variable sized array. Variable sized array can be implemented as linked objects as well as contigously stored objects.  If indexing operation is required then Variable sized array should be implemented with contigously stored objects.  But it is little tricky to implement Variable sized array with contigously stored objects. In [this lecture](https://www.youtube.com/watch?v=BRO7mVIFt08) Prof. Erik Demaine explains implementing Variable sized array as contigously stored objects using Table Doubling. [Python's `list` data type is backed by table doubling implementation](https://docs.python.org/3/faq/design.html#how-are-lists-implemented-in-cpython) where as [`Deque` is implemented as double-linked data objects](https://github.com/python/cpython/blob/v3.8.1/Modules/_collectionsmodule.c#L33). Both can grow to variable length, but Deque can do insert and delete operations on both the sides effectively. And, [python's `deque` container can be used as a Circular Queue aswell.](https://docs.python.org/3/library/collections.html#deque-objects)

Python's [list](https://docs.python.org/3/faq/design.html#how-are-lists-implemented-in-cpython) can be used as both Stack and Queue. But, [`list` is not an optimal option to implement Queue](https://docs.python.org/3/tutorial/datastructures.html#using-lists-as-queues) as queue will take insertions at the beginning. [Deque](https://docs.python.org/2/library/collections.html#collections.deque) from Python's collections library is more suitable as Queue.

### Queue

FIFO is the removal strategy.
Read:

- [To Queue Or Not To Queue](https://medium.com/basecs/to-queue-or-not-to-queue-2653bcde5b04)
- Queue can be implemented with both array and linked list. Lined list based implementation is easy to do. Because dequeue will create empty slots at the front of the list.
- [Circular queue or ring buffer can be thought of a bounded queue, and when this is implemented using array, each deque operation will create empty slots at the front of the array. We need to use these empty slots to enqueue items further, hence we can see circular usage.][Circular-queue-implementation].
  - We can use ring buffer implementation to track _most recent_ elements in the stream, for example, given a stream of data, to perform computation ( _max_/_sum_) on most recent K elements.

### Stack

LIFO is removal strategy.
Example problem that uses stack datastructure:

- [Find immediante next greater element for each element in an array](https://github.com/nsaikiran/MyPrograms/blob/master/Python/interview-prep/nge.py)
- [Given a string print it in reverse]() TODO
- [Check for valid paranthesis]() TODO
- [Backtracking, Browser tab's next and previous navigation and undo-redo on a document] () TODO
Read:

- [Stacks and Overflows](https://medium.com/basecs/stacks-and-overflows-dbcf7854dc67)
- Know both array based and linked list based implementation.

### Map

Map datastructure store _key and value_ pairs to provide efficient look-ups. For example, a key to value _map_ can be implemented with hashing which is called as [HashMap](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html) (in Python it is Dictionary) or with a balanced BST which can be called as [TreeMap](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html). [LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html) with a doubley linked list. These different implementations have different characteristics.

- HashMap provides faster access and can't provide _natural_/_sorted_ ordering of the keys. No order.
- TreeMap provides O(log N) operations and provides _natural_/_sorted_ ordering of the keys. Natural order.
- [LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html) remembers the insertion order.

#### Examples

- [LRU cache implementation](https://www.enjoyalgorithms.com/blog/implement-least-recently-used-cache) using HashMap and DLL
  - HashMap will have the mapping of key and the objects location in a double-linked-list of the objects. We use linked list of objects to sort the objects based on "recently access", the "least recently used" object will be at the end of the linked list. Whenever we insert/update a key's value, we move that object to the beginning. When we needed to evict a key to insert a new key, we delete the last element from linked list (which is least recently used).
  - We could also use a single linked list here, but the implementation will be tricky. The main reason for double linked list is to facilitate the easy deletion of object. So, [check how we can replace that with single  linked list](https://stackoverflow.com/questions/49621983/lru-cache-with-a-singly-linked-list)
- [LFU cache implementation - ADT](https://arpitbhayani.me/blogs/lfu/) using HashMap and DLL.
  - This cache will evict the "least frequently used" element when it has to insert a new key-value pair and its limit is reached.
  - Usually this is implemented with "min-heap" in which objects are stored based on their "access count", but this provides O(log N) complexity. O(1) solution is discussed, using double linked list and HashMap, go through this.

### Set

[A set of items can be implemented](https://docs.oracle.com/javase/tutorial/collections/implementations/set.html) with Hashing, which is [HashSet](https://docs.oracle.com/javase/8/docs/api/java/util/HashSet.html) or a balanced BST, which is [TreeSet](https://docs.oracle.com/javase/8/docs/api/java/util/TreeSet.html), or a linked list, which is [LinkedHashSet](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashSet.html). The characteristics of HashSet, TreeSet and LinkedHashSet are different. HashSet doesn't remember any order, where as TreeSet and LinkedHashSet remember order of elements. Along with set functionality, if we need an iterator of elements to traverse them in the natural order then TreeSet can be used, if we need an iterator to traverse in the order of insertion then LinkedHashSet (ADT contains set and double linked list) can be used. Also, the timecomplexities may vary. HasSet and LinkedHashSet provides O(1) but TreeSet provides O(log N) for all basic operations like find, insert and delete.

Difference between Map and Set is elements of set will not have associated values but map's elements will have associate members. In Map values are _indexed_ by the key like elements in array are indexed by their position.

### Redis Sorted Set

Let us say we've below requirements:

- We've a key value pair, we want to get the value of given key: HasMap functionality (Or dict in python).
- We want the ordering of elements based on their _value_. To answer queries like, rank of an element, top X elements based on value and update the value as and when required.

Example could be a leader board. Given a player you want to know their score. Who are top 10 players, what is the rank of a particular user.

This ADT is interesting and is provided in Redis, this isn't provided in Python or Java but we can implement. This ADT can be implemented using a HasMap and a [skip list](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/SkipList.html#). Here, skip list will have elements (key-value pairs) sorted using their _values_. Through skip list we can achieve _binary search_ like functionality in linked lists -- O(long N) for search. In this ADT, skip list can be augmented with extra information (like how many elements are skipped) to calculate the rank very quickly.

For more information refer [Sorted Set](https://jothipn.github.io/2023/04/07/redis-sorted-set.html). The name _sorted set_ seems counterintuitive but I also have no name for a HashMap which also maintains the sorted ordering based on values.

### Heap

When you want min/max or given elements. This data structure can be used in priority queue, and computing k min/max elements from a stream, can be thought of as stream processing.

Read:

- [Learning to Love Heaps by Vaidehi Joshi](https://medium.com/basecs/learning-to-love-heaps-cef2b273a238)
- [Python's heapq](https://docs.python.org/3/library/heapq.html)

### Priority Queue

Priority Queue can be implemented with [Heap data structure](https://en.wikipedia.org/wiki/Heap_(data_structure)).

### Graphs

Collections of nodes connected with edges
Graphs can be implemented with linked nodes or adjacency matrix (two-dimentional array) or adjacency list.
<!-- (array of pointers? to array) -->
Several real-world problems are converted/reduced to graph problems.

Refer striver's graph playlist, [full](https://youtube.com/playlist?list=PLgUwDviBIf0oE3gA41TKO2H5bHpPd7fzn&si=2emJOcUS1ziWCCR5), or [condensed](https://youtube.com/playlist?list=PLgUwDviBIf0rGEWe64KWas0Nryn7SCRWw&si=Z8bQYYw_VfsrLeMS), they are helpful.

### Trees

A connected acyclic graph is called a tree.
Explore trees: Height balanced binary search trees(AVL or Red black trees), B-trees etc

- in-order, pre-order and post-order traversal.
- depth-first traversal (recursion/backtracking/stack), breadth-first traversal (queue).

### Data structuers in programming lanauges

Many high level programming lanauges provide abstract data types built-in and other will have a library where we can pick up. For ex: Python, Javascript have common ADTs as built-in. For C++ and Java, you can use from their standard library. Maybe try to Understand the _memory model_ of each the language you are using.

In the above section, we've mentioned some datastructures from Java and some from Python. The goal is to understanding various implementations of ADTs, hence we've collated for different languages.

TODO: more in this section?

## Algorithms

As discussed, for producing algorithms for computers, we need good understanding on how computers work. Algorithm must be correct, and we should be able to check/prove the correctness before using it(For all problem instances it should produce correct result). There are techniques to prove that given algorithm is correct. Out of all correct algorithms (candidate algorithms), we pick the optimal one(in our context). So, we should always focus on producing correct algorithms, and then we can think about optimations.

Refer Chapter 1.1 of CLRS. (CLRS 3rd Ed)
<!-- what to check in this chapter -->

Some important points:

- Understanding _recursion_ is important, because for reusing/parameterizing the code, we break the code in functions. _Recursion_ is very useful in breaking down the problems into smaller problems of easy analysis.  
- Understanding the problem context and the input data very well can help producing algorithms that work best in that specific context. For example, if we are solving a sorting problem, if our data range is very small compared to the size of the data, we go for a counting sort. 

<!-- Give more exmaples for context based best things -->
### Various common problems and solutions/algorithms

Some well-known problems and their solutions are listed here. Examples here are not mentioned in above section.

#### Searching solutions

- [Binary Search](https://en.wikipedia.org/wiki/Binary_search_algorithm): This algorithm requires data to be sorted. This implementation works on arrays.
- [Skip lists](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/SkipList.html#): This algorithm can be understood as `binary search` on linked lists, where the nodes/elements are added dynamically.

### Counting

- Some problems can be reduced to counting problems. For example:
  - Sort array of numbers _that are bounded_, meaning range of number is very small butwe can sort these numbers by counting. When data is bounded.
  - [Find K-th Smallest Pair Distance](https://leetcode.com/problems/find-k-th-smallest-pair-distance/editorial/), refer [this explanation](https://www.youtube.com/watch?v=bQ-QcFKwsZc).Here, we are just counting the pair differences without actually computing them. This may not be intuitive at first.

#### Sorting solutions

- Insertion sort: in-place, _stable_, optimal for almost sorted data. If other properties of data are unknown, then suitable for small number of data items.
- [Quick sort](https://www.enjoyalgorithms.com/blog/quick-sort-algorithm): Example of divid and conquer approach. Can be thought of blanket sorting algorithm if no extra useful information about data is not present (like elements to sort are less or elements are bounded etc.). The worst case complexity can be o(n2), which happens when the data is sorted in opposite order. There are strategies to choose pivot, such as randomized selection, to achieve desired complexity, O(nlong). It is commonly used in practice. This is in-place.
- Merge sort: Example of divide and conquer, can be thought of blanket algorithm like quick sort. It is stable, guarantees O(nlog n), sorting linked lists and external sorting for big data. But 
- Heap sort: Using heap data structure. Pick root (which will max/min)
- Counting/bucket sort: Linear time sorting, data assumptions
- Lower bounds for searching or sorting.

### Sliding window

To do certain computations on linear datastrctures (arrays) in linear time.

- Maximum/Minimum subarray sum of size k
- Longest substring without repeating characters
- Anagrams in string,
- Moving average of stream etc

### Prefix sum/cumulative sum

Range sum queries

### Graph algorithms

<!---
problem, problem instance, solutions, candidate solution
Algorithm (for computer) is the steps of solution that a computer can perform. We can produce many *correct* algorithms for a problem, we call them *candidate algorithms*. And we pick one of those *candidate algorithms* that takes less time to produce result and(or or both) consumes less memory. 
-->

TODO: update this section

General problem solving techniques:

- Divide and conquer
- Greedy
- Dynamic programming
- etc
- Explain the relation among those.
- Sliding window technique

Give reference to examples of these:

- Recursion
- Back tracking 
- Search algorithms
- Sort algorithms
- Graph algorithms
  - Exploration algorithms
    - BFS, DFS
  - Shortest path algorithms (generic cases to specific cases (positive weighted, negative weighted, negative weighted with cycles))
  - [Union-find](https://sedgewick.io/wp-content/uploads/2022/04/Algs01-UnionFind.pdf) and Hacker rank union-find
  - Famous problems (TSP, Knapsack)

Talk about these terms:
- Invariant
- Correctness
- Reductions

### Computablility
Introduce Computatability of problem, NP-Completeness?

### Preperation for interviews
- [Top 5 Data Structures](https://www.youtube.com/watch?v=DDRo29ptFwE) and [Top 5 Algorithms](https://www.youtube.com/watch?v=EM8IgIIiOdY)
- [Several Coding Patterns for Solving Data Structures and Algorithms Problems during Interviews] (https://github.com/Chanda-Abdul/Several-Coding-Patterns-for-Solving-Data-Structures-and-Algorithms-Problems-during-Interviews)
- [Coding Interview University](https://github.com/jwasham/google-interview-university/blob/master/README.md#google-interview-university)

## References:
### Videos
- [MIT 6.006 Introduction to Algorithms, Fall 2011](https://www.youtube.com/playlist?list=PLUl4u3cNGP61Oq3tWYp6V_F-5jb5L2iHb)
- [Better Code series by Sean Parent](https://developer.adobe.com/cpp/training/)

### Textbooks
- CLRS
- Algorithms by by Robert Sedgewick and Kevin Wayne. [Refer slides for each topic](https://sedgewick.io/courses/algorithms/). In this link, go to slides section, useful content. Also, the related videos are hosted [here](https://www.cubits.ai/collections/117/)
- Algorithm Design Manual

## Further Reading:
- [What is Datastructure](https://web.archive.org/web/20230924100056/https://www.enjoyalgorithms.com/blog/introduction-to-data-structures)
- [Vaidehi Joshi's CS basics](https://medium.com/basecs)


[Circular-queue-implementation]: https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/Queue.html#the-circular-queue
<!---
==========================================================
EDITING NOTES:


(Should we mention that the number of basic operatons will be propotional to the real time, hence we need to minimize the memroy access/ or operations)

( We know memroy is a gaint array of bytes. For the ease of discussion, let us define, group of bytes to be an object. We work on  a collection of objects. WHat shall we do with collection? We may create an new object and add it to the collection, deleting an existing one, change an existing one and find/search for an object in the collection. These are the fundamental operations. Our goal is to organize these objects in memory such that the above mentioned operations can be perfomred quickly. This is the crucial part. Because the memroy is an array. How to store the objects and how to traverse the objects. To do that we think about relations, relation b/w objects: value relations or physical relations etc,. and see if we can bring correlation among such relations so that we can device a plan to organize and traverse the objects on memeory. And this organizatino should be suitable for our usecase. For example: Let us take an example of searching an object in an array. If we were give an array of objects, searhcing an object will take liner time. From the beginning, each object is stored next to the previsous object physically. But when we sort the objects we've established correlation between value and physical relation -- all elements lesser or equal to a given value will be stored left side and greater value objects will be stored right side. With this, we can search for a given object in a logarthmic time. )

(You may not be interested in all of those operations, because frequency of all those operations may not be same in your usecase. DS/Algo that is optimized for very frequent operations would be preferreable for example. 

Essentially you would learn as many DS as possible, and keep them in your toolbox. Understand your problem very well, interms what is the nature of data you would handle: sorted, radom, range can be bounded?, any duplicates etc.

Picture of toolbox and understanding of data. Would give better DS understanding.)



# Introduction
This is hyper-linked notes I took for DS.
Computers operates on data. By studying various DS, their strenghs and drawbacks helps us 
The generic idea of computer is that it (takes in some data and )operates on data. 
And to operate on data it takes certtain amount of time. When learning data structures our goal is to keep the time spent reasonable.
After all, we don't want to stare at screen till the problem is solved.

Computers have memory to store data under opration. ie. main memory. How we organize data in memory affects the time to finish certain operation. 
Hence we explore how to better organize the data in memory.

We always ideate on different datastructures that provide a specific operation. And analyse the time each DS takes, and by comparing those we decide on the best DS and pick that. Hence the analysis is importatnt.

# RAM
As there are many varieties of computers produced by various vendors, and have different capabilities. But we want our prolesm to be solved by all of them.
For that we want to pick a better DS or algorithm. 
Faster computers will be faster, than the ones which have less computing resources. But the analysis we do is regardless of the harware. Hence we should reason the the time/space complexity independent of the machine specifics, memory hierarichies, no. of processors, amount of memory, processor speed etc.
If we choose a better algorithm it will be better for any machine. As there are many varieties of computers, we don't consider the specifics of computers in reasoning about time to finish operations.
Hence, to make our analysis indepedent of machine,
To explore this we use a need simple (abstract)model of computer probably therotical one. We take RAM model of computer.
Get to know more about RAM [here](https://www8.cs.umu.se/kurser/TDBA77/VT06/algorithms/BOOK/BOOK/NODE12.HTM#SECTION02131000000000000000) and [here](https://www.cse.cuhk.edu.hk/~taoyf/course/comp3506/lec/ram.pdf).

But I'm also listing provoerties here:

- Each simple operation (+, *, -, =, if, call) takes exactly 1 time step.
- Loops and subroutines are not considered simple operations. Instead, they are the composition of many single-step operations. It makes no sense for ``sort'' to be asingle-step operation, since sorting 1,000,000 items will take much longer than sorting 10 items. The time it takes to run through a loop or execute a subprogramdepends upon the number of loop iterations or the specific nature of the subprogram.  
- Each memory access takes exactly one time step, and we have as much memory as we need. The RAM model takes no notice of whether an item is in cache or on the disk, which simplifies the analysis.   

Because RAM should only read data from Ram memenory, and computations done in registers is constant. Hence we are more focused on accessing data from ram memory.
As descibed above RAM has memroy in which data is stored. 
- The memroy is byte addressible. It has indexes sarting from 0. We can access any byte with its index
- Each byte can be accessded independtly.
- Accessing anybyte takes a same amount time.
- index of a byte can be stored in another 


RAM takes some time to finish the operation. Our goal is to minimize the time. The way data is organized affects the time it takes to finish.

Pointer is an address of a byte. We point to first byte of the record usually. To go to next object, we jump to address of first byte of the next record. 
We do this by adding the current pointer with no. of byte current occupies.

## Accounting on RAM
The running time is a function. It is a funciton of input.
By choosing RAM model we've removed the machine specifics from the analysis (analysis to choose the best DS), this is the first major step in analysis.
And we are also intersted in the runtime when the input is very large, we call this asymptotic analysis. Because it gives us the worst-case running time.
So, we produce worst-case running-time function, best-case running-time funciton and average-case running time function after the analysis. Thought most of the time we are interested in worst-case running-time function.
As it is a theoretical model and we cover a problem not a specific instalce of the proglem. We mention time and space needed to finish operation in function of input. 
For example accessing memory takes a unit time. If we needed to do 10 such operations to finish an operation. Genrelizing it makes O(n) - descibe it.
Mention asysmptotic analysis and give links here.
https://www.cs.cmu.edu/~clo/www/CMU/DataStructures/Lessons//lesson9_1.htm

## Organizing data on memory
Data is organized in memory.
How we organize the data affects the time it takes to finish certain operations.
Assume we are working at at time with on ly one problem. 
The data is related to a bpoblem.
As we use memory of RAM to store data under operaton, we read and write the to the memory. 
For the sake of ease of explaning, I'll consider data under operaion to be group of objects.
Data strcture is group of objects with a specific way to traversing them.

Each reacord/object is the basic element of data structure.

Hence the basic operations will be be adding more objects, deleting existing objects, modify existing ones and finding objects with required attributes.
For now assume all records in DS are of same type and has same size.

We explore the well known data structures.
### linear/contigous
Lets explore organizin data under processing in RAM and check the time to take all basic operations:
The first way to organize the data is storing records linear/consecutive. Here, every element is stored next to its previsou element.

While traversing the DS, may be to find an object with a specific key, we jump to the next object, until we read end of DS or we found our target object.
Another advantage here, is if you want to access nth element the next address to jump can be calculated. And the nth element can be accessed in constant time.
But what about delete? Deleting someting in middle destroys the assumption that the next element can be found just beside.
You can easly insert at the end. finiding takes O(n) time.
https://www.cs.cmu.edu/~clo/www/CMU/DataStructures/
Array is traversed linearly and data is organized in memory contigously.
So, if you are storing the data contigously, then you need to specify the max. amount of size, as we need to check for that much of contigous memeory. We need to reserve tat much of memory.
### non-linear/linked
To facilitate the deletion, we can need to use another way of organzing data, a non-linear/non-contigous ways.
Where each record will store the address of next record. Linked list. single/double. your wish.
But here we loose the benefit of accessing nth element in constant time.

See, find/seach are such an importatnt operation as it may be subproblem of every operation.
Hence we try to optimize.
Linked lists are traversed lineraly and stored non-contigously.

#### trees
https://www.cs.cmu.edu/~clo/www/CMU/DataStructures/
If you traverse the DS linearly, then we may call it linaer DS. Ex: array, LL, and ADTs constructed with these. Find/search will be of linear in the size of input.
Hence tree is an example of non-linear DS.
Considering our goal of reducing time, we are more interested in a DS like BBST. Where you'll have find/search operation in logarithmic of input. 
In this DS we store 
We may be storing he elements the order we received into linear structure and finding an element in that is of linear tiem.


But if we were to arrange data in such a way that the find operation we want will be fast.
Hence we need to rearrage the items/ or create a specific DS that helps to achie our goal. One such example is BBST or height balanced BST.
First we understand BST. every node has two children right childre in greater and left children is lesser than the key.
By this, the key under seach is greater you make a right move, with this you've reduced your search space to only the right subtree of the no de.

When the height is balanced, every steap we take we are reducing the searchs pace by half. Which is similar to binary search.

We shoud make sure that before and after any operation that alters the DS, the DS is in which it caters the initial need. For example to get the 
benefit we need the BST to be always balanced. so, we will spend time to make it balance every time we didi a operation.

Reference: https://www.cs.cmu.edu/~clo/www/CMU/DataStructures/Lessons//lesson4_1.htm

More generilized structres are also there.
B trees, B+ trees that has wide usage in the databaeses.



As we discussed fundametal DSs. linear (array, linked list) and non-linear (trees)
linked list and array are different, as the items are stored non-linearly but accessed lineary in linked list.


If we wan to insert new object, we need to find a place for it, if we wan to delete/update we first need to find it.
The structre (how do we traverse or relation among elements) governs the way 

Importance of find/search operation:
find/search is so importatnt. it is prerequisite to other operations. For eaxmple, to insert in a position


Computer takes certain amount of time to do an operation on the data. 
Our goal when we talk about DS is that to make sure the time to do an operation is reasonable.
We don't want to wait to see the operation to be finished.


So it has a place to store the data. It is called Random Access Memory or main memory or simply memroy.




































Contigous memory: why should we mention size of memory while requesting?
determinism
Pointer based structuring. - heap.

*Refer the notes and other references.*

"call stack" vs heap memory
call stack - https://www.youtube.com/watch?v=aCPkszeKRa4
https://en.wikipedia.org/wiki/Stack_(abstract_data_type)#Software_stacks



traversing array is better than traversing LL?
https://www.quora.com/Is-it-faster-to-iterate-through-an-array-or-a-linked-list

Know about implicit-datastructures. array is implicit DS for heaps.
https://en.wikipedia.org/wiki/Implicit_data_structure

static vs dynamic data structures.
array is static. length is fixed.
list is dynamic it can grow or shrink




How all other data structures fit in this model.

- https://www.learncpp.com/cpp-tutorial/79-the-stack-and-the-heap/
- https://en.wikipedia.org/wiki/Data_structure

Built-in data types of different languges.
Data structure support in different languges.

The concept of *private heap*, a managed heap space.
https://docs.python.org/2/c-api/memory.html


OS, in its memory management, does protect memory from being accessed by other running programs.
https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/8_MainMemory.html

Handling big data in problem solving. http://homes.sice.indiana.edu/yye/lab/teaching/spring2014-C343/datatoobig.php
External algorithms




What computers can do?
RAM.
Random access memory.
Structuering data
ADT
implementation of ADTs

How programming languages like python provies implementations to the datastructues.
- https://wiki.python.org/moin/TimeComplexity


Dynamic storage allocation: https://users.cs.northwestern.edu/~pdinda/ics-s05/doc/dsa.pdf



================== for editing =================


There are problems that demand certain operations on given data(collection of primitive or user-defined objects). Those operations may include searching for an object in the data, adding an object to and deleting an object from the data with other problem specific queries on the data. To accommodate these operations effectively, we choose a suitable data structure. For this we study various existing data structures and should be able to create data structure that satisfeid our requirement. As these operations of the data affects the overall performace of the algorithm

(Some problems are about working with data(collection of primitive or user-defined types) rather Some solutions, rather than being a  computation As part of our solution, we may have to store the data and perform such operations as: store more elements, delete elements, update existing elements and more importantly find required element. (For example, while working on a solution for token allocatoin system  and CRUD operations on the data improvise?). As these operations on data affects the overal effectiveness of an algorithm, we always ensure those operations are effective by picking suitable data structure. Solutions that just need numerical computation may not any data structure. Calculating GCD, Prime number checking etc. are some examples to give an idea.)



# RAM model of computation

We use computers to offload our work. Computers are dumb. They must be instructed precisely. When we understand how computers work always make us a better programmer. Now computers are ubiquitous, we've already built great solutions to our tasks. As physically computers are ubiquitous, the software that is solves our tasks and complex software systems that aids the programmers and makes best out of computers is invisible. Through software we either make the best out of the physical computer or the worst out of it. These days we rarely program or instrcut the actual physical computer which is really a tough job. We've built complex software systems -- compilers, interpreters, operating systems, VMS, IDEs that makes our job or programming easier. We program those software systems which intern program other software systems or the actual computers. But at the end, we've to do the thinking, we've to solve the problem. Once we've the solution ready, we tell it computer. Computers good at precisely doing what's beeing told, nothing more. We not only think to solve the problem. We should ensure that our solution on a computer effectively uses the resources. Make most of the computer.

So, algorithm is a written solution for computer. If our solution needs storing and retriving data as well then we use data structures so that storing and retriving is done effectively (there by boosing our solution). (some algo may not need DS).

## Better solution
We can produce many algorithms for the very same problem but we tend to choose the one that takes comparatively less resouses like time, memory etc. (With high data needs memory is also a constraint.).

https://cs.lmu.edu/~ray/notes/algds/



-------------------
We structure the data so that If our solution requires data related operations like storing, retrieving Most of the times algorithm the solution may work on the input data -- storing, retrive sometimes large amounts of data less Some algorithms are and data structures are the tools to reason about our solutions.  Some algorithms work


  it would be easier to  various physical componentsIt is a good thing to understand various physical components of computer and the way these components are used to follow. 
Importatnt ref: 

----------------------

Solving problems with computers involves manipulating data in memory
If your problem requires to work on a group of objects.
When your solution or algorithm involves data related operations
Computers take time when we .
We study data structures to understand how program execution time is affected by data structure used.
The reason we strudy data strures is that program execution time is affected by the data structure used.
The memory or main-memory is where computer stores data an important component of Computer. Data is fetched from and written to main-memory during execution. So, Data Structure is the way the data is 
Computer store data in main-memory
Data Structrus is the way we organize group of 

To understand computers we need to understand the memory, an important component.
 under process. CPU fetches/store the data from/to memory as it works. So, obviously how fast we are able to fetch/store the data will affect the overall time to finish certain operation on the data. Hence, data structues is organizing the data in memory for efficient retrieval/storing.

(Usually while solving problems we deal with data that is stored on memory. As part of our solution we will perform certain operations on data. How fast we finish those operations affects the time our solution take to finish. So, DS is aobut how we organize data such a way that desired opeations on the data is fast.)

To better organize the data we need to understand how memory looks like, we may need to compare various data structures based on the constraints/requirements etc. 
Memory is a gaint array of byes. 
We compare various DS on the basis of time they take to finish certain operations. Because we always want to minimize the time taken.
To compare various DS we need a theoretical computer, on which we calculate time each operation takes. 
Why a theoritical omputer? Because we've computers with various hardware configuration like computers with more clock speed, more/less RAM, various levels of cache, these days we've multi-core processors. So, to simplify the analysis, to make a common ground for benchmarking we choose a theoritical model, RAM model of computation.
So, with RAM model, we are avoiding the influence of hardware config on analysis. 

Studying DS and Algo will be useful in efficient use of computer resources. Solving a problem with a computer takes time, which higly depends on Runtime of our solution depends on 

Studying DS and Algo will be useful in efficient use of computer resources. Solving a problem with a computer takes time which highly depends on the algorithm and data structure used. Along with time we may also want to minimize the memory required, because general purpose computers have limited memory which may be shared among different processes(rewrite?).  Algorithms and data structures exists together. The time it takes to solve a problem is propotional to the no. of `physical operations` in the solution. Our goal is to reduce the no. of those physical operations. We produce various solutions to solve a particular problem. We finalize a solution which takes less time and utilize less memory. The solutions will be comprised of algorithms and data structures. To pick the best solution for a particular problem irrespective of hardware it is run on, we benchmark all solutions on a simplified, theoretical computer. It is RAM machine of computer

But, we do not want to execute 
Further reading: https://softwareengineering.stackexchange.com/questions/239045/what-is-the-relationship-between-data-structures-and-algorithms
Fruther reading: https://www.unf.edu/~wkloster/3540/wiki_book.pdf
Further reading: https://www.youtube.com/watch?v=V5he1JXiQbg
Studying DS will be useful in efficient use of computer resources.


RAM:

[So, as we are using RAM as our only computer on which we benchmark everything, the only factor that affects the runtime of algorithm is size of the datastruture/size of data incase of algorithms. For example, to fetch the last element on linked list with 10 elements will take us 10 memory accesse, if the size if 5 then 5. etc.
So, the time complexity is functino of input.]

[The running time is a function. It is a funciton of input.
By choosing RAM model we've removed the machine specifics from the analysis (analysis to choose the best DS), this is the first major step in analysis.
And we are also intersted in the runtime when the input is very large, we call this asymptotic analysis. Because it gives us the worst-case running time.
So, we produce worst-case running-time function, best-case running-time funciton and average-case running time function after the analysis. Thought most of the time we are interested in worst-case running-time function.
As it is a theoretical model and we cover a problem not a specific instalce of the proglem. We mention time and space needed to finish operation in function of input. 
For example accessing memory takes a unit time. If we needed to do 10 such operations to finish an operation. Genrelizing it makes O(n) - descibe it.]
-->