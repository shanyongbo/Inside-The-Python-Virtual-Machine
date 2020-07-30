---
layout: post
title:  "Inside The Python Virtual Machine --  2、The View From 30,000ft"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第二章介绍部分的中文内容。

<!-- more -->



## 2、The View From 30,000ft

本章从高层次上介绍了解释器如何执行 python 程序。在随后的章节中，我们将会放大难题的各个部分，并提供对这些内容更加详细的描述。无论 python 程序的复杂程度如何，这个过程都是相同的。 Yaniv Aknin 在 Python Internal series 这篇文章中对这个过程的提供了精彩解释。

给定一个 python 模块，如：test.py，可以将该模块作为参数传递给 python 解释器程序，从而可以在命令行中执行该模块，例如：$ python test.py。这只是调用 python 可执行文件的方式之一；我们还可以启动交互式解释器，把字符串当做代码执行等等，但是其他这些执行方法对我们而言并不重要。当在命令行上把模块作为参数传递给可执行部分 (executable) 时，图 2.1 阐述了所提供模块实际执行中涉及的各种活动的流程。

![image-20191030161358606](./images/image-20191030161358606.png)



python 可执行文件是 C 程序，就像其他任何 C 的程序 (例如 linux 内核或 C 中的 hello world 程序) 一样，因此，在调用 python 可执行文件时，几乎发生相同的过程。花一点时间来理解这一点，python 可执行程序只是另一个运行你自己的程序的程序。 C 与汇编语言或 llvm 之间的关系和上述的关系相同。一旦以模块名称作为参数调用 python 可执行程序，就会启动基于可执行文件运行平台的初始化标准程序，C 运行时执行所有初始化方法：加载库，检查或设置环境变量，然后像任何其他的 C 程序一样运行 python 可执行程序的 main 方法。python 可执行程序的 main 程序位于 ./Programs/python.c 文件中，它处理一些初始化操作，例如把传递给模块的程序命令行参数制作副本。然后，main 函数调用位于 ./Modules/main.c 中的 Py_Main 函数，该函数处理解释器程序初始化的过程：解析命令行参数和设置程序标记，读取环境变量，运行钩子，执行哈希随机化等等。在初始化过程中，会从 pylifecycle.c 调用 Py_Initialize 方法；它对解释器和线程状态的数据结构进行初始化操作，这是两个非常重要的数据结构。查看解释器和线程状态的数据结构定义可以为这些数据结构的方法提供上下文。解释器和线程状态只是具有指向一些字段的指针的结构体，这些字段包含程序执行所需的信息。list 2.1 中提供了解释器状态 typedef 结构体 (这并不完全正确，因为假定这是由 C 语言定义的类型) 。

Listing 2.1：解释器状态的数据结构

```c
typedef struct _is {
	struct _is *next;
	struct _ts *tstate_head;

	PyObject *modules;
	PyObject *modules_by_index;
	PyObject *sysdict;
	PyObject *builtins;
	PyObject *importlib;

	PyObject *codec_search_path;
	PyObject *codec_search_cache;
	PyObject *codec_error_registry;
	int codecs_initialized;
	int fscodec_initialized;

	PyObject *builtins_copy;
} PyInterpreterState;
```

任何使用 Python 语言的时间足够长的人都可以识别此结构中提到的一些字段 (sysdict，builtins 和 codec)*。

1.  *next 字段是对另一个解释器实例的引用，因为同一进程中可以存在多个 python 解释器。
2.  *tstate_head 字段指向执行的主线程：如果 python 程序是多线程的，则解释器由程序创建的所有线程共享，接下来就会讨论线程状态的结构。
3.  modules, modules_by_index, sysdict, builtins 和 importlib 字段意义是不言自明的，它们都被定义为 PyObject 的实例，在虚拟机中，PyObject 是所有 python 对象的根类型。随后的章节有更多关于 Python 对象的详情。
4.  与 codec* 相关的字段中包含了关于加载编码和位置的相关帮助信息，这些对于解码字节非常重要。

程序的执行必须在线程内进行。线程状态结构体中包含了线程执行 python 某些代码对象所需的所有信息，list 2.2 中展示了线程数据结构的一部分。

Listing 2.2: A cross-section of the thread state data structure

```c
typedef struct _ts {
	struct _ts *prev;
	struct _ts *next;
	PyInterpreterState *interp;

	struct _frame *frame;
	int recursion_depth;
	char overflowed;

	char recursion_critical;
	int tracing;
	int use_tracing;

	Py_tracefunc c_profilefunc;
	Py_tracefunc c_tracefunc;
	PyObject *c_profileobj;
	PyObject *c_traceobj;

	PyObject *curexc_type;
	PyObject *curexc_value;
	PyObject *curexc_traceback;

	PyObject *exc_type;
	PyObject *exc_value;
	PyObject *exc_traceback;
    
	PyObject *dict; /* Stores per-thread state */
	int gilstate_counter;

	...
} PyThreadState;
```

解释器和线程状态数据结构将在后续章节中详细讨论。初始化过程还设置了导入机制以及基本的 stdio。一旦完成所有初始化后，Py_Main 函数将调用也位于 main.c 模块中的 run_file 函数。

接下来是一系列函数调用：PyRun_AnyFileExFlags -> PyRun_SimpleFileExFlags -> PyRun_FileExFlags -> PyParser_ASTFromFileObject，这些调用组成了 PyParser_ASTFromFileObject 函数。PyRun_SimpleFileExFlags 函数调用创建了 \__main__ 命名空间，并在其中执行文件的内容。

它还会检查文件的 pyc 版本是否存在：pyc 文件只是一个包含正在执行文件的编译版本的文件。如果文件有 pyc 版本，则将尝试以二进制方式读取，然后运行。在这种情况下，如果没有 pyc 文件，则会调用 PyRun_FileExFlags 方法等等。 

PyParser_ASTFromFileObject 函数调用 PyParser_ParseFileObject，后者读取模块内容并从中构建一个解析树（parse tree）。然后创建好的解析树会被传递到 PyParser_ASTFromNodeObject 之中，再之后 PyParser_ASTFromNodeObject 继续从该解析树创建抽象语法树 (abstract syntax tree) 。

然后将生成的 AST 会传递给 run_mod 函数。该函数调用 PyAST_CompileObject 函数，它会从 AST 创建代码对象。请注意，在调用 PyAST_CompileObject 的过程中生成的字节码是通过一个简单的 peephole 优化器传递的，该优化器在创建代码对象之前对生成的字节码进行低悬挂 (low hanging) 优化。

然后 run_mod 函数从代码对象上的 ceval.c 文件中调用 PyEval_EvalCode。这会导致另外的一系列函数调用：PyEval_EvalCode -> PyEval_EvalCode -> _ PyEval_EvalCodeWithName -> _ PyEval_EvalFrameEx。代码对象以某种形式作为参数传递给这些函数。 

_PyEval_EvalFrameEx 是处理代码对象执行的常规解释器循环 (interpreter loop) 。但是，它不仅仅把代码对象作为参数来调用，具有引用代码对象字段的 frame 对象也是其参数之一。该 frame 对象提供了代码对象执行的上下文。这里发生的事情可以简单描述为：解释器循环从指令数组中持续不断的读取指令计数器然后指向的下一条指令。然后执行该指令：在进程中从 value stack 中添加或删除对象，直到数组中不再有需要执行的指令或发生破坏该循环的异常事件为止。

Python 提供了一组函数，可用于探索实际的代码对象。例如，可以将一个简单程序编译为一个代码对象，然后将其反汇编以获取由 python 虚拟机执行的操作码，如 list 2.3 所示。

Listing 2.3: Disassembling a python function

```python
>>> def square(x):
... return x*x
...

>>> dis(square)
2 			0 LOAD_FAST 		0 (x)
			2 LOAD_FAST 		0 (x)
			4 BINARY_MULTIPLY
			6 RETURN_VALUE
```

./Include/opcodes.h 头文件中包含 python 虚拟机的所有指令/操作码的完整列表。从概念上讲，这些操作码非常简单。以 list 2.3 中的操作码为例，其中包含四个指令：

LOAD_FAST 将其参数的值 (即 x) 加载到计算 (值) 堆栈上。 python 虚拟机是基于堆栈的虚拟机，因此这意味着从堆栈中获取操作码进行计算的值，会被放回到堆栈上，以供其他操作码进一步使用。

然后 BINARY_MULTIPLY 操作码从值堆栈中弹出两个元素，对两个值都执行二进制乘法，并将二进制乘法的结果放回到值堆栈上。 

RETURN VALUE 指令从堆栈中弹出一个值，将对象的返回值设置为该值，然后退出解释器循环。

从 list 2.3 的反汇编中可以明显看出，这种对解释器循环操作的简单解释遗漏了许多细节，这些细节将在后续章节中进行讨论。其中可能会包括一些悬而未决的问题。

在执行完所有指令之后，Py_Main 函数将继续执行，但是这次是围绕它开始清理的过程。就像在解释器启动期间调用 Py_Initialize 进行初始化一样，清理过程会调用 Py_FinalizeEx 进行清理工作。此清理过程包括等待线程退出，调用所有退出钩子以及释放由解释器分配的仍在使用的所有内存。

上面的描述是 python executable 执行 python 程序所经过的过程的高级 (high-level) 描述。正如前面所说的那样，有许多细节尚待回答。在接下来的章节中，我们将深入探讨涉及的每个阶段，并尝试提供每个阶段的细节。我们将从下一章中的编译过程的描述入手。