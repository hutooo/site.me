---
title: The Little GC
author: ash
tags: ["gc"]
categories: ["LittleToys"]
date: 2021-09-25T20:39:00+08:00
cover: "/images/gc.jpg"
---


"小伙子，你是什么垃圾?" --- 垃圾回收管理员

"噢~ 可回收垃圾..."

# 0x01 什么是GC

GC 是 `Garbage Collection` 的简称，通常叫做 垃圾回收. 在 GC 中，把程序不用的内存空间视为垃圾. 

## GC是用来干嘛的?

GC针对垃圾需要做两件事.

1. 找到内存空间里的垃圾
2. 回收垃圾，让程序能再次利用这部分空间

满足这两项功能的程序就是GC.

## 使用GC的好处

在没有GC的世界里，程序员必须手动进行内存管理，必须清楚地确保必要的内存空间，释放不要的内存空间，但人总会遗漏和犯错.

* 忘记释放内存空间将会导致内存泄露，这样的程序放任不管，会把内存占满，甚至导致系统崩溃.
* 另外，在释放内存空间时，如果忘记初始化指向释放空间的指针，将产生"悬垂指针"，如果程序中误用悬垂指针，将产生难以预料的BUG，也可能导致严重的安全漏洞.

有个GC之后，就能省去人工管理内存的麻烦，同时也减轻了程序员的负担，把精力集中在更重要的地方.

> 弊端: GC算法会增加程序的运行时开销，同时大部分GC算法会有STW问题.
> 当然现在有很多更好的GC算法[一种新的引用技术算法, 参考koka语言]，也有很多别样的内存管理方法: RAII, Ownership&lifetime(所有权与生命周期)... [参考Rust语言]

# 0x02 GC基本原理 [Reduce, Reuse, Recycle]

GC背后的基本思想是，程序语言(在大多数情况下)似乎可以访问无限内存. 开发人员只需要不断地分配、分配和分配，就像变魔术一样，它永远不会失败.

当然，机器没有无限的内存. 因此，GC的实现方式是，当它需要分配内存并意识到自己的内存不足时，会收集垃圾[回收内存].

如果程序访问的对象开始被回收，就比较尴尬. 因此为了GC正常运作，必须确保程序无法再次使用这个对象. 如果它不能得到对象的引用，那么它显然不能再次使用它. 因此，"在用"可以这么定义和理解:

1. 任何被范围内的变量引用的对象都在使用中

2. 任何被另一个正在使用的对象引用的对象都在使用中

第二条规则是递归规则. 如果对象a 被一个变量引用，并且它有一些引用对象b 的字段，那么 b 就被使用了，因为你可以通过 a 找到它.

根据规则，最终呈现的是一个 可达对象 图表，你可以从一个变量开始遍历对象，对于程序来说，任何不在可访问对象图中的对象都是非活跃对象，可以回收它们的内存空间.

> 实际上，GC相当于虚拟内存. 一般的虚拟内存技术是在较小的物理内存的基础上，利用辅助存储创造一片看上去很大的“虚拟”地址空间，也就是说，GC是扩大内存空间的技术，因此我称其为空间性虚拟存储。这样一来，GC就成了永久提供一次性存储空间的时间轴方向的时间性虚拟存储. 神奇的是，比起"垃圾回收"，把GC称为"虚拟内存“令人感觉其重要了许多. --- 《垃圾回收的算法与实现》

# 0x03 GC算法

关于GC算法，有非常多的选择，引用计数，标记清除，GC复制，分代回收等等... 不同的算法也有很多变种，以适应不同的语言和项目~

本篇采用简单的标记清除算法，它也是最早的GC算法，由计算机和人工智能先驱--约翰·麦卡锡 发明~

# 0x04 标记清除算法

标记清除算法的原理与我们对可达性的定义差不多:

1. 从根对象开始，遍历整个对象图,每达到一个对象，就对它进行标记.

2. 完成1后，将所有未标记的对象删除.

看~ 就是这么简单~

那么，再开始实现GC之前，我们需要一个准备一些东西...

# 0x05 一个小型虚拟机

垃圾回收，需要垃圾才能回收~ （emmm... 听君一席话，如听一席话）

所以我们需要一个虚拟机来制造一些垃圾进行回收~ 因为主题是GC，所以这个虚拟机简单一点就好...

## 语言部分

现在我们设计一个玩具级语言~ 它是个动态类型语言，只有两种类型，int 和 pair， 用枚举体表示:

```c++
typedef enum {
    OBJ_INT,
    OBJ_PAIR
} ObjectType;
```

pair翻译成中文 `对/一对` 就有点别扭~ 总之，它表示两个东西的组合...

pair可以是两个int, 也可以是一个int和两一个pair，也可以是两外两个pair. 

虚拟机(`VM`)种的对象可以是Int、Pair 这两种类型的任意一种，使用联合体来表示:

```c++
typedef struct sObject {
    ObjectType type;  // 用来区分对象的类型

    union {
        // INT类型对象
        int value; 

        // PAIR类型对象
        struct {
            struct sObject* head;
            struct sObject* tail;
        };
    };

} Object;
```

Object结构用于表示对象，ObjectType用来区别是Int还是Pair类型；之后的联合体用来存储类型相关的数据.

因为本篇目的是GC，所以语言部分就先到这里~

## 虚拟机部分

现在我们把这个玩具小语言包装到一个虚拟机结构中.

本篇的虚拟机使用一个堆栈，来存储当前作用域中的变量.

> 大多数语言的虚拟机要么是基于堆栈(JVM、CLR...) ，要么是基于寄存器(Lua...). 但无论哪一种，事实上依然会有一个堆栈，用来存储表达式中间所需的局部变量和临时变量.

方便起见，这里简单的构建这个虚拟机模型:

```c++
// 虚拟机堆栈的最大容量
#define STACK_MAX 256

typedef struct {
    // 存放对象的堆栈
    Object* stack[STACK_MAX];

    // 堆栈的当前大小
    int stack_size;
} VM;
```

嗯~ 确实很"简洁"的模型呢.. 接着来创建一些函数吧~

首先，需要一个虚拟机的初始化函数:

```c++
VM* newVM() {
    // 为VM申请内存空间
    VM* vm = malloc(sizeof(VM));
    vm->stack_size = 0;  // 初始化VM的堆栈大小
    return vm;
}
```

好呀~ VM有了，还得能对它进行简单的操作:

```c++
// 将对象推入VM的堆栈
void push(VM* vm, Object* obj) {
    assert(vm->stack_size < STACK_MAX, "stack overflow!");
    vm->stack[vm->stack_size] = obj;
    vm->stack_size += 1;
}

// 从虚拟机堆栈弹出对象
Object* pop(VM* vm) {
    assert(vm->stack_size > 0, "stack underflow!");
    Object* obj = vm->stack[vm->stack_size];
    vm->stack_size -= 1;
    return obj;
}
```

以上，我们有了将对象推入和弹出VM的操作了，那么再添对象的创建操作:

```c++
// 建立一个新对象, 标记上对象的类型
Object* newObject(VM* vm, ObjectType type) {
    Object* obj = malloc(sizeof(Object));
    obj->type = type;
    return obj;
}
```

创建了对象，我们就可以把不同类型的对象推入VM了~

```c++
void pushInt(VM* vm, int intVal) {
    Object* obj = newObject(vm, OBJ_INT);
    obj->value = intVal;
    push(vm, obj);
}

Object* pushPair(VM* vm) {
    Object* obj = newObject(vm, OBJ_PAIR);
    obj->tail = pop(vm);
    obj->head = pop(vm);
    push(vm, obj);
    return obj;
}
```

至此, 一个小型VM就算完成了，如果有配套的解释器和解析器，我们就有了一个真正的语言！

好吧，没有~ 那就收垃圾好了...

# 0x06 构建GC

## 标记

由于本篇使用的是标记清除算法，所以第一步，就是标记.

我们需要遍历所有可达的对象，设置他们的标记位, 所以当然得给Object结构一个标记位..

```c++
typedef struct sObject {
    bool marked;  // 新加的标记位
    // ... 剩余部分，往上翻，看之前的定义
} Object;
```

有了标记位，我们创建新对象时(newObject)，需要将标记位初始化:

```c++
Object* newObject(VM* vm, ObjectType type) {
    Object* obj = malloc(sizeof(Object));
    obj->type = type;
    obj->marked = false;
    return obj;
}
```

为了标记所有可访问的对象，需要遍历堆栈:

```c++
void markAll(VM* vm) {
    for (int i = 0; i < vm->stack_size; i++) {
        mark(vm->stack[i]);
    }
}

void mark(Object* obj) {
    obj->marked = true;  // 将对象标记为可达.
}
```

但是这样还不够，因为对象可能还有引用(可达性递归)，如果对象是pair，那么它的两个字段也是可达的.. 

我们修改mark函数:

```c++
void mark(Object* obj) {
    obj->marked = true;

    if (obj->type == OBJ_PAIR) {
        mark(obj->head);
        mark(obj->tail);
    }
}
```

完工！！ 哎，等等，这里貌似有个BUG~ 

我们递归的调用了mark，但是如果有些pair互相引用，则递归永远不会结束，直至栈溢出并导致崩溃..

这问题也相对比较容易解决，只需要在遇到已经标记过的对象时，退出即可.. 

再次修改mark函数:

```c++
void mark(Object* obj) {

    if (obj->marked == true) return;

    obj->marked = true;

    if (obj->type == OBJ_PAIR) {
        mark(obj->head);
        mark(obj->tail);
    }
}
```

OK! 现在我们的markAll函数，可以正确标记内存中所有的可达对象~

## 清除

现在对象标记完了，我们需要清除所有未标记的对象.

但是麻烦的地方在于，这些没用打上标记的对象，都访问不到(不可达)~ 这可咋整呢~

> 我们的VM实现，只在类型元素中存储了指向对象的指针~ 一旦丢失，就无法再访问，并且也造成了内存泄露.

为了克服这么问题，有个简单粗暴的方法，就是用VM来维护和跟踪每一个分配的对象，我们修改Object结构:

```c++
typedef struct sObject {
    // 我们把所有的Object串起来~ 这样VM就可以通过第一个对象，找到所有的对象..
    struct sObject * next;

    // ... 剩余部分，往上翻，看之前的定义
} Object;
```

所以让VM来跟踪第一个对象，修改VM结构:

```c++
typedef struct {
    // 所有对象链表的头部(第一个对象)
    Object* firstObj;

    // 存放对象的堆栈
    Object* stack[STACK_MAX];

    // 堆栈的当前大小
    int stack_size;
} VM;
```

同时，就也需要修改 VM 和 Object 的创建函数:

```c++
VM* newVM() {
    // 为VM申请内存空间
    VM* vm = malloc(sizeof(VM));
    vm->stack_size = 0;  // 初始化VM的堆栈大小
    vm->firstObj = NULL; // 初始化对象链表头
    return vm;
}

Object* newObject(VM* vm, ObjectType type) {
    Object* obj = malloc(sizeof(Object));
    obj->type = type;
    obj->marked = false;

    obj->next = vm->firstObj;  // 更改新建对象的next元素为 VM的对象链表头部
    vm->firstObj = obj; // 更改VM的对象链表头部为 新创建的对象
    return obj;
}
```

现在我们可以清除未标记的对象了:

```c++
void sweep(VM* vm) {
    Object** obj = &vm->firstObj;
    while (*obj != NULL) { // 对象非空
        // 对象未标记
        if (!(*obj)->marked) {
            Object* unreached = *obj; // 将对象指针复制给 unreached 变量

            // 清理未标记对象
            *obj = unreached->next;
            free(unreached);  // 释放unreached变量
        } else {
            // 此处的对象已经标记了，所以取消标记，以备下一轮GC
            (*obj)->marked = false;

            obj = &(*obj)->next; // 使用next对象, 进入下一轮循环
        }
    }
}
```

上面代码看起来有点绕，其实本质的工作就是遍历整个对象链表，清理未标记的遍历，将已标记变量的标记初始化..

那么把 markAll 和 sweep 函数结合一下，就是一个 垃圾回收器了:

```c++
void gc(VM* vm) {
    markAll(vm);
    sweep(vm);
}
```

## 启动GC

到这里，GC是有了，那么与之而来的新问题就是，什么时候启动GC？ 内存不足嘛？什么时候才算内存不足呢?

这里貌似没用什么标准的答案，只能根据具体的情况，具体分析了~

这里为了简单起见，我们就让GC在分配了一定数量的对象后，开始启动并回收垃圾吧~ (其实一些语言就是这么做的..🤫)

那么，我们扩展VM结构，对分配的对象数量进行追踪:

```c++
typedef struct {
    // 当前已分配的对象数量
    int numObjs;

    // 最大可分配的对象数量
    int maxObjs;

    // 所有对象链表的头部(第一个对象)
    Object* firstObj;

    // 存放对象的堆栈
    Object* stack[STACK_MAX];

    // 堆栈的当前大小
    int stack_size;
} VM;

VM* newVM() {
    // 为VM申请内存空间
    VM* vm = malloc(sizeof(VM));
    vm->stack_size = 0;  // 初始化VM的堆栈大小
    vm->firstObj = NULL; // 初始化对象链表头

    vm->numObjs = 0;

    //GC启动的 初始阈值: 数字越小在内存方面越保守，数字越大则在垃圾收集上的时间越少~
    vm->maxObjs = GC_THRESHOLD;

    return vm;
}
```

也得修改 newObject 函数:

```c++
Object* newObject(VM* vm, ObjectType type) {

    // 到达阈值，启动GC
    if (vm->numObjs == vm->maxObjs) {
        gc(vm);
    }

    Object* obj = malloc(sizeof(Object));
    obj->type = type;
    obj->marked = false;

    obj->next = vm->firstObj;  // 更改新建对象的next元素为 VM的对象链表头部
    vm->firstObj = obj; // 更改VM的对象链表头部为 新创建的对象

    vm->numObjs += 1; // 创建了新的对象，当前对象数量 +1
    return obj;
}
```

最后，调整一下 gc函数，以适应上面的更改:

```c++
void gc(VM* vm) {

    // int numObjs = vm->numObjs;

    markAll(vm);
    sweep(vm);

    vm->maxObjs = vm->numObjs * 2;  // GC阈值更改为当前对象数量的2倍.
}
```

这里，我让 阈值随着每次GC后的活动对象数量进行变动~ 因此会自动扩容和收缩..


# 0x07 完结

至此，我们构筑了一个简单的垃圾回收器~ 

别看它现在很简陋，它也是一个真正合法的GC呢~ 如果在这个基础上加入大量的优化，就可以接近 Ruby、Lua的GC.

[这里是全部代码](https://github.com/ash-z01)

# 0x08 参考

1. [Book <垃圾回收的算法与实现>](https://book.douban.com/subject/26821357/)
2. [Site <journal.stuffwithstuff.com>](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/)

