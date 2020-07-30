---
layout: post
title:  "Inside The Python Virtual Machine --  7、Interpreter and Thread States"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第7章介绍部分的中文内容。

<!-- more -->


## 7. Interpreter and Thread States

如前所述，在 python 解释器初始化的过程中，其中的一个步骤就是解释器状态和线程状态数据结构的初始化。在本章中，我们将会详细研究这些数据结构并解释这些数据结构的重要性。

### 7.1 The Interpreter state

pylifecycle.c 模块中的 Py_Initialize 函数是在初始化 python 解释器时调用的函数之一。该函数处理 python 运行时的创建以及解释器状态和线程状态数据结构的初始化等。

解释器状态是一个非常简单的数据结构，它捕获由 python 进程中的一系列协作执行线程共享的全局状态。list 7.0 中提供了此数据结构定义中的代码片段，从而进一步了解这个重要的数据结构。

Listing 7.0: Cross-section of the interpreter state data structure

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
    ...
    PyObject *builtins_copy;
    PyObject *import_func;
} PyInterpreterState
```

只有熟悉目前为止所有章节内容并且使用了很长时间 python 的人，才可能对 list 7.0 中显示的字段感到熟悉。下面我们再次对解释器状态数据结构中的一些字段进行讨论。

* \*next：单个 OS 进程中运行的 Python 可执行程序可以有多个解释器状态。这个 \*next 字段引用 python 进程中的另一个解释器状态数据结构 (如果存在的话) ，它们形成了一个解释器状态的链表，如图 7.0 所示。每个解释器状态都有它自己的一组变量，这些变量会被引用该解释器状态的执行线程所使用。但是，该进程中的所有解释器线程共享该进程可用的内存和全局解释器锁 (Global Interpreter Lock) 。
* \*tstate_head：此字段引用当前正在执行的线程的线程状态，或者在多线程的情况下，引用当前持有全局解释器锁 (GIL) 的线程。这是一个映射到 ~~executin~~ 正在执行的 (executing) 操作系统线程的数据结构。

![](./images/image-20191130122630.png)

其余字段是由解释器状态的所有协作线程 (cooperating threads) 共享的变量。modules 字段是已安装的python 模块的表，稍后我们将看到：在讨论 import system 以及 builtins 字段是内置 sys 模块的引用时，解释器是如何找到这些模块的。该模块的内容是 len，enumerate 等内置函数的集合，而 Python/bltinmodule.c 模块包含此模块大部分内容的实现。 importlib 是一个引用 import 机制实现的字段，当我们详细讨论 import system 的时候，我们会说明更多的内容。 \*codec_search_path，\*codec_search_cache，\*codec_error_registry，\*codecs_initialized 和 \*fscodec_initialized 是 python 用来编码和解码字节和文本的编解码相关的字段。这些字段的值用于查找此类编解码器以及处理可能与使用此类编解码器相关的错误。一个正在执行的 python 程序由一个或多个执行线程组成。解释器必须为每个执行线程维护一组状态，并且能够通过为每个执行线程维护一个线程状态数据结构来做到上面说的这些。下面我们看一下这个数据结构。

### 7.2 The Thread state

直接查看 list 7.1 中所示的线程状态数据结构，我们可以看到线程状态数据结构是比解释器状态数据结构更复杂的数据结构。

Listing 7.2: Cross-section of the thread state data structure

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
    
    ...
        
} PyThreadState;
```

线程状态数据结构的下一个和上一个字段的引用是在给定线程状态之前和之后创建的线程状态。这些字段形成一个由线程状态组成的双向链表，这些线程状态共享一个解释器状态。 interp 字段是线程状态所属的解释器状态的引用。该 frame 字段引用当前的执行 frame ；当执行的代码对象更改时，此字段引用的值也会更改。

顾名思义，recursion_depth 指定了在递归调用期间 frame 栈应达到的深度。堆栈溢出时设置溢出标志，堆栈溢出后，线程允许再执行 50 次调用来启用某些清理操作。 recursion_critical 标志用于向线程发送信号，表明正在执行的代码不应溢出。 tracing 和 use_tracing 标志与用于跟踪线程执行的功能有关。就像在后续章节中将看到的那样， \*curexc_type，\*currexc_value，\*curexc_traceback，\*exc_type，\*exc_value 和 \*curexc_traceback 是在异常处理过程中使用的字段。

理解线程状态和实际线程之间的区别很重要。线程状态只是一个数据结构，它封装了正在执行的线程的某些状态。在运行的 python 进程中，每个线程状态都与本机 OS 线程相关联。图 7.1 说明了这种关系。我们可以清楚地看到，单个 python 进程是至少是一个解释器状态的宿主，而每个解释器状态是一个或多个线程状态的宿主，并且这些线程状态都映射到操作系统的执行线程。

![](./images/image-20191130132830.png)

​								Figure 7.1: Relationship between interpreter state and thread states

操作系统线程和相关的 python 线程状态是在解释器初始化期间或由线程模块调用时创建的。即使在 python 进程中存在多个线程，但在任何给定时间点上，只有一个线程可以主动执行 CPU 绑定的任务。这是因为执行线程必须持有 GIL 才能在 python 虚拟机中执行字节码。如果不了解臭名昭著的 GIL 的概念，本章的内容将不够完整，因此我们将在下面继续对 GIL 进行介绍。

**Global Interpreter Lock - GIL**

尽管 python 线程是操作系统线程，但是除非该线程持有GIL，否则该线程是无法执行 python 字节码的。操作系统可能会调度一个不运行 GIL 的线程，但是正如我们看到的那样，此类线程实际上可以做的就是等待获取 GIL ，并且只有当它持有 GIL 时，它才能执行字节码。下面我们来看一下整个过程。

------

**The Need for a GIL**

在开始对 GIL 进行任何讨论之前，有一个问题值得讨论，为什么我们需要一个可能对线程产生不利影响的全局锁？有很多原因都说明了 GIL 是很重要的。但是，首先最重要的是要了解到 GIL 是 CPython 的实现细节，而不是实际的语言细节，在 Java 虚拟机上实现的 python，Jython 就没有 GIL 的概念。 GIL 存在的主要原因就是为了简化 CPython 虚拟机的实现。实现单个全局锁比实现细粒度锁要容易得多，而且核心开发人员也选择这样去做。但是，已经有一些项目在 python 虚拟机中实现细粒度的锁，但是这些项目有时会减慢单线程程序的速度。执行某些任务时，全局锁还提供了非常需要的同步功能。CPython 用于内存管理的是引用计数机制，如果没有 GIL 的概念，则可能使两个线程交错引用计数的增加和减少，从而导致内存处理方面严重的问题。使用全局锁的另一个原因是，CPython 调用的某些 C 库本来就不是线程安全的，因此在使用它们时候需要某种同步。

------

在解释器启动时，将创建一个执行的主线程，并且由于没有其他线程，因此 GIL 没有争用，因此主线程不会费心去获取锁。当使用 python 线程模块生成另一个线程时，GIL 就会开始起作用。list 7.3 中的代码片段来自Modules/_threadmodule.c，它提供了有关在创建新线程时该进程该如何处理的方法。

Listing 7.3: Cross-section of code for creating new thread

```c
boot->interp = PyThreadState_GET()->interp;
boot->func = func;
boot->args = args;
boot->keyw = keyw;
boot->tstate = _PyThreadState_Prealloc(boot->interp);
if (boot->tstate == NULL) {
    PyMem_DEL(boot);
    return PyErr_NoMemory();
}
Py_INCREF(func);
Py_INCREF(args);
Py_XINCREF(keyw);
PyEval_InitThreads(); /* Start the interpreter's thread-awareness */
ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);
```

list 7.3 中的代码片段来自 thread_PyThread_start_new_thread 函数，该函数被调用以创建新的线程。 boot 是一个数据结构，其中包含新线程需要执行的所有信息。 _PyThreadState_Prealloc 函数被调用为尚未创建的线程创建新的线程状态。在实际创建线程之前，执行主线程必须去获取 GIL；调用 PyEval_InitThreads 即可解决这个问题。现在解释器中线程被唤醒，并且主线程持有 GIL ，PyThread_start_new_thread 将会被调用来创建新的操作系统线程。当生成新的线程的时候，会把该线程在活动时应调用的函数传递给该线程。在这种情况下，这个需要被调用的函数是 Modules/_threadmodule.c 模块中的 _tbootstrap 函数。list 7.4 展示了这个函数的片段。

Listing 7.4: Cross-section of thread bootstrapping function

```c
static void t_bootstrap(void *boot_raw){
    struct bootstate *boot = (struct bootstate *) boot_raw;
    PyThreadState *tstate;
    PyObject *res;
    
    tstate = boot->tstate;
    tstate->thread_id = PyThread_get_thread_ident();
    _PyThreadState_Init(tstate);
    PyEval_AcquireThread(tstate);
    nb_threads++;
    res = PyEval_CallObjectWithKeywords(
    	boot->func, boot->args, boot->keyw);
    ...
```

注意 list 7.4 中对 PyEval_AcquireThread 函数的调用。 PyEval_AcquireThread 函数是在 Python/ceval.c 模块中定义的，它调用 take_gil 函数，后者是试图获取 GIL 的实际函数。以下的文本中引用了源文件中提供的有关此过程的说明。

GIL 只是一个布尔变量 (gil_locked)，其访问受互斥锁保护 (gil_mutex)，其更改是由条件变量 (gil_cond) 发出信号。 gil_mutex 的使用时间很短，因此几乎没有竞争。在 GIL-holding 线程中，主循环 (PyEval_EvalFrameEx) 必须能够根据另一个线程的需要释放 GIL 。因此使用一个易变的布尔变量 (gil_drop_request)，该变量在每次执行循环时都会检查。在 gil_cond 上等待微秒级别的间隔后，该变量将会被重置。[实际上，使用另一个可变布尔变量 (eval_breaker) ，将多个条件合并为一个条件。可变布尔值足以作为线程间信号传递的手段，因为 Python 仅在高速缓存一致性体系结构上运行。] 想要获取 GIL 的线程首先需要在设置 gil_drop_request 之前让其经过给定的时间 (微秒级别的间隔) 。这鼓励一定周期后进行切换，但是由于操作码可能花费任意时间执行，因此不强制执行。用户可以使用 Python API 中的 sys.getswitchinterval() 和 sys.setswitchinterval() 这两个方法来读取和修改时间间隔值。当一个线程释放 GIL 并设置了 gil_drop_request 时，该线程将确保安排另外一个等待 GIL 的线程。它通过等待一个条件变量 (switch_cond) 直到 gil_last_holder 的值更改为自己的线程状态指针以外的值来表明另一个线程能够使用 GIL。这意味着要禁止多核计算机上等待时间的有害行为，在多核计算机上，一个线程会随机性的释放 GIL，但仍在运行并最终成为第一个重新获取 GIL 的对象，这使得 ”时间片“ 比预期的长得多。

以上对于新产生的线程意味着什么？list 7.4 中的 t_bootstrap 函数调用 PyEval_AcquireThread 函数，该函数处理对 GIL 的请求。因此，对发出此请求时会发生什么情况的一般解释是，假设 A 是持有 GIL 的执行主线程，而 B 是正在产生的新线程。

1. 生成 B 时，将调用 take_gil 。这将检查是否设置了条件变量 gil_cond。如果未设置，则线程开始等待。 
2. 等待时间过后，将设置 gil_drop_request。 
3. 在执行循环 (evaluation loop) 上执行的线程 A 检查循环的每次迭代是否设置了 gil_drop_request 变量。 
4. 线程 A 在检测到已设置 gil_drop_request 变量时删除 GIL，然后还设置了 gil_cond 变量。 
5. 线程 A 还等待另一个变量 switch_cond，直到 gil_last_holder 的值设置为除线程 A 的线程状态指针以外的值，该值指示另一个线程已采用 GIL 。 
6. 线程 B 现在具有 GIL ，可以继续执行字节码。 
7. 线程 A 等待给定时间，设置 gil_drop_request ，然后循环继续。

------

**GIL and Performance**

GIL 就是为什么大多数情况下，在 python 中增加在一个 CPU 上的绑定程序的单个进程中的工作线程数量通常不会加快此类程序的主要原因。实际上，与单线程程序相比，添加线程有时会对程序的性能产生不利影响。这与所有切换和等待相关的成本有关。

------

总结本章的内容，我们回顾了到目前为止在 python 虚拟机上创建的模型。当 python 可执行文件是包含某些有效源码内容的文件时，首先会初始化解释器和线程状态，然后将源文件编译为代码对象。然后将代码对象传递到解释器循环模块，在该模块中，为了执行代码对象，创建了一个 frame 对象并将其附加到执行的主线程中。因此，我们有一个 python 进程，该进程可能包含一个或多个解释器状态，并且每个解释器状态可能具有一个或多个线程状态，并且每个线程状态都引用了一个 frame ，该 frame 可以引用另一个 frame ，依此类推，形成一个 frame 堆栈。图7.2 展示了此顺序。

![](./images/image-20191130134130.png)

​								Figure 7.2: Interpreter state, thread state and frame relationship

在下一章中，我们将展示我们所描述的所有部分如何实现 python 代码对象执行的。

