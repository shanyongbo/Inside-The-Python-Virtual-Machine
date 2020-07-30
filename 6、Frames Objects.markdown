---
layout: post
title:  "Inside The Python Virtual Machine --  6、Frames Objects"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第6章介绍部分的中文内容。

<!-- more -->


## 6. Frames Objects

代码对象包含可执行的字节码，但缺少执行此类代码所需的上下文信息。以 list 6.0 中的一组字节码指令为例，LOAD_COST 将索引作为参数，但是代码对象没有数组或者从索引处加载值的数据结构，

Listing 6.0: A set of bytecode instructions

```python 
0 LOAD_CONST 					0 (<code object f at 0x102a028a0, file "fizzbuzz.py",\
line 1>)
2 LOAD_CONST 					1 ('f')
4 MAKE_FUNCTION 				0
6 STORE_NAME 					0 (f)
8 LOAD_CONST 					2 (None)
10 RETURN_VALUE
```

提供此类上下文信息的另一种数据结构是执行代码对象所必需的，而这正是 frame 对象所在的地方。人们可以将 frame 对象视为执行代码对象的容器：它了解代码对象，并且引用了执行某些代码对象期间所需的数据和值。像通常一样，python 确实为我们提供了一些函数用来检查 frame 对象，如 list 6.1 中使用的 sys._getframe() 函数。

Listing 6.1: Accessing frame objects

```python
>>> import sys
>>> f = sys._getframe()
>>> f
<frame object at 0x10073ed48>
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
NameError: name 'f_' is not defined
>>> dir(f)
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__',
'__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__le__',
'__lt__', '__ne__', '__new__','__reduce__', '__reduce_ex__', '__repr__',
'__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'clear',
'f_back', 'f_builtins', 'f_code', 'f_globals', 'f_lasti', 'f_lineno',
'f_locals', 'f_trace']
```

在代码对象可以被执行之前，必须创建一个 frame 对象，然后在 frame 对象中执行该代码对象。这样的 frame 对象包含可执行代码对象 (局部，全局和内置) 所需的所有命名空间，对当前执行线程的引用，用于执行 (evaluating) 字节码的堆栈以及对于可执行字节码来说其它的重要内部信息。为了更好地认识 frame 对象，我们可以看一下 Include/frame.h (此结构体实际在 Include/frameobject.h 中) 模块中 frame 对象数据结构的定义，如 list 6.2 所示。

Listing 6.2: Frame object definition in the vm

```c
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back; 		/* previous frame, or NULL */
    PyCodeObject *f_code; 		/* code segment */
    PyObject *f_builtins; 		/* builtin symbol table (PyDictObject) */
    PyObject *f_globals; 		/* global symbol table (PyDictObject) */
    PyObject *f_locals; 		/* local symbol table (any mapping) */
    PyObject **f_valuestack; 	/* points after the last local */
    PyObject **f_stacktop;
    PyObject *f_trace; 			/* Trace function */
    
    /* fields for handling generators*/
    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;
    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;
    
    int f_lasti; 				/* Last instruction if called */
    int f_lineno; 				/* Current line number */
    int f_iblock; 				/* index in f_blockstack */
    char f_executing; 			/* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1]; 	/* locals+stack, dynamically sized */
} PyFrameObject;
```

frame 中的字段以及文档不难理解，但我们提供了更多有关这些字段以及它们与字节码执行之间关系的细节。

1. f_back：这个字段是在当前代码对象之前执行的代码对象的 frame 的引用。给定一组 frame 对象，这些 frame 的 f_back 字段一起形成一个 stack of frames ，最终返回至 initial frame 。然后，此 initial frame 在的 f_back 字段的值为 NULL 。这种隐式的 stack of frames 形成了被我们称为调用栈 (call stack) 的东西。
2. f_code：此字段是对代码对象的引用。此代码对象包含了字节码 (bytecode) ，这些字节码在此 frame 的上下文中执行。

3. f_builtins：这是对内置命名空间的引用。该名称空间包含诸如 print，enumerate 等的名称及其对应的值。 
4. f_globals：这是对代码对象的全局命名空间的引用。
5. f_locals：这是对代码对象的局部命名空间的引用。如前所述，这些名称会在函数作用域内定义。当我们讨论 f_localplus 字段时，我们会看到 python 在使用局部定义的名称时所做的优化。 
6. f_valuestack：这是对 frame 执行栈 (evaluation stack) 的引用。回想一下，Python 虚拟机是基于堆栈的虚拟机，因此在字节码执行 (evaluation) 期间，将从堆栈的顶部读取值，并将字节码执行 (evaluation) 的结果存储在堆栈的顶部。该字段是在代码对象执行期间使用的堆栈。 frame 中的代码对象的堆栈大小 (stacksize) 提供了此数据结构可以扩展的最大深度。 
7. f_stacktop：顾名思义，该字段指向执行栈 (evaluation stack) 的下一个空闲的 slot 。创建一个新的 frame 时，该值会被设置为 value stack：这是堆栈上的第一个可用空间，此时堆栈上没有任何东西 (items) 。 
8. f_trace：此字段引用一个函数，该函数用于追踪 python 代码的执行。 
9. f_exc_type，f_exc_value，f_exc_traceback，f_gen：是用于记录 (book keeping) 的字段，以便能够干净地执行生成器代码。之后我们会对 python 生成器进行更多的讨论。

10. f_localplus：这是对数组的引用，该数组包含足够的空间来存储单元和局部 (cell and local) 变量。该字段为执行循环 (evaluation loop) 提供了一种机制，该机制可使用 LOAD_FAST 和 STORE_FAST 指令来优化变量与 name stack 之间的加载和存储。 LOAD_FAST 和 STORE_FAST 操作码比相应的 LOAD_NAME 和 STORE_NAME 操作码提供更快的访问，因为它们使用数组索引访问变量的值，并且此操作基本上会在恒定的时间内完成，不像对应的操作码那样，会搜索映射中与给定名称相关联的值。当我们讨论执行循环 (evaluation loop) 时，我们将会看到在 frame 建立过程中是如何设置此值的。 
11. f_blockstack：此字段是一个充当栈的数据结构的引用，该栈用于处理循环和异常处理。除了对虚拟机最重要的值堆栈 (value stack) 外，这是第二个堆栈，但这没有得到相应的重视。块堆栈 (block stack) ，异常和循环结构之间的关系非常复杂，我们将会在接下来的章节中对这个堆栈进行介绍。

### 6.1 Allocating Frame Objects

 frame 对象在 python 代码执行 (evaluation) 期间无处不在，每个代码块被执行的时候都需要一个 frame 对象来提供一些上下文信息。通过调用 Objects/frameobject.c 模块中的 PyFrame_New 函数来创建新的 frame 对象。这个函数被调用了很多次 ：每当一个代码对象被执行的时候，它都会被调用，减少调用这个函数开销的方法主要有两个，下面我们会简要的介绍一下这两个优化方法。

首先，代码对象具有一个 co_zombieframe 字段，该字段引用了一个惰性 frame 对象。当一个代码对象被执行的时候的时候，执行它的 frame 对象并不会立即被释放。该 frame 会被保留在 co_zombieframe 字段中，因此当下一个相同的代码对象被执行的时候，不需要花费时间为新的执行 frame 分配内存。 ob_type，ob_size，f_code，f_valuestack 字段会保留各自的值；f_locals，f_trace，f_exc_type，f_exc_value，f_exc_traceback，f_localplus  这些字段会保留分配的内存空间，但是局部变量的值会被清空。其余字段不保留对任何对象的引用。虚拟机使用的第二个优化方法是维护一个预分配 frame 对象的空闲列表 (free list) ，从这个空闲列表中可以获取 frame 用来执行代码对象。

frame 对象的源代码实际上是一个易读的，并且可以通过查看闭合的代码对象 (enclosed code object) 执行后如何重新分配已被分配的 frame 来了解 zombie frame 和自由列表 (free list) 的概念是如何实现的。list 6.3 中展示了重新分配 frame 的代码部分中有趣的部分。

Listing 6.3: Deallocating frame objects

```c
if (co->co_zombieframe == NULL)
	co->co_zombieframe = f;
else if (numfree < PyFrame_MAXFREELIST) {
    ++numfree;
    f->f_back = free_list;
    free_list = f;
}
else
	PyObject_GC_Del(f);
```

仔细观察发现，只有在进行递归调用时，即代码对象试图执行自身时，自由列表才会增长，因为仅在这个时间zombieframe 字段为 NULL 。使用 freelist 的这种微小优化有助于在某种程度上消除此类递归调用重复分配的内存。

本章不涉及与 frame 对象紧密相关的执行循环 (evaluation loop)，而是涵盖了 frame 对象的要点。虽然我们在上面的讨论中仍然遗漏了一些内容，但我们会在后续章节中对这些进行介绍。例如，

1. 当代码执行击中 return 语句时，值如何从一个 frame 传递到下一个 frame ？ 
2. 线程状态是什么，线程状态从何而来？ 
3. 当在执行 frame 中引发异常时，异常如何在 frame 栈中冒泡？等等。

我们在下一章中会介绍非常重要的解释器和线程状态数据结构，然后在后续各章中讨论执行循环 (evaluation loop)，在之后的讨论期间，上面这些问题将会得到解答。

