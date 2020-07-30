---
layout: post
title:  "Inside The Python Virtual Machine --  9、The evaluation loop, ceval.c"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第9章介绍部分的中文内容。

<!-- more -->


## 9. The evaluation loop, ceval.c

我们终于到了虚拟机的核心部分：在这里，虚拟机遍历代码对象的 python 字节码指令并执行此类指令。这是通过使用一个 for 循环来实现的，该循环遍历每种类型上的操作码切换以运行所需的执行代码。。 Python/ceval.c 模块约有 5411行，实现了所需的大多数功能：此模块的核心是 PyEval_EvalFrameEx 函数，该函数约 3000 行，其中包含实际的执行循环 (evaluation loop) 。 PyEval_EvalFrameEx 函数是本章重点研究的重点。 

Python/ceval.c 模块提供了特定于平台的优化，例如 threaded gotos，以及 python 虚拟机优化，例如 opcode 预测。在本文中，我们更加关注虚拟机的流程和优化，因此我们可以忽略此处介绍的任何特定于平台的优化流程，只要它不超出我们对执行循环 (evaluation loop) 的解释即可。在这里，我们会进行比平时更详细地介绍，以便于对虚拟机的核心结构和工作方式提供可靠的解释。值得注意的是，操作码及其实现始终在不断变化，因此这里的描述可能会不太准确。

在执行字节码之前，必须执行许多内部操作，例如创建和初始化 frames ，设置变量以及初始化虚拟机变量 (例如指令指针) 。我们首先需要了解这些操作，然后在执行 (evaluation) 开始前必须对进程进行设置。

### 9.1 Putting names in place

如上所述，虚拟机的核心是 PyEval_EvalFrameEx 函数，该函数实际执行 python 字节码，但是在执行该字节码之前，必须进行许多设置以进行准备执行 (evaluation) 的上下文，如错误检查，帧创建和初始化等。这也是 Python/ceval.c 模块中 _PyEval_EvalCodeWithName 函数出现的地方。为了解释这些，我们假定我们正在用 list 9.0 中所示的模块内容。

Listing 9.0: Content of a simple module

```python
def test(arg, defarg="test", *args, defkwd=2, **kwd):
    local_arg = 2
    print(arg)
    print(defarg)
    print(args)
    print(defkwd)
    print(kwd)

test()
```

回想一下为代码块创建的代码对象；这些代码块可以是函数，模块等，因此对于具有上述内容的模块，我们可以安全地假设我们正在处理两个代码对象，一个用于模块，一个用于模块内定义的功能测试。

list 9.0 中的模块生成代码对象后，通过 Python/pythonrun.c 模块中函数调用链的执行来生成代码对象：run_mod -> PyEval_EvalCode -> PyEval_EvalCodeEx -> _ PyEval_EvalCodeWithName -> PyEval_EvalFrameEx 。目前，我们关注的是 _PyEval_EvalCodeWithName 函数，该函数具有 list 9.1 中所示的特征。此函数处理在 PyEval_EvalFrameEx 中的字节码执行 (evaluation) 之前所需的名称设置。但是，如 list 9.1 所示，通过查看 _PyEval_EvalCodeWithName 的函数特征，有人可能会问为什么这个函数与执行的模块对象相关而不是与实际函数有关。

Listing 9.1: _PyEval_EvalCodeWithName function signature

```c
static PyObject * _PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals, PyObject **args, int argcount, PyObject **kws, int kwcount,
		 PyObject **defs, int defcount, PyObject *kwdefs, PyObject *closure,
		 PyObject *name, PyObject *qualname)
```

为了解决这个问题，人们必须更多地思考代码块和代码对象，而不是函数或者模块。代码块可以具有_PyEval_EvalCodeWithName 函数特征中指定的任何参数，也可以不包含任何参数：函数恰好是代码块的一种更特定的类型，它提供了大多数的这些值。这意味着为模块代码对象执行 _PyEval_EvalCodeWithName 的情况不是很有趣，因为其中大多数参数都没有值。通过 CALL_FUNCTION 操作码进行 python 函数调用时，会发生一些有趣的情况。这也会导致在 Python/ceval.c 模块中也会调用 fast_function 函数。该函数从函数对象中提取函数参数，然后使用 _PyEval_EvalCodeWithName 函数执行所有必要的健全性检查 (sanity checks) 。 

_PyEval_EvalCodeWithName 是一个很大的函数，因此我们在此处不讨论它，但是它的大多数设置过程都非常简单。例如，回想一下我们提到的， frame 对象的 fastlocals 字段为局部命名空间提供了一些优化，并且非位置参数仅在运行时才完全知道；这基本上意味着，如果不进行仔细的错误检查，就无法填充 fastlocals 这个数据结构。因此，正是在通过 _PyEval_EvalCodeWithName 函数进行这个设置过程当中，一个 frame 的 fastlocals 字段所引用的数组才填充了全部的局部值。 _PyEval_EvalCodeWithName 调用时要进行的设置过程涉及的步骤如 list 9.1 中所示。

Listing 9.2: _PyEval_EvalCodeWithName setup steps

------

1. 初始化为代码对象执行提供上下文的 frame 对象。
3. 将关键字 \*dict* 添加到快速局部 frame 中。
4. 添加关键字参数到 \`fastlocals` 中.
5. 将非位置、非关键字参数的可变序列添加到\`fastlocals\`数组中 (示例模块中的 \*args 参数) 。
这些值一起存储在一个元组数据结构中。
5. 请检查提供给代码块的任何关键字参数是否为预期参数，并且没有提供两次。
6. 检查缺少的位置参数，如果发现任何错误，则抛出错误。
7. 将默认参数添加到 “fastlocals” 数组中 (在我们的示例模块中为 “defarg” ) 。
8. 将关键字默认值添加到\`fastlocals\` (在我们的示例模块中为\`defkwd\` ) 。
9. 初始化单元格变量的存储并将自由变量数组复制到框架中。
10. 做一些生成器相关的内部工作：我们在讨论生成器时会更加详细地介绍这一点。

------

### 9.2 The parts of the machine

使用所有名称后，将使用 frame 对象作为其参数之一调用 PyEval_EvalFrameEx 函数。大致看一下该函数，发现该函数由相当多的 C 宏和变量组成。宏是执行循环 (execution loop) 中不可或缺的一部分：宏提供了一种抽象方法，可以抽象出重复的代码，而不会产生函数调用的开销，因此，我们会描述其中一部分宏。在本节中，我们假设虚拟机未运行 C 优化，例如启用 computed gotos ，因此我们可以忽略与此类优化相关的宏。我们从描述一些对执行循环 (evaluation loop) 执行至关重要的变量开始。

1. **stack_pointer: 引用执行帧的值堆栈中的下一个空闲插槽 (slot) .

   ![](./images/image-20191130135751.png)

   ​		Figure 9.0: Stack pointer after a single value has been pushed onto the stack

1. \*next_instr：是指执行循环 (evaluation loop) 要执行的下一条指令。可以将其视为虚拟机的程序计数器。 Python 3.6 将此值的类型更改为 2 个字节的无符号 short ，以处理新的字节码指令大小。 
2. opcode：指当前正在执行的 python 操作码或将要执行的操作码。 
3. oparg：引用当前正在执行的操作码或要接受参数的操作码。 
4. why：评估循环是一个无限循环，由无限for循环：for (;;) 实现，因此循环需要一种机制来跳出循环并指定发生中断的原因。该值表示退出执行循环 (evaluation loop) 的原因。例如，如果代码块由于返回语句而退出循环，则此值将包含 WHY_RETURN 状态。 
5. fastlocals：指局部定义名称的数组。 
6. freevars：引用在代码块中使用但未在该代码块中定义的名称的列表。 
7. retval：指执行代码块后的返回值。 
8. co：引用包含将由执行循环 (evaluation loop) 执行的字节码的代码对象。 
9. names：引用执行 frame 的代码块中所有值的名称。 
10. consts：引用代码对象使用的常量。

------

**Bytecode instruction**

我们在代码对象一章中讨论了字节码指令的格式，但是它与我们的讨论非常相关，因此我们在这里重复对字节码指令格式的描述。假设我们正在使用 python 3.6 字节码，则所有字节码均为 16 位长。 Python VM 在我目前正在写这本书的机器上使用一点 endian 字节编码，因此 16 位代码的结构如下图所示，其中 opcode 占用 1 个字节，而 opcode 的参数占用第二个字节。

![](./images/image-20191126223951.png)

​						Bytecode instruction format showing opcode and oparg

提取操作码和参数涉及一些位操作，我们将在接下来的小节中看到。重要的是要注意，由于操作码现在是两个字节而不是一个字节，因此指令指针的操作属于指针操作。

------

以下宏在评估循环中起着非常重要的作用。 

1. TARGET (op) ：扩展到 case op 语句。这会将当前操作码与实现该操作码的代码块进行匹配。
2. DISPATCH：扩展以继续。它与下一个宏 FAST_DISPATCH 一起在执行操作码后处理执行循环 (evaluation loop) 的控制流。 
3. FAST_DISPATCH：扩展为跳转到执行 (evaluation) for 循环内的 fast_next_opcode 标签。

随着 Python 3.6 中标准的 2 字节操作码的引入，以下宏集用于处理代码访问。

1. INSTR_OFFSET() ：此宏将当前指令的字节偏移量提供给指令数组。这扩展为 (2 *(int)(next_instr -  first_instr))。 
2. NEXTOPARG()：这会将操作码和 oparg 变量更新为要执行的下一个字节码指令的操作码和参数的值。此宏扩展为以下代码段。

Listing 9.3: Expansion of the NEXTOPARG macro

```c
do { \
    unsigned short word = *next_instr; \
    opcode = OPCODE(word); \
    oparg = OPARG(word); \
    next_instr++; \
} while (0)
```

OPCODE 和 OPARG 宏处理用于提取操作码和参数的位操作。图 9.0 显示了字节码指令的结构，其中操作码的参数取低 8 位，操作码本身取高 8 位，因此 OPCODE 扩展为 ((word) ＆ 255) ，从而从字节码指令中提取最高有效字节，同时扩展为 ((word) >> 8) 的 OPARG 提取最低有效字节。

1. JUMPTO(x)：此宏扩展为 (next_instr = first_instr + (x) / 2)，并执行绝对跳转到字节码流中的特定偏移量。 
2. JUMPBY(x)：此宏扩展为 (next_instr + =(x) / 2)，并执行从当前指令偏移量到字节码指令流中另一点的相对跳转。 
3. PREDICT(op)：此操作码与 PREDICTED(op) 操作码一起实现 python 执行循环 (evaluation loop) 操作码预测。该操作码扩展为以下代码段。
4. PREDICTED(op)：此宏扩展为 PRED _##op: 。

Listing 9.4: Expansion of the PREDICT(op) macro

```c
do{ \
    unsigned short word = *next_instr; \
    opcode = OPCODE(word); \
    if (opcode == op){ \
        oparg = OPARG(word); \
        next_instr++; \
        goto PRED_##op; \
    } \
} while(0)
```

上面定义的最后两个宏可处理操作码预测。当执行循环 (evaluation loop) 遇到 PREDICT(op) 宏时，解释器会假定要执行的下一条指令是 op 。宏会检查这是否确实有效，如果有效则获取实际的操作码和参数，然后跳转到标签 PRED_##op，其中 ## 是实际操作码的占位符。例如，如果我们遇到了诸如 PREDICT(LOAD_CONST) 之类的预测，则如果该预测有效，则 goto 语句参数将为 PRED_LOAD_CONSTop 。通过检查 PyEval_EvalFrameEx 函数的源代码，可以找到扩展到 PRED_LOAD_CONSTop 的 PREDICTED(LOAD_CONST) 标签，因此，在成功预测该指令后，将跳转至该标签，否则将继续正常执行。这种预测节省了 switch 语句额外遍历所涉及的成本，否则常规代码执行会发生这种情况。

我们关注的下一组宏是堆栈操作宏，用于处理 frame 对象的值栈 (value stack) 中的值的放置和提取。这些宏非常相似，下面的代码片段显示了一些示例。

1. STACK_LEVEL()：这将返回堆栈上的 items 数量。宏扩展为 ((int) (stack_pointer - f -> f_valuestack)) 。 
2. TOP()：返回堆栈上的最后一项。这扩展为 (stack_pointer[-1]) 。 
3. SECOND()：这将返回堆栈上的倒数第二个项目。这扩展为 (stack_pointer[-2]) 。 
4. BASIC_PUSH(v)：这将 v 入栈。它扩展为 (* stack_pointer ++ =(v)) 。该宏的当前别名为 PUSH(v)。 
5. BASIC_POP()：将栈顶元素出栈。这扩展为 (*--stack_pointer)。当前的别名是 POP()。

我们关心的最后一组宏是那些处理局部变量操作的宏。这些宏 GETLOCAL 和 SETLOCAL 用于在 fastlocals 数组中获取和设置值。

1. GETLOCAL(i)：这扩展为 (fastlocals[i]) 。这处理从局部数组中获取局部定义的名称。 
2. SETLOCAL(i，value)：这将扩展为 list 9.5 中的代码段。此宏将局部数组的第 i 个元素设置为提供的值。

Listing 9.5: Expansion of the SETLOCAL(i, value) macro

```c
do { PyObject *tmp = GETLOCAL(i); \
    GETLOCAL(i) = value; \
    Py_XDECREF(tmp);
} while (0)
```

UNWIND_BLOCK 和 UNWIND_EXCEPT_HANDLER 与异常处理相关，我们将在后续部分中对其进行介绍。

### 9.3 The Evaluation loop

我们终于来到了虚拟机的核心：实际执行 (evaluted) 操作码的循环。实际循环的实现是非常反常的，因为这里实际上没什么特别的，只是永无休止的 for 循环和用于匹配操作码的大量 switch 语句。为了获得对该语句的具体理解，我们看一下 list 9.6中 hello world 函数的执行。

Listing 9.6: Simple hello world python function

```python
def hello_world():
	print("hello world")
```

list 9.6 显示了list 9.7 中函数的反汇编，我们将展示这组字节码如何通过执行 (evaluation) 开关循环。

Listing 9.7: Disassembly of function in listing 9.6

```python
LOAD_GLOBAL 						0 (0)
LOAD_CONST 							1 (1)
CALL_FUNCTION 						1 (1 positional, 0 keyword pair)
POP_TOP
LOAD_CONST 							0 (0)
RETURN_VALUE
```

![](./images/image-20191130141014.png)

​				Figure 9.1: Evaluation path for LOAD_GLOBAL and LOAD_CONST instructions

图 9.1 显示了 LOAD_GLOBAL 和 LOAD_CONST 指令的执行 (evaluation) 路径。图 9.2 的两个图像中的第二个和第三个块表示在执行循环 (evaluation loop) 的每次迭代中执行的整理任务 (housekeeping tasks) 。在上一章的解释器和线程状态中讨论了 GIL 和信号处理检查，正是在这些检查期间，执行线程可能放弃对 GIL 的控制权，让另一个线程执行。 fast_next_opcode 是紧随 GIL 和信号处理代码之后的代码标签，当循环希望跳过先前的检查时，这些代码将作为跳转目标，就像我们在查看 LOAD_CONST 指令时所看到的那样。

第一条指令 LOAD_GLOBAL 由 switch 语句的 LOAD_GLOBAL case 语句执行 。像其他操作码一样，此操作码的实现是一系列 C 的语句和函数调用，如 list 9.8 所示。操作码的实现将全局或内置命名空间中由给定名称标识的值加载到执行堆栈中。 oparg 是元组的索引，其中包含代码块中使用的所有名称 co_names 。

Listing 9.8: LOAD_GLOBAL implementation

```c
PyObject *name = GETITEM(names, oparg);
PyObject *v;
if (PyDict_CheckExact(f->f_globals)
    && PyDict_CheckExact(f->f_builtins)){
    v = _PyDict_LoadGlobal((PyDictObject *)f->f_globals,
                           (PyDictObject *)f->f_builtins,
                           name);
    if (v == NULL) {
        if (!_PyErr_OCCURRED()) {
            /* _PyDict_LoadGlobal() returns NULL without raising
            * an exception if the key doesn't exist */
            format_exc_check_arg(PyExc_NameError,
                                 NAME_ERROR_MSG, name);
        }
        goto error;
    }
    Py_INCREF(v);
}
else {
    /* Slow-path if globals or builtins is not a dict */
    /* namespace 1: globals */
    v = PyObject_GetItem(f->f_globals, name);
    if (v == NULL) {
        if (!PyErr_ExceptionMatches(PyExc_KeyError))
            goto error;
        PyErr_Clear();
        /* namespace 2: builtins */
        v = PyObject_GetItem(f->f_builtins, name);
        if (v == NULL) {
            if (PyErr_ExceptionMatches(PyExc_KeyError))
                format_exc_check_arg(
                PyExc_NameError,
                NAME_ERROR_MSG, name);
            goto error;
        }
    }
}
```

如果 LOAD_GLOBAL 操作码是 dict 对象，则 LOAD_GLOBAL 操作码的查找算法首先尝试从 f_globals 和 f_builtins 字段中加载名称，否则它将尝试从 f_globals 或 f_builtins 对象中获取与名称相关联的值，并假设它们实现了某种协议来获取与给定名称相关的值。如果找到该值，则使用 PUSH(v) 将该值加载到执行堆栈 (evaluation stack) 上，否则会报错，并且会跳转到错误代码标签以进行错误处理。如流程图所示，将此值压入执行堆栈 (evaluation stack) 后，将调用 DISPATCH() 宏，该宏是 Continue 语句的别名。

图 9.1 中标记为 2 的第二张图显示了 LOAD_CONST 的执行。list 9.9 是 LOAD_CONST 操作码的实现。

Listing 9.9: LOAD_CONST opcode implementation

```c
PyObject *value = GETITEM(consts, oparg);
    Py_INCREF(value);
    PUSH(value);
    FAST_DISPATCH();
```

这将通过常规设置进行，即 LOAD_GLOBAL，但在执行后，将调用 FAST_DISPATCH() 而不是 DISPATCH() 。这将导致跳转到 fast_next_opcode 代码标签，在该标签处循环执行将继续跳过信号，并在下一次迭代时进行 GIL 检查。具有执行 C 函数调用的实现的操作码由 DISPATCH 宏组成，而诸如 LOAD_GLOBAL 之类的操作码在其实现中未进行 C 函数调用的操作码则使用 FAST_DISPATCH 宏。这意味着仅在执行执行 C 函数调用的操作码后才能放弃 GIL 。

![](./images/image-20191130141551.png)

​						Figure 9.2: Evaluation path for CALL_FUNCTION and POP_TOP instruction

下一个执行的操作码是 CALL_FUNCTION 操作码，如图 9.2 中的第一张图所示。当仅在调用中使用位置参数进行函数调用时，编译器将发出此操作码。list 9.10 显示了此操作码的实现。操作码实现的核心是 call_function( ＆sp, oparg, NULL) 。 oparg 是传递给该函数的参数数量，而 call_function 函数从执行堆栈 (evaluation stack ) 中读取该数目的值。

Listing 9.10: CALL_FUNCTION opcode implementation

```c
PyObject **sp, *res;
PCALL(PCALL_ALL);
sp = stack_pointer;
res = call_function(&sp, oparg, NULL);
stack_pointer = sp;
PUSH(res);
if (res == NULL) {
	goto error;
}
DISPATCH();
```

图 9.2 的图 4 中显示的下一条指令是 POP_TOP 指令，该指令从执行堆栈 (evaluation stack) 的顶部删除单个值，它清除了上一个函数调用放置在堆栈上的所有值。

![](./images/image-20191130141800.png)

​				Figure 9.3: Evaluation path for LOAD_CONST and RETURN_VALUE instruction

下一组指令是图 9.3 的图 5 和图 6 中所示的 LOAD_CONST 和 RETURN_VALUE 。 LOAD_CONST 操作码将 None 值加载到执行堆栈 (evaluation stack) 上，以供 RETURN_VALUE 使用。当 python 函数未明确返回任何值时，这些总是在一起。我们已经研究了 LOAD_CONST 指令的机制。 RETURN_VALUE 指令将堆栈的顶部弹出到 retval 变量中，将 WHY 状态代码设置为 WHY_RETURN ，然后跳转到 fast_block_end 代码标签。从那里继续执行，退出 for 循环，然后将 retval 变量的值返回给调用函数。

请注意，我们查看的许多代码片段都有 goto error 跳转，但是到目前为止，我们刻意讨论了错误，排除了异常。我们将在下一章中讨论异常处理。尽管本节介绍的功能相当琐碎，但它在执行字节码指令时封装了执行循环 (evaluation loop) 的主要行为。任何其他操作码可能会有一些稍微复杂的实现，但是执行的本质与上述相同。

接下来，我们看一下 python 虚拟机支持的其他一些有趣的操作码。

### 9.4 A sampling of opcodes

python 虚拟机有大约 157 个操作码，因此我们随机选择一些操作码并进行解构，以更加了解这些操作码的功能。下面是这些操作码中的一部分：

1. MAKE_FUNCTION：顾名思义，操作码根据执行堆栈 (evaluation stack) 上的值创建一个函数对象。思考一个包含 list 9.11 所示功能的模块。

Listing 9.11: Function definitions in a module

```python
def test_non_local(arg, *args, defarg="test", defkwd=2, **kwd):
    local_arg = 2
    print(arg)
    print(defarg)
    print(args)
    print(defkwd)
    print(kwd)
    
def hello_world():
	print("Hello world!")
```

从已编译模块的代码对象反汇编可以得到 list 9.12 中所示的字节码指令集。

Listing 9.11: Disassembly of code object from listing 9.11

```python
0 LOAD_CONST 				8 (('test',))
2 LOAD_CONST 				1 (2)
4 LOAD_CONST 				2 (('defkwd',))
6 BUILD_CONST_KEY_MAP 		1
8 LOAD_CONST 				3 (<code object test_non_local at 0x109eead0\
0, file "string", line 17>)
10 LOAD_CONST 				4 ('test_non_local')
12 MAKE_FUNCTION 			3
14 STORE_NAME 				0 (test_non_local)
16 LOAD_CONST 				5 (<code object hello_world at 0x109eeae80,\
file "string", line 45>)
18 LOAD_CONST 				6 ('hello_world')
20 MAKE_FUNCTION 			0
22 STORE_NAME 				1 (hello_world)
24 LOAD_CONST 				7 (None)
26 RETURN_VALUE
```

我们可以看到，MAKE_FUNCTION 操作码在一系列字节码指令中出现了两次：模块中的每个函数定义各一次。 MAKE_FUNCTION 的实现创建了一个函数对象，然后使用函数定义的名称将函数存储在局部命名空间中。要注意，定义了默认参数后，会将默认参数压入堆栈。 MAKE_FUNCTION 的实现通过使用位掩码的 oparg (and’ing the oparg with a bitmask) 并从堆栈中弹出值来消耗这些值。

Listing 9.12: MAKE_FUNCTION opcode implementation

```c
TARGET(MAKE_FUNCTION) {
    PyObject *qualname = POP();
    PyObject *codeobj = POP();
    PyFunctionObject *func = (PyFunctionObject *)
    	PyFunction_NewWithQualName(codeobj, f->f_globals, qualname);
    
    Py_DECREF(codeobj);
    Py_DECREF(qualname);
    if (func == NULL) {
    	goto error;
    }
    
    if (oparg & 0x08) {
        assert(PyTuple_CheckExact(TOP()));
        func ->func_closure = POP();
    }
    if (oparg & 0x04) {
        assert(PyDict_CheckExact(TOP()));
        func->func_annotations = POP();
    }
    if (oparg & 0x02) {
        assert(PyDict_CheckExact(TOP()));
        func->func_kwdefaults = POP();
    }
    if (oparg & 0x01) {
        assert(PyTuple_CheckExact(TOP()));
    	func->func_defaults = POP();
    }
    PUSH((PyObject *)func);
    DISPATCH();
}
```

上面的标志表示以下内容。 

1. 0x01：堆栈上按位置顺序排列的默认参数对象的元组。 
2. 0x02：堆栈上关键字参数默认值的字典。
3. 0x04：堆栈上的一个注释字典。
4. 0x08：堆栈上一个闭合的包含自由变量的元组。

实际上创建函数对象的 PyFunction_NewWithQualName 函数在 Objects/funcobject.c 模块中实现，其实现非常简单。该函数初始化一个函数对象并在该函数对象上设置值。

2.LOAD_ATTR：此操作码处理诸如 x.y 之类的属性引用。假设我们有一个实例对象 x ，则诸如 x.name 之类的属性引用将转换为 list 9.13 中所示的一系列操作码。

Listing 9.13: Opcodes for an attribute reference

```python
24 LOAD_NAME 				1 (x)
26 LOAD_ATTR 				2 (name)
28 POP_TOP
30 LOAD_CONST 				4 (None)
32 RETURN_VALUE
```

LOAD_ATTR 操作码实现非常简单，如 list 9.14 所示。

Listing 9.14: LOAD_ATTR opcode implementation

```c
TARGET(LOAD_ATTR) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = TOP();
    PyObject *res = PyObject_GetAttr(owner, name);
    Py_DECREF(owner);
    SET_TOP(res);
    if (res == NULL)
    	goto error;
    DISPATCH();
}
```

PyObject_GetAttr 函数是我们在对象一章中介绍的函数。任何在对象的 tp_getattro 属性中的值都会被该函数取消引用，并使用该函数将对象属性的值加载到值堆栈的顶部。

3.CALL_FUNCTION_KW：此操作码的功能与前面讨论的 CALL_FUNCTION 操作码非常相似，但用于带有关键字参数的函数调用。该操作码的实现在 list 9.15 中。请注意，CALL_FUNCTION 操作码实现的主要变化之一是：当调用 call_function 时，names组成的元组 (a tuple of names) 现在会作为参数之一进行传递。

Listing 9.15: CALL_FUNCTION_KW opcode implementation

```c
PyObject **sp, *res, *names;

names = POP();
assert(PyTuple_CheckExact(names) && PyTuple_GET_SIZE(names) <= oparg);
PCALL(PCALL_ALL);
sp = stack_pointer;
res = call_function(&sp, oparg, names);
stack_pointer = sp;
PUSH(res);
Py_DECREF(names);

if (res == NULL) {
	goto error;
}
```

名称是函数调用的关键字参数，它们在 _PyEval_EvalCodeWithName 中用于在执行函数的代码对象之前初始化操作。

这限制了我们对执行循环 (evaluation loop) 的解释。正如我们看到的那样，执行循环 (evaluation loop) 背后的概念并不复杂：每个操作码都是用 C 进行定义和实现，这些实现就是实际的功能。我们没有涉及的一个非常重要的区域就是异常处理和块堆栈，这是我们在下一章中将会看到的两个紧密相关的概念。

