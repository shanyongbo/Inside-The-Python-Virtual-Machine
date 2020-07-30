---
layout: post
title:  "Inside The Python Virtual Machine --  12、Generators Behind the scenes"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第12章介绍部分的中文内容。

<!-- more -->



## 12. Generators: Behind the scenes.

生成器是 python 中真正美丽的概念之一。生成器函数是包含 yield 语句的函数，当调用生成器函数时，它将返回生成器。在 python 中，生成器的一种非常简单的用法是作为一个迭代器，它根据需要生成用于迭代的值。list 12.0 是生成器函数的一个非常简单的示例，该函数返回一个生成器，该生成器生成从 0 到 n 的值。

Listing 12.0: A simple generator

```python
def firstn(n):
    num = 0
    while num < n:
        v = yield num
        print(v)
        num += 1
```

firstn 是一个生成器函数，因此用一个值调用 firstn 函数不会像常规函数那样返回简单的值，而是会返回生成器对象，该对象捕获计算的 continuation 。然后，我们可以使用 next 函数从返回的生成器对象或生成器对象的 send 方法获取连续值，以将值发送到生成器对象。在本章中，我们对生成器对象的语义以及生成器的使用方式或使用方式不感兴趣。我们的兴趣在于在 CPython 的幕后如何实现生成器。我们对如何暂停计算然后随后恢复这种计算感兴趣。我们来看一下这个概念背后的数据结构和思想，令人惊讶的是它们并不太复杂。首先，我们来看一个生成器对象的 C 实现。

### 12.1 The Generator object

生成器对象定义如 list 12.1 所示，通过该定义可以直观地了解如何暂停或恢复生成器执行。我们可以看到，生成器对象包含一个 frame 对象（从 frame 一章中调用执行 frame ）和一个代码对象，这两个对象对于执行 python 字节码至关重要。

Listing 12.1: Generator object definition

```c
/* _PyGenObject_HEAD defines the initial segment of generator
and coroutine objects. */
#define _PyGenObject_HEAD(prefix) 												\
    PyObject_HEAD 																\
    /* Note: gi_frame can be NULL if the generator is "finished" */ 			\
    struct _frame *prefix##_frame; 												\
    /* True if generator is being executed. */ 									\
    char prefix##_running; 														\
    /* The code object backing the generator */ 								\
    PyObject *prefix##_code; 													\
    /* List of weak reference. */ 												\
    PyObject *prefix##_weakreflist; 											\
    /* Name of the generator. */ 												\
    PyObject *prefix##_name; 													\
    /* Qualified name of the generator. */ 										\
    PyObject *prefix##_qualname;

typedef struct {
    /* The gi_ prefix is intended to remind of generator-iterator. */
    _PyGenObject_HEAD(gi)
} PyGenObject;
```

以下内容包含生成器对象的主要属性。 
1. prefix##_frame：此字段引用 frame 对象。该 frame 对象包含生成器的代码对象，并且在该 frame 内执行生成器对象的代码对象。 
2. prefix##_running：这是一个布尔值字段，指示生成器是否正在运行。 
3. prefix##_code：此字段引用与生成器关联的代码对象。这是在生成器运行时执行的代码对象。 
4. prefix##_name：这是生成器的名称，在list 12.0 中，值是 firstn。 
5. prefix##_qualname：这是生成器的标准名称。大多数情况下，此值与前缀##_name的值相同。

**Creating generators**

调用生成器函数时，生成器函数不会运行到完成状态并返回值，而是会返回生成器对象。这是可能的，因为在生成器函数的编译期间设置了 CO_GENERATOR 标志，并且该标志在代码对象执行之前发生的设置过程中非常有用。

在执行该函数的代码对象的过程中，调用 _PyEval_EvalCodeWithName 可以执行一些设置。作为设置过程的一部分，将执行功能代码对象的 CO_GENERATOR 标志的检查，并且在设置该标志而不是调用求值循环函数的情况下，将创建一个生成器对象并将其返回给调用者。魔术发生在 _PyEval_EvalCodeWithName 的最后一个代码块，如 list 12.2 所示。

Listing 12.2: _PyEval_EvalCodeWithName returns a generator object when processing a code object with generator flag

```c
/* Handle generator/coroutine/asynchronous generator */
if (co->co_flags & (CO_GENERATOR | CO_COROUTINE | CO_ASYNC_GENERATOR)) {
    PyObject *gen;
    PyObject *coro_wrapper = tstate->coroutine_wrapper;
    int is_coro = co->co_flags & CO_COROUTINE;
    
    if (is_coro && tstate->in_coroutine_wrapper) {
        assert(coro_wrapper != NULL);
        PyErr_Format(PyExc_RuntimeError,
                     "coroutine wrapper %.200R attempted "
                     "to recursively wrap %.200R",
                     coro_wrapper,
                     co);
        goto fail;
    }
    
    /* Don't need to keep the reference to f_back, it will be set
    * when the generator is resumed. */
    Py_CLEAR(f->f_back);
    
    PCALL(PCALL_GENERATOR);
    
    /* Create a new generator that owns the ready to run frame
    * and return that as the value. */
    if (is_coro) {
        gen = PyCoro_New(f, name, qualname);
    } else if (co->co_flags & CO_ASYNC_GENERATOR) {
        gen = PyAsyncGen_New(f, name, qualname);
    } else {
        gen = PyGen_NewWithQualName(f, name, qualname);
    }
    if (gen == NULL)
        return NULL;
    
    if (is_coro && coro_wrapper != NULL) {
        PyObject *wrapped;
        Generators: Behind the scenes. 120
            tstate->in_coroutine_wrapper = 1;
        wrapped = PyObject_CallFunction(coro_wrapper, "N", gen);
        tstate->in_coroutine_wrapper = 0;
        return wrapped;
    }
    
    return gen;
}
```

从 list 12.2 中可以看到，生成器函数代码对象的字节码在调用生成器函数时从未执行，字节码仅在返回的生成器对象执行才会执行，我们接下来再看。

### 12.2 Running a generator

我们可以通过将生成器对象作为参数传递给下一个内置函数来运行它。这将导致生成器执行直到命中 yield 表达式，然后中止执行。这里对我们而言重要的问题是生成器如何能够捕获执行状态并随意更新执行状态。

从 list 12.1 回顾生成器对象定义，我们看到生成器有一个引用 frame 对象的字段，并且在创建生成器时 (如 list 12.2 所示) 将其填充。我们记得， frame 对象具有执行代码对象所需的所有状态，因此，通过引用该执行 frame ，生成器对象可以捕获其执行所需的所有状态。

现在我们知道了生成器对象是如何捕获执行状态的，接下来我们来解决如何恢复挂起的生成器对象的执行的问题，考虑到我们已经掌握的信息，这并不难解决。当使用生成器作为参数调用下一个内置函数时，下一个函数将取消引用生成器类型的 tp_iternext 字段，并调用该字段引用的任何函数。对于生成器对象，该字段引用一个函数 gen_iternext ，该函数简单地调用另一个函数 gen_send_ex，该函数执行恢复生成器对象执行的实际工作。回想一下，在创建生成器对象之前，所有初始化设置已由 _PyEval_EvalCodeWithName 函数执行，初始化了帧对象并正确初始化了变量，因此生成器对象的执行涉及到调用 PyEval_EvalFrameEx 并使用生成器中包含的帧对象
对象作为 frame 参数。然后，按照执行循环 (evaluation loop) 中有关章节的说明，执行 frame 中包含的代码对象的执行。

为了更深入地了解一个生成器函数，我们从 list 12.0 开始查看该生成器函数。从 list 12.0 中分解生成器功能将产生 list 12.3 中所示的一组字节码。

Listing 12.3: Disassembly of generator function from listing 12.0

```python
4 	0 LOAD_CONST 						1 (0)
	2 STORE_FAST 						1 (num)
5 	4 SETUP_LOOP 						34 (to 40)
	>> 6 LOAD_FAST 						1 (num)
	8 LOAD_FAST 						0 (n)
	10 COMPARE_OP 						0 (<)
	12 POP_JUMP_IF_FALSE 				38
6 	14 LOAD_FAST 						1 (num)
	16 YIELD_VALUE
	18 STORE_FAST 						2 (v)
7 	20 LOAD_GLOBAL 						0 (print)
	22 LOAD_FAST 						2 (v)
	24 CALL_FUNCTION 					1
	26 POP_TOP
8 	28 LOAD_FAST 						1 (num)
	30 LOAD_CONST 						2 (1)
	32 INPLACE_ADD
	34 STORE_FAST 						1 (num)
	36 JUMP_ABSOLUTE 					6
	>> 38 POP_BLOCK
	>> 40 LOAD_CONST 					0 (None)
	42 RETURN_VALUE
```

当 list 12.3 中显示的生成器函数的字节码的执行到达字节偏移量 16 处的 YIELD_VALUE 操作码时，该操作码会导致执行 (evaluation) 挂起并将堆栈顶部的值返回给调用方。通过暂停，我们的意思是退出了当前正在执行的 frame 的执行循环 (evaluation loop) ，但是未释放该 frame ，因为它仍被生成器对象引用，因此当以该 frame 作为其参数之一调用 PyEval_EvalFrameEx 时， frame 可以再次继续执行。 

Python 生成器不仅可以生成值，还可以使用生成器的 send 方法来使用值。这是可能的，因为 yield 是一个计算得出值的表达式。在带有值的生成器上调用 send 方法时，gen_send_ex 方法在调用执行 (evaluation) frame 对象之前将值放在生成器对象 frame 的执行堆栈 (evaluation stack) 上。在 list 12.3 中，紧随 YIELD_VALUE 指令的指令是 STORE_FAST，它将存储在堆栈顶部的任何值存储到赋值左侧的名称上。如果没有发送函数调用，则放在堆栈顶部的值是 “None” 。
