---
title: 【算法导论】：van Emde Boas树
date: 2023-02-02 14:07:47
cover: /2023/02/02/【算法导论】：van-Emde-Boas树/vEB.png
categories: 算法模块
tags:
    - 算法
    - 编程语言
    - 啃书
    - 算法导论
photos:
    - /2023/01/19/【算法导论】：红黑树/算法导论.png
link_refer:
    -
     url: https://www.luogu.com.cn/blog/RuntimeErrror/ni-suo-fou-zhi-dao-di-shuo-ju-jie-gou-van-emde-boas-shu
     title: https://www.luogu.com.cn/blog/RuntimeErrror/ni-suo-fou-zhi-dao-di-shuo-ju-jie-gou-van-emde-boas-shu
---
# van Emde Boas树

本章重点是探究如何实现时间复杂度为：$O(loglog\mu)$的树形结构。

更多是查看书籍中的理论以及伪码。

其中$\mu$表示关键字全域的大小，$n$表示当前集合中的元素值，关键字不能重复。

（$\mu$是2的幂，则$\sqrt{\mu}$是一个整数）

van Emde Boas树的关键思想就是使用度为$\sqrt{\mu}$的叠加树

<!-- more -->

叠加二叉树时间复杂度：$O(log\mu)$

度为$\sqrt\mu$的树时间复杂度：$O(\sqrt\mu)$

虽然上面两种时间复杂度都不是我们所追求的，但它们都是van Emde Boas树的基础。

详细结构可以直接看算法导论P307-308，这里不再赘述。

## **递归结构**

我们这里假设$\mu=2^{2^k}$，其中$k$为整数。因此$\mu^{\frac{1}{2^k}}$都为整数，这个限制在实际应用中限制太大，但在下一节中会进行改进。

我们的目标是使得增删改查操作的时间复杂度为：$T(\mu)=O(loglog\mu)$

其递归式应当为（这是关键递归式）：
$$
T(\mu)=T(\sqrt\mu)+O(1)
$$
设$m=log\mu$，则：
$$
T(2^m)=T(2^{\frac{m}{2}})+O(1)
$$
现在重命名$S(m)=T(2^m)$，则有：
$$
S(m)=S(m/2)+O(1)
$$
为了处理下标，我们定义下面的三个函数，分别表示$x$的簇号（$high$函数）、$x$在簇中的位置（$low$函数）、且$x=index(high(x),low(x))$
$$
high(x) = \lfloor x/\sqrt\mu \rfloor
\\
low(x) = x \% \sqrt\mu
\\
index(x,y) = x\sqrt\mu+y
$$

### **原型van Emde Boas结构（proto-vEB结构）**

全域：{$0,1，…，\mu-1$}

1. 如果$\mu=2$，那么它是基础大小，只包含一个两个位的数组$A[0…1]$；
2. 否则，由于我们这里设定$\mu=2^{2^k}$，因此$\mu \ge 4$，除了全域大小外，$proto-vEB(\mu)$还具有以下属性：
    1. 一个名为$summary$的指针，指向一个$proto-vEB(\sqrt\mu)$结构；
    2. 一个数组$cluster[1…\sqrt\mu-1]$，存储$\sqrt\mu$个指针，每个指针都指向一个$proto-vEB(\sqrt\mu)$结构。

$summary$用于标记是否存在某些元素，它也是一个$proto-vEB(\sqrt\mu)$结构。

$cluster[i]$表示了$i\sqrt\mu$到$(i+1)\sqrt\mu-1$的值，形成了第$i$个簇。

![原型vEB](【算法导论】：van-Emde-Boas树/原型vEB.png)

上面是一个$proto-vEB(16)$结构的展开图。

关于$summary$的解释，感觉像是标记某个$cluster[]$中是否存在元素，具体可以看算法导论第311页。

#### **原型van Emde Boas 树的操作**

##### **判断一个值是否在集合中**

借助$summary$结构，找到该值对应的位。

递归的往下找$summary$结构。

直到找到基础情形：$proto-vEB(2)$，则直接返回对应位。

时间复杂度：$O(loglog\mu)$

##### **查找最小元素**

首先判断是否为基础情形，使用$O(1)$的时间复杂度解决。

若不为基础情形，接着使用$summary$数组判断是否存在元素，若都不存在，则直接返回空。

若存在元素，则找到$summary$数组中的最小元素$idx$，表示最小元素在$cluster[idx]$中，接着查询$cluster[idx]$的最小元素即可。

该时间复杂度为$\Theta(log\mu)$

##### **查找后继**

同样的，首先判断是否为基础情形，使用$O(1)$的时间复杂度解决。

若不为基础情形，则递归查询元素对应簇的后继$idx$：

当后继不为空时，直接返回对应$index(high(x),idx)$；否则，需要查询$summary$数组中，该簇的后继：

若簇已经是最后一个簇，则返回空；否则，在对应下一个簇中查找该元素的后继并尝试返回。

该时间复杂度为$\Theta(log\mu \times loglog\mu)$

##### **插入元素**

要插入一个元素，需要将其插入对应簇中。

并将这个簇中的$summary$位设为$1$。

##### **删除元素**

与插入类似，删除后，将这个簇中的$summary$为设为$0$。

插入与删除的时间复杂度：$\Theta(log\mu)$

## **van Emde Boas树及其操作**

完整的van Emde Boas树不具有全域过多的局限性，$\mu$可以为任意一个$2$的幂，当$\sqrt\mu$不为整数时，我们将一个数的$log\mu$位分隔成最高$\lceil (log\mu)/2 \rceil$位与最低$\lfloor (log\mu)/2 \rfloor$位。

我们设：$\sqrt\mu^{up} = 2^{\lceil (log\mu)/2 \rceil}$，$\sqrt\mu^{down} = 2^{\lfloor (log\mu)/2 \rfloor}$

且$\mu = \sqrt\mu^{up} \times \sqrt\mu^{down}$

上面的辅助公式改为：
$$
high(x) = \lfloor x/\sqrt\mu^{down} \rfloor
\\
low(x) = x \% \sqrt\mu^{down}
\\
index(x,y) = x\sqrt\mu^{down}+y
$$
van Emde Boas树的结构：

递归情况：

1. 属性$summary$指向一棵$vEB(\sqrt\mu^{up})$树；
2. 数组$cluster[0…\sqrt\mu^{up}-1]$指向$\sqrt\mu^{up}$个$vEB(\sqrt\mu^{down})$树；
3. $min$存储$vEB$树中的最小元素；
4. $max$存储$vEB$树中的最大元素。

普通情况：

1. 只包含$min$和$max$两个元素

增加$min$和$max$两个元素，可以减少很多递归情况。

![vEB](【算法导论】：van-Emde-Boas树/vEB.png)

### **van Emde Boas树的操作**

#### **查找最小元素和最大元素**

可以直接利用属性$min$和$max$，返回最小元素和最大元素。

#### **判断一个值是否在集合中**

先判断该值是否为最大元素或最小元素；

再判断该集合大小是否为2；

若都不是，则递归查询对应的$cluster$

#### **查找后继与前驱**

首先从基础情况判断；

接着判断$x$是否比最小值还小，若是，则返回最小值；

查询$x$的同一簇的最大值，当该值不为空且$x$比它小，在该簇中查询$x$的后继；

否则，查询$summary$中$x$对应簇的后继，在后继簇中查找$x$的后继。

#### **插入一个元素**

特判插入空集合的情况。

当插入元素比$min$小时，需要修改$min$；

当插入元素比$max$大时，需要修改$max$；

否则查找插入元素对应簇的最小值，若为空，则在$summary$中插入$high(x)$，表示增加该簇，并插入$x$；否则递归的插入$cluster$中。

#### **删除一个元素**

特判当前$vBE$树的$min==max$的情况，表示当前树只包含一个元素，直接删除这个元素即可。

再特判$\mu==2$的基础情况。

接着判断当$x==min$时的情况，找到除了$min$最小元素所在的簇号，设置$x$为这个簇最小元素的值。

递归的删除$x$，并更新$summary$与$min$、$max$。

**PS**：本来想用模板元编程和面向对象形式维护一棵van Emde Boas，
但是突然想起模板元编程只允许常量参数，搞起来还挺麻烦的，
这里就不再给出代码（洛谷已经有佬写过，可运行且完善），
具体代码可以参考下方链接。
