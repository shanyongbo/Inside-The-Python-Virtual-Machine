---
layout: post
title:  "Inside The Python Virtual Machine --  10、The Block Stack"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第10章介绍部分的中文内容。

<!-- more -->


## 10. The Block Stack

没有获得应有关注的数据结构之一就是块堆栈 (block stack) ，它是 frame 对象内的另一个堆栈。Python VM 的大多数讨论都只是顺带提及了块堆栈，然后就会将重点放在执行堆栈 (evaluation stack) 上。但是，块堆栈非常重要：可能还有其他方法可以实现异常处理，但是正如我们在本章学习的过程中所看到的那样，使用块堆栈会使实现异常处理变得异常简单。块堆栈和异常处理交织在一起，以至于如果不思考异常处理，就不会完全理解块堆栈重要性。块堆栈也用于循环，但是很难弄清带有循环的块堆栈，直到人们研究了诸如 break 之类的循环结构如何与异常处理程序交互时，才让我们直接了解细节。块堆栈使这种交互的实现变得简单。

块堆栈是 frame 对象内的堆栈数据结构字段。就像 frame 的执行堆栈 (evaluation loop) 一样，在执行 frame 的代码期间，会将值压入块堆栈并从中弹出。但是，块堆栈仅用于处理循环和异常。解释块堆栈的最好方法是举一个例子，因此我们用一个简单的 try ... finally 构造一个循环，如 list10.0 所示。

Listing 10.0: Simple python function with exception handling

```python
def test():
    for i in range(4):
        try:
            break
        finally:
            print("Exiting loop")
```

当 list10.0 中的函数被分解时，其结果如 lsit 10.1 所示。

Listing 10.1: Disassembly of function in listing 10.0

```python
2		0 SETUP_LOOP 					34 (to 36)
		2 LOAD_GLOBAL 					0 (range)
		4 LOAD_CONST 					1 (4)
		6 CALL_FUNCTION 				1
		8 GET_ITER
	>> 10 FOR_ITER 						22 (to 34)
	   12 STORE_FAST 					0 (i)
3	   14 SETUP_FINALLY 				6 (to 22)

4	   16 BREAK_LOOP
	   18 POP_BLOCK
	   20 LOAD_CONST 					0 (None)

6	>> 22 LOAD_GLOBAL 					1 (print)
	   24 LOAD_CONST 					2 ('Exiting loop')
	   26 CALL_FUNCTION 				1
	   28 POP_TOP
	   30 END_FINALLY
	   32 JUMP_ABSOLUTE 				10
	>> 34 POP_BLOCK
	>> 36 LOAD_CONST 					0 (None)
	   38 RETURN_VALUE
```

对于一个简单的函数，list 10.1 有很多操作码，但这是由于 for 循环和 try .. finally 构造的结合导致的。这里重点的操作码是 SETUP_LOOP 和 SETUP_FINALLY 操作码，因此我们看一下它们的实现，以了解其工作的要旨 (所有 SETUP_* 操作码都映射到相同的实现) 。 

SETUP_LOOP 操作码的实现是一个简单的函数调用：PyFrame_BlockSetup( f, opcode, INSTR_OFFSET() + oparg, STACK_LEVEL()); 参数是很容易解释的：f 是 frame，opcode 是当前正在执行的操作码，INSTR_OFFSET() + oparg 是该块之后的下一条指令的指令增量 (对于上面的代码，SETUP_LOOP 的增量为 50)，并且 STACK_LEVEL 表示该 frame 的值堆栈上有多少个 items 。函数调用将创建一个新块并将其压入块堆栈。该块中包含的信息足以使虚拟机在该块中发生某些情况时继续执行。list 10.2 展示了此函数的实现。

Listing 10.2: Block setup code

```c
void PyFrame_BlockSetup(PyFrameObject *f, int type, int handler, int level){
    PyTryBlock *b;
    if (f->f_iblock >= CO_MAXBLOCKS)
    	Py_FatalError("XXX block stack overflow");
    b = &f->f_blockstack[f->f_iblock++];
    b->b_type = type;
    b->b_level = level;
    b->b_handler = handler;
}
```

list 10.2 中的处理程序是指向 SETUP_ * 块之后应执行的下一条指令的指针。最好从上方用图形表示执行过程来说明，而图 10.0 则用一部分字节码说明该示例。

![](./images/image-20191130143901.png)

![](./images/image-20191130143902.png)

![](./images/image-20191130143903.png)

图 10.0 显示了块堆栈如何随每条指令的执行而变化。

在图 10.0 的第一个图中，SETUP_LOOP 操作码被执行，并将单个 SETUP_LOOP 块放置在块堆栈上。该块的处理程序是偏移量为 36 处的指令，因此当在正常执行下弹出堆栈时，解释器将跳转到该偏移量处并从此处继续执行。遇到 SETUP_FINALLY 操作码时，另一个块被压入块堆栈。我们可以看到，由于堆栈是后进先出数据结构，因此 finally 块将是最先出来，回顾一下，无论 break 语句如何，都必须执行 finally 。

使用块堆栈的真正地方是当在循环内的异常处理程序中遇到 break 语句时。当执行 BREAK_LOOP 操作码时，将 why 变量设置为 WHY_BREAK 并跳转到 fast_block_end 代码标签，如图 10.0 的第二张图所示，其中处理了块堆栈展开。展开只是在堆栈上弹出块并执行其处理程序的一个别名。因此，在这种情况下，SETUP_FINALLY 块从堆栈中弹出，并且解释器以字节码偏移量为 22 跳转到其处理程序。正常执行从该偏移量继续执行，直到遇到 END_FINALLY 语句为止。由于代码为何为 WHY_BREAK，因此将再次执行一次跳转到 fast_block_end 代码标签，在该标签处发生更多的堆栈展开操作：循环块保留在堆栈上。这次 (从图 10.0 中未显示)，从堆栈弹出的块在字节偏移量为 36 处有一个处理程序，因此在该字节码偏移量处程序继续执行，从而完成循环退出并继续正常执行。

块堆栈的使用大大简化了虚拟机实现的实现。如果循环不是使用块堆栈实现的，则 BREAK_LOOP 之类的操作码将需要跳转目标。如果随后使用该 break 语句抛出一个 try..finally 结构，就会需要一个复杂的实现，在该实现中，我们必须跟踪 finally 块内的可选跳转目标，依此类推。

### 10.1 A Short Note on Exception Handling

有了对块堆栈的基本了解，就不难理解如何实现异常和异常处理。list 10.3 中的代码段试图向字符串添加数字。

Listing 10.3: Simple python function with exception handling

```python
def test1():
    try:
    	2 + 's'
    except Exception:
    	print("Caught exception")
```

list 10.4 中显示了 list 10.3 中的简单函数生成的操作码。

Listing 10.4: Disassembly of function in listing 10.3

```python
2 	0 SETUP_EXCEPT 						12 (to 14)
3 	2 LOAD_CONST 						1 (2)
	4 LOAD_CONST 						2 ('s')
	6 BINARY_ADD
	8 POP_TOP
	10 POP_BLOCK
	12 JUMP_FORWARD 					28 (to 42)
4 	>> 14 DUP_TOP
	16 LOAD_GLOBAL 						0 (Exception)
	18 COMPARE_OP 						10 (exception match)
	20 POP_JUMP_IF_FALSE 				40
	22 POP_TOP
	24 POP_TOP
	26 POP_TOP
5 	28 LOAD_GLOBAL 						1 (print)
	30 LOAD_CONST 						3 ('Caught exception')
	32 CALL_FUNCTION 					1
	34 POP_TOP
	36 POP_EXCEPT
	38 JUMP_FORWARD 					2 (to 42)
	>> 40 END_FINALLY
	>> 42 LOAD_CONST 					0 (None)
	44 RETURN_VALUE
```

鉴于前面的解释，我们应该对如果发生异常会如何执行此代码块有一个概念性的想法。总之，我们期望 Objects/abstract.c 模块中的 PyNumber_Add 函数为 BINARY_ADD 操作码返回 NULL 。这里需要说明的是，事实上函数除了返回 NULL 值之外，函数还在当前正在执行的线程的线程状态数据结构上设置异常值。回想一下，线程状态具有用于保存执行线程中当前异常的 curxc_type，curxc_value 和 curxc_traceback 字段；这些字段在展开异常搜索处理程序中的块堆栈时非常有用。你可以遵循从 Objects/abstract.c 模块中的 binop_type_error 函数一直到在当前执行线程上已设置值的同一模块中的 PyErr_Restore 函数的函数调用链。

在当前执行的线程上设置了异常值并且从函数调用返回了 NULL 值之后，解释器循环执行跳转到 error 标签，在该标签上不知道发生了什么。对于上面的示例，我们在块堆栈上只有一个块，即 SETUP_EXCEPT 块，其处理程序的字节码偏移量为 14 。一旦跳转到 error 处理程序标签，就可以开始展开堆栈。异常值的 traceback，异常值和异常类型会被推入值堆栈的顶部，SETUP_EXCEPT 处理程序从块堆栈中弹出，然后跳转到该处理程序，在这种情况下，程序从字节偏移为 14 的地方继续执行。现在观察 list 10.4 中从字节码偏移量为 16 到字节码偏移量为 20 的字节码：将 Exception 类加载到堆栈上，然后将其与引发并出现在堆栈上的异常进行比较。如果异常匹配，则正常执行可以继续，从值堆栈中弹出 exception 和 traceback ，并执行任意错误处理程序代码。如果没有异常匹配，则执行 END_FINALLY 指令，并且由于堆栈上仍存在异常，因此异常循环会中断。

在没有异常处理机制的情况下，用于测试功能的操作码更为直接，如清单10.5所示。

Listing 10.5: Disassembly of function in listing 10.3 when there is no exception handling

```python
2 	0 LOAD_CONST 						1 (2)
	2 LOAD_CONST 						2 ('s')
	4 BINARY_ADD
	6 POP_TOP
	8 LOAD_CONST 						0 (None)
	10 RETURN_VALUE
```

操作码不会在块堆栈上放置任何内容，因此，当发生异常并且跳转到 error 处理标签时，不会从堆栈上展开块，从而导致循环退出并存储错误。

这并未涵盖异常处理机制的全部的工作原理，但涵盖了 python 虚拟机中块堆栈与错误处理之间的交互作用的基本原理。还有一些其他的细节，例如在处理异常时引发异常的情况，嵌套异常处理程序和嵌套异常的情况等。
