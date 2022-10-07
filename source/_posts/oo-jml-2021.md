---
title: BUAA OO 第三单元（ JML & 社交关系模拟） 博客总结
date: 2021-05-27
updated: 2021-05-27
tags:
- BUAA
- OO
- Java
description: JML 规格化设计，社交网络
---

## (1) 实现规格所采取的设计策略

这部分似乎没有太多特别的，总体上来说分为这样几个步骤：

1. 先阅读指导书和接口的定义，从宏观的角度了解要求实现的是什么；
2. 阅读接口的整体规格，并重点关注接口开头的 `public instance model ......` 部分（通常该部分提示着这个对象需要维护哪些数据）
3. 从小（ `Person`, `Message` 等结构较简单，维护的数据较少的类）到大（ `Group`, `Network` 结构复杂，维护的数据和含有的方法较多的类）进一步阅读方法的规格，在这一步考虑采用何种**容器**或**数据结构**来存储这些数据。
4. 详细阅读方法规格，依次实现方法。

## (2) 基于 JML 规格来设计测试的方法和策略

自己对于本单元的测试采用了对拍与黑盒为主，JUnit 为辅的测试策略。采用 JUnit 对自己感到不太把握的方法进行了较为基础的功能性弱测，测试了 qnr, qgvs, sim 方法，但并未对整个程序进行全面的单元测试。本地的黑盒测试主要针对了下述性能分析栏目中较容易 TLE 的方法，有针对性地手动构造了少量性能的极端测试点，验证了自己的实现并没有 TLE 。除此以外就是对拍，将自己程序的输出与其他人程序的输出进行比对以确保自己没有出现较严重的实现偏差甚至对规格理解的偏差。

## (3) 容器的选择和使用

在本次作业中自己主要采用了 `HashMap` ，`HashSet` 以及 `LinkedList` 容器。下面具体列出实现中采用的容器以及如此选择容器的依据。

### Person

- `HashMap: <acquaint, value>` ，用来储存一个人的有关系的人以及这些关系的 `value` 
  - `acquaint` 和 `value` 存在一一对应关系，故选用 `Map`.
- `HashSet: <group>` ，储存一个人所属的 Group。
  - 对于一个人只需知道它属于哪些 Group，主要关注"有没有"而与顺序无关，故选用 `Set` 。
- `LinkedList: <Message>` ，储存一个人接收到的 Message。
  - 由于 `getReceivedMessages` 方法以及 `Network` 中 `sendMessage` 方法的规格对 `messages` 的顺序有要求，故采用 `List` 进行存储；又由于 `sendMessage` 对 `messages` 进行的是头插操作，且没有涉及到对这个 `List` 的按下标访问，故相较于 `ArrayList` 而言选用了更适合头插与顺序遍历的 `LinkedList` 

### Group

- `HashMap: <Person.id, Person>` ，储存该 group 内含有的人。
  - 这里选用了 `Map` 而非 `Set<Person>` 是出于考虑到我们实现的 `Network` 需要大量的根据 `id` 查询到某个实体（人/组/消息）的操作，所以将 id 和实体建立 `Map` 以便于查询。
- `HashSet: <Edge>` ，储存该 Group 中所有的边。
  - 用于 `qgvs` 操作。

### Network

- `HashMap: <Person.id, Person>` 
- `HashMap: <Message.id, Message>` 
- `HashMap: <Group.id, Group>` 
- `HashMap: <Emoji.id, Emoji.heat>` ，用于存储该 Network 中含有的表情以及表情对应的热度。

### 并查集

- 采用了一个 `HashMap` 来储存每个对象的直接 `father` ，相较于 C 中传统的数组写法更加灵活。

## (4) 性能问题分析

本部分简要列举容易出现性能问题的方法以及避免 TLE 的策略。

- `isCircle` (qci)： 该方法实际意义为判断无向图上的两个节点是否连通，可采用并查集来实现。

  （稍微吐槽一下方法名）

- `queryBlockSum` (qbs)：求无向图中连通块的数量，和 `isCircle` 方法一样，采用并查集实现。（亦可采用单次 $O(n)$ 的 DFS/BFS 实现）

- `getAgeMean` / `getAgeVar` (qgam/qgav)：求一个 Group （Group 意义为无向图的一个子图）中所有人的年龄的均值与方差。这两个方法可以采用较为朴素的 $O(n)$ 方法实现，直接遍历该 Group 中所有的 Person；如果希望在这里卷性能(bushi)，可以在 Group 中添加两个变量来维护组内 Person 年龄之和与平方和，从而将这两个方法的复杂度降为 $O(1)$ 

- `getValueSum` (qgvs)： 求一个 Group 中所有边的边权之和。对于该方法如果直接按照规格去对组内的 Person 进行二重循环遍历并两两判断是否有边，则单次执行该方法时间复杂度为 $O(n^2)$ ，如果强测数据中有大量的 qgvs 指令则总体复杂度为 $O(n^3)$ 可能发生 CTLE 。一种可能的实现方法是采用一个变量来记录当前 Group 中所有边的边权之和，在对该组 `addPerson` 与 `delPerson` ，以及对组内的人进行 `addRelation` 时对边权和进行维护，使得 `getValueSum` 方法本身的复杂度为 $O(1)$ ，而 `addPerson` 与 `delPerson` 的复杂度分别为 $O(n)$ 与 $O(E)$ （$E$ 为图的边数）则总体的复杂度为 $O(n^2)$  ，可以满足性能要求。

- `sendIndirectMessage` (sim)：该方法的返回值是两个人之间的最短路，需采用 Dijkstra 算法对无向图中的单源最短路进行求解。注意使用 Dijkstra 时需要采用堆优化。

最后来点题外话 && Tips：如果一个方法的规格中出现了嵌套的量化表达式（即 `\exists` , `\forall` 等用于描述一定范围元素的量词），如果完全照翻规格（没有选用合适的容器也没有分析这段规格的含义）可能会写出时间复杂度 $O(n^2)$ 的双重循环时就需要小心了，这时需要理解规格并选用合适的数据结构与算法来实现。

## (5) 作业架构分析

本次作业在架构上不太需要我们实现者去过多考虑，所需要的类的数量以及实现哪些方法均是由规格给出的。在作业的实现中主要是考虑数据之间的关系，合适容器的选用以及对于较复杂的方法保证性能满足要求。只有第三次作业中增加了三种不同的 Message 所以对特殊种类的 Message 继承自己实现的 `MyMessage` 类并且对 `getSocialValue` 方法进行了 override ，满足了面向对象的特性。