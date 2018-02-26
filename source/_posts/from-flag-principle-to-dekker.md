---
title: 软件方式的线程互斥方法的实现
date: 2016-02-03 13:46:44
tags: [锁, 虚拟化]
---

我们知道在多线程的环境中，对共享资源的互斥控制是必不可少的，最熟悉的方式就是锁。

## 四种同步互斥的控制方法
- 临界区(critical section)
- 互斥量(mutex)
- 信号量(semaphores)
- 事件驱动(event)

本文将描述一种软件方法实现的临界区控制方法，以及对他的优化。

<!-- more -->

## 场景描述
在做kvm虚拟化项目的时候，碰到了这样的一个场景：

hypervisor为虚拟机额外分配了1GB的内存空间，虚拟机操作这1GB的内存页会有特殊的行为(不是本文所关注的重点，只需要知道上层和下层会同时操作这内存大小为1GB的，共262,144个4K页)。

- 上层的虚拟机内核会不定期修改某些内存页的内容；
- 然而下层的虚拟机监控器(kvm or hypervisor)结合当前系统的运行状况，有选择性的对为上层虚拟机分配的内存页做迁移或者调度。

这两者并行执行，如果下层hypervisor进行调度的同时，上层虚拟机内核修改了内存页的内容，可能会导致不一致性。所以在这种场景下，需要对针对某个页的操作进行锁保护。 

具体实现来说，为这些内存页再分配一块内存区域，这块区域由VM和hypervisor共享，实际上这一区域存放一个大数组，数组的每个元素存放一些关于对应下标的内存页的使用信息。
实际上用已有的spin\_lock也是可以的，不过spin\_lock 是用xadd指令实现的，在guest-hypervisor这样的组合中，貌似会有问题，具体问题出现在什么地方还不知道，本文暂不关注。

到这儿，问题简化成：两个线程A和B，同时竞争进入critical section，如何用软件的方式实现互斥。


## Flag Principle
废话不多说，直接上代码：
```c
int flag[2] = {0, 0};

void process0() {
    ...

retry0:
    flag[0] = 1;
    if (flag[1] == 1) {
        barrier();
        flag[0] = 0;
        goto retry0;
    }

    // critical section here...

    flag[0] = 0;

    ...
}

void process1() {
    ...

retry1:
    flag[1] = 1;
    if (flag[0] == 1) {
        barrier();
        flag[1] = 0;
        goto retry1;
    }

    // critical section here...

    flag[1] = 0;

    ...
}
```

简单来说，两个线程共享两个flag变量，每一次进入critical section之前，首先置上自己的flag，然后去检查对方的flag，如果对方的flag已经置上，说明对方有可能已经进入、或者等待进入临界区，此时应该将自己的flag置为0，然后进入下一轮尝试。

加上barrier的原因是，编译器会认为进入临界区flag的值永远会是1，这个赋值0的操作是多余的，然后将flag[1] = 0优化掉，这样带来的结果就是：deadlock。

### 缺点
虽然这个设计能够避免死锁，但是由于flag = 0 与goto retry之后 flag = 1 之间间隔的时间非常短，有非常大的可能造成活锁。有一个简单的解决办法，就是在两个赋值中间加上一个随机的sleep时间，这样能够减缓活锁发生的可能性，不过这并不是好的解决方法。为了避免过分的谦让，我们引入了Dekker算法。

## Dekker 算法
```c

int flag[2] = {0, 0};
int turn = 0;

void process0() {
    ...

retry0:
    flag[0] = 1;
    if (flag[1] == 1) {
        if (turn == 1) {
            barrier();
            flag[0] = 0;
            while (turn == 1);
        }
        
        goto retry0;
    }

    // critical section here...

    turn = 1;
    flag[0] = 0;

    ...
}

void process1() {
    ...

retry1:
    flag[1] = 1;
    if (flag[0] == 1) {
        if (turn == 0) {
            barrier();
            flag[1] = 0;
            while (turn == 0);
        }
        
        goto retry1;
    }

    // critical section here...

    turn = 0;
    flag[1] = 0;

    ...
}

```

相比于flag principle，dekker算法加了一个turn的变量，这个变量等于0，表示0线程不必谦让，等于1表示1线程不必谦让。如果双方进入了之前的“活锁”状态，就会根据turn变量来裁决哪一方进入临界区，而另一方则无限循环等待。

## Peterson算法
Peterson算法看起来更直观，代码也更简洁，原理和dekker无大异。
```c
int flag[2] = {0, 0};
int turn = 0;

void process0() {
    ...

    flag[0] = 1;
    turn = 1;
    while(flag[1] && turn==1);

    // critical section here...

    flag[0] = 0;

    ...
}

void process1() {
    ...

    flag[1] = 1;
    turn = 0;
    while(flag[0] && turn==0);

    // critical section here...

    flag[1] = 0;

    ...
}
```

并且这个算法有一个好处，能够比较容易地扩展成N线程的情况。

## Filter算法
扩展成N个线程同时访问一个资源的filter算法，伪代码如下：
```c
// initialization
waiting[N-1] = { -1 }; // the waiting process of each level 0...N-2
level[N] = { -1 };     // current level of processes 0...N-1

// code for process #i
for(l = 0; l < N-1; ++l) {
    level[i] = l;
    waiting[l] = i;
    while(waiting[l] == i &&
          (there exists k ≠ i, such that level[k] ≥ l)) {
        // busy wait
    }
}

// critical section

level[i] = -1; // exit section
```

其中waiting数组类似于一个等待的房间，所有的线程想要进入critical section，就必须从小到大经过所有的房间，最坏的情况是N的线程有N-1个呆在房间里面等待，只有一个线程进入critical section。

对于level数组，表示的是每一个线程在第几个房间等待。例如 level[i] = j; 表示第 i 个线程在房间 j 等待。另外 -1 表示没有进入critical section的请求。

我们可以看出代码最核心的一段while判断，如果某个房间是的等待着是当前的线程，并且已经有别的线程在序号更大的房间了，那么就继续等待。由此可见，有两种方式能够让当前线程向前进：

1. 前方没有别的线程在更大的房间里面等待，则可以向前推进。
2. 有别的后来者线程也想进入我当前的房间，并且把waiting号给改了，这样当前线程就被动地被推进到下一级的房间。

由filter算法去反思Peterson算法，可见其中的flags数组表示两个进程的等待级别，而turn变量则是阻塞（忙等待）的线程队列，这个队列只需要容纳一个元素。



## 结语

事实上，软件方法实现的临界区算法有一个最大的问题就是没有拿到入场券的线程将会无意义地等待，这也是软件互斥方法无法避免的一个通病，不过在之前所描述的场景下，dekker算法是比较好的解决方案了。

---END---
