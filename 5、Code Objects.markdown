---
layout: post
title:  "Inside The Python Virtual Machine --  5、Code Objects"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第5章介绍部分的中文内容。

<!-- more -->


## 5. Code Objects

在本文的这部分中，我们探索的内容是代码对象。代码对象是 python 虚拟机操作的核心部分。 代码对象封装了 python 虚拟机的字节码；我们可以将字节码称为 python 虚拟机的汇编语言。

顾名思义，代码对象代表着已经编译的并且可执行 python 代码。在讨论 python 源代码的编译之前，我们已经见过了代码对象。正如 python 文档中所述，每当编译 python 代码块时，都会生成代码对象。

Python 程序是由代码块构成的。块是作为一个单元 (unit) 执行的一段 Python 程序文本。块有以下类型：模块，函数体和类的定义。交互键入的每个命令都是一个块。脚本文件 (作为标准输入给解释器或指定作为解释器命令行参数的文件) 是代码块。脚本命令 (在解释器命令行上使用 “_c” 选项指定的命令) 是代码块。传递给内置函数 eval() 和 exec() 的字符串参数是一个代码块。

代码对象包含可运行的字节码指令，这些指令在运行时会更改 python 虚拟机的状态。给定一个函数，我们可以使用函数的 __code__ 属性访问函数主体的代码对象，如以下代码片段所示。

Listing 5.1: Function code objects

```python
def return_author_name():
	return "obi Ike-Nwosu"

>>> return_author_name.__code__
<code object return_author_name at 0x102279270, file "<stdin>", line 1>
```

对于其他代码块，可以通过编译这些代码来获取该代码块的代码对象。 在 python 解释器中 compile 函数为此提供了便利。代码对象带有许多在执行时由解释器循环使用的字段，在下面部分中，我们将介绍其中的一些字段。

### 5.1 Exploring code objects

了解代码对象的一个好办法就是编译一个简单的函数，然后检查由该函数生成的代码对象。我们使用函数 fizzbuzz 来作为实验的对象，如 list 5.2 所示。

Listing 5.2: Function code objects attributes of Fizzbuzz function

```python
co_argcount = 1
co_cellvars = ()
co_code = b'|\x00d\x01\x16\x00d\x02k\x02r\x1e|\x00d\x03\x16\x00d\x02k\x02r\x1ed\\
x04S\x00n,|\x00d\x01\x16\x00d\x02k\x02r0d\x05S\x00n\x1a|\x00d\x03\x16\x00d\x02k\x02r\
Bd\x06S\x00n\x08t\x00|\x00\x83\x01S\x00d\x00S\x00'
co_consts = (None, 3, 0, 5, 'FizzBuzz', 'Fizz', 'Buzz')
co_filename = /Users/c4obi/projects/python_source/cpython/fizzbuzz.py
co_firstlineno = 6
co_flags = 67
co_freevars = ()
co_kwonlyargcount = 0
co_lnotab = b'\x00\x01\x18\x01\x06\x01\x0c\x01\x06\x01\x0c\x01\x06\x02'
co_name = fizzbuzz
co_names = ('str',)
co_nlocals = 1
co_stacksize = 2
co_varnames = ('n',)
```

除了包含乱码的 co_lnotab 和 co_code 字段，其它打印出来的字段的含义都是显而易见的。我们将会解释这些字段及其对 python 虚拟机的重要性。

1. co_argcount：这是代码块参数的数量。只有函数代码块具有这个值。该值在编译过程中被设置为代码块  AST 的参数集合的长度。执行循环 (evaluation loop) 在代码执行 (code evaluation) 过程中利用这些变量进行完整性的检查，例如检查所有变量是否存在以及是否用于存储局部变量。 
2. co_code：这包扩了执行循环 (evaluation loop) 执行的字节码指令序列。这些字节码指令序列中的每一个字节码指令都由一个 opcode 和一个 oparg (opcode 所在的参数) 组成的。例如，co.co_code[0] 返回指令的第一个字节，124 映射到 python LOAD_FAST 操作码上。 
3. co_consts：此字段是常量的列表，例如代码对象中包含的字符串和数字。上面的示例显示了 fizzbuzz 函数这个字段的内容。这个列表中包含的值是代码执行必不可少的，因为它们是 LOAD_CONST opcode 引用的值。字节码指令 (例如 LOAD_CONST) 的操作数参数是此常量列表的索引。例如，思考 FizzBuzz 函数的 co_consts 值，其值为 (None, 3, 0, 5, “FizzBuzz”, “Fizz”, “ Buzz”) ，然后与下面的反汇编代码对象进行对比。

Listing 5.3: Cross section of bytecode instructions for Fizzbuzz function

```python
0 LOAD_BUILD_CLASS
2 LOAD_CONST 0 (<code object Test at 0x101a02810, file "fiz\
zbuzz.py", line 1>)
4 LOAD_CONST 1 ('Test')
6 MAKE_FUNCTION 0
8 LOAD_CONST 1 ('Test')
10 CALL_FUNCTION 2
12 STORE_NAME 0 (Test)
...
66 LOAD_GLOBAL 0 (str)
68 LOAD_FAST 0 (n)
70 CALL_FUNCTION 1
72 RETURN_VALUE
74 LOAD_CONST 0 (None)
76 RETURN_VALUE
```



回想一下，在编译过程中，如果在函数末尾没有 return 语句，则添加 return None，因此我们可以判断出偏移量为 74 的地方的字节码指令是一个值为 None 的 LOAD_CONST 指令。操作码的参数为 0， 我们可以看到，在 LOAD_CONST 指令实际加载的 None 值在常量列表里的索引为 0 。 
4. co_filename：顾名思义，此字段包含文件的名称，该文件包含创建代码对象的源代码。 
5. co_firstlineno：这里给出了源代码对象开始所在的行号。在如调试代码之类的活动中起着非常重要的作用。 
6. co_flags：此字段指示代码对象的类型。例如，当代码对象是协程对象时，该 flag 会被设置为 0x0080 。还有一些其他的 flags ，例如 CO_NESTED 指示一个代码对象是否嵌套在另一个代码块中，CO_VARARGS 指示一个代码块是否具有可变参数等等。这些 flags 影响字节码 (Bytcode) 执行期间执行循环的行为。 
7. co_lnotab：包含一个字节字符串，用于计算字节码偏移量处的指令所对应的源行号。例如，dis 函数在计算指令的行号时会使用此功能。 
8. co_varnames：这是在代码块局部中定义的名称的数量。将此与 co_names 对比。 
9. co_names：这是在代码对象内使用的非局部名称的集合。例如，list 5.4 中的代码段引用了非局部变量 p 。

Listing 5.4: Illustrating local and non-local names

```python
def test_non_local():
    x = p + 1
    return x
```

list 5.5 中显示了对 list 5.4 中函数代码对象的自省的结果。

Listing 5.5: Illustrating local and non-local names

```python
co_argcount = 0
co_cellvars = ()
co_code = b't\x00d\x01\x17\x00}\x00|\x00S\x00'
co_consts = (None, 1)
co_filename = /Users/c4obi/projects/python_source/cpython/fizzbuzz.py
co_firstlineno = 18
co_flags = 67
co_freevars = ()
co_kwonlyargcount = 0
co_lnotab = b'\x00\x01\x08\x01'
co_name = test_non_local
co_names = ('p',)
co_nlocals = 1
co_stacksize = 2
co_varnames = ('x',)
```

从这个例子中可以看出，c_names 和 co_varnames 之间的区别是显而易见的。 co_varnames 引用局部定义的名称，而 co_names 引用非局部定义的名称。请注意，只有在程序执行期间，如果找不到变量 p，才会引发错误。list 5.6 中显示了 list 5.4 中该函数的字节码指令，这里如何生效是显而易见的。

Listing 5.6: Bytecode instructions for test_non_local function

```python
0 LOAD_GLOBAL 0 (0)
3 LOAD_CONST 1 (1)
6 BINARY_POWER
7 STORE_FAST 0 (0)
10 LOAD_FAST 0 (0)
13 RETURN_VALUE
```

注意我们有一个 LOAD_GLOBAL 指令而不是上一个示例中看到的 LOAD_FAST 指令。当我们稍后讨论执行循环 (evaluation loop) 时，我们将会讨论执行循环 (evaluation loop) 所执行的优化，该优化利用了 LOAD_FAST 指令。

10. co_nlocals：这是一个数值，它代表了代码对象使用的局部名称的数量。在 list 5.4 的示例中，唯一使用的局部变量是 x，因此对于该函数的代码对象来说，该值为 1。 
11. co_stacksize：python 虚拟机是基于堆栈的，即用于执行 (evaluation) 和执行结果 (results of evaluation) 的值可从执行堆栈读取或写入执行堆栈。这个 co_stacksize 值是代码块执行期间任意时刻执行栈上存在的最大 item 数量。 
12. co_freevars：co_freevars 字段是在代码块内定义的自由变量的集合。此字段与形成闭包的嵌套函数密切相关。不同于全局变量，自由变量是在一个块内使用的但未在该块内定义的变量。list 5.7 所展示的例子说明了自由变量的概念。

Listing 5.7: A simple nested function

```python
def f(*args):
    x=1
    def g():
    	n = x
```

对于 f 函数的代码对象，co_freevars 字段为空，而 g 函数的代码对象中 co_freevars 的值为 x 。自由变量与单元变量 (cell variables) 密切相关。 
13. co_cellvars：co_cellvars 字段是名称的集合，在执行代码对象期间必须创建单元 (cell) 用来存储对象。以list 5.7 中的代码段为例，函数 f 的代码对象的 co_cellvars 字段仅包含名称 x，而嵌套函数的代码对象的co_cellvars 字段为空；回想一下有关自由变量的讨论，嵌套函数的代码对象的 co_freevars 集合仅包含 x 。这说明了单元变量和自由变量之间的关系：嵌套范围内的自由变量是闭包范围内的单元变量。在代码对象执行期间，将创建特殊的单元对象以将值存储在此单元格变量集合中。之所以如此，是因为该字段中的每个值都被嵌套的代码对象使用，它们的生存期可能会超过闭包代码对象的生存时间，因此此类值必须存储在代码对象执行完成时不会释放的位置。

**The bytecode - co_code in more detail.**

如前所述，代码对象的实际虚拟机指令字节码包含在代码对象的 co_code 字段中。例如，来自 fizzbuzz 函数的字节代码是 list 5.7 中所示的字节字符串。

Listing 5.7: Bytecode string for fizzbuzz function

```python
b'|\x00d\x01\x16\x00d\x02k\x02r\x1e|\x00d\x03\x16\x00d\x02k\x02r\x1ed\x04S\x00n,|\x0\
0d\x01\x16\x00d\x02k\x02r0d\x05S\x00n\x1a|\x00d\x03\x16\x00d\x02k\x02rBd\x06S\x00n\x\
08t\x00|\x00\x83\x01S\x00d\x00S\x00'
```

为了获得人类可读的字节字符串版本，我们使用 dis 模块中的 dis 函数来提取人类可读的打印输出，如list 5.8 所示。

Listing 5.8: Bytecode instruction disassembly for fizzbuzz function

```python
7 		0 LOAD_FAST 			0 (n)
		2 LOAD_CONST 			1 (3)
		4 BINARY_MODULO
		6 LOAD_CONST 			2 (0)
		8 COMPARE_OP 			2 (==)
		10 POP_JUMP_IF_FALSE 	30
		12 LOAD_FAST 			0 (n)
		14 LOAD_CONST 			3 (5)
		16 BINARY_MODULO
		18 LOAD_CONST 			2 (0)
		20 COMPARE_OP 			2 (==)
		22 POP_JUMP_IF_FALSE 	30
		...
14 	>>	66 LOAD_GLOBAL 			0 (str)
		68 LOAD_FAST 			0 (n)
		70 CALL_FUNCTION 		1
		72 RETURN_VALUE
	>> 	74 LOAD_CONST 			0 (None)
		76 RETURN_VALUE
```

输出的第一列显示该指令的行号。多个指令可以映射到同一行号。使用来自代码对象的 co_lnotab 字段的信息来计算此值。第二列是给定指令与字节码开头的偏移量。假设字节码字符串包含在数组中，则此值是可以在该数组中找到给定指令的索引。第三列是实际的人类可读指令操作码；完整的操作码可以在 Include/opcode.h 模块中找到。第四列是指令的参数。

第一条 LOAD_FAST 指令的参数为 0 。此值是 co_varnames 数组的索引。最后一列是参数的值由 dis 函数提供，以方便使用。一些参数不采用显式参数。请注意，BINARY_MODULO 和 RETURN_VALUE 指令没有任何显式参数。回想一下，python 虚拟机是基于堆栈的，因此这些指令可以从堆栈顶部读取值。

字节码指令的大小为两个字节：一个字节用于操作码，第二个字节用于操作码的参数。如果操作码不带参数，则第二个参数字节为 0 。 在写这本书期间，Python 虚拟机在机器上使用一些字节序 (endian) 字节编码，因此 16 位代码的结构如图 5.0 所示，其中操作码占据了较高的 8 位，操作码的参数占据了 8 位。

![](./images/image-20191126223951.png)

​							Figure 5.0: Bytecode instruction format showing opcode and oparg

有时候，操作码的参数可能无法放入默认的单个字节中。对于这些类型的参数，python 虚拟机使用 EXTENDED_ARG 操作码。 python 虚拟机的做法是当接受一个太大而无法容纳一个字节的参数时，将其拆分为两个字节 (我们假设此处可以容纳两个字节) ：最高有效字节是 EXTENDED_ARG 操作码的参数，而最低有效字节是其实际操作码的参数。 EXTENDED_ARG 操作码将在操作码序列中的实际操作码之前出现，然后可以通过向右移动or’ing 参数的其他部分参数一起来重构操作码和参数。例如，如果希望将值 321 作为参数传递给 LOAD_CONST 操作码，则该值不能放入单个字节中，因此使用 EXTENDED_ARG 操作码。此值的二进制表示形式为 0b101000001 ，因此实际的操作码 (LOAD_CONST) 将第一个字节  (1000001) 作为参数 (十进制65)，而 EXTENDED_ARG 操作码将下一个字节 (1) 作为参数，因此我们具有(144, 1), (100, 65) 作为输出的指令序列。

dis 模块的文档包含有关虚拟机当前实现的所有操作码的完整列表和说明。

### 5.2 Code Objects within other code objects

另一个值得关注的代码块代码对象是正在编译的模块。假设我们正在编译一个带有 fizzbuzz 函数作为内容的模块，那么输出将是什么样？为了找出答案，我们使用 python 中的 compile 函数来编译模块，其内容如 list 5.9 所示。

Listing 5.9: Nested function to illustrated nested code objects

```python
def f():
    print(c)
    a = 1
    b = 3
    def g():
        print(a+b)
        c=2
        def h():
        	print(a+b+c)
```

编译模块代码块后，我们得到如 list 5.10 所示的输出。

Listing 5.10: Bytecode instruction disassembly for listing 5.10

```python
		0 LOAD_CONST 			0 (<code object f at 0x102a028a0, file "fizzbuzz.py",\
line 1>)
		2 LOAD_CONST 			1 ('f')
		4 MAKE_FUNCTION 		0
		6 STORE_NAME 			0 (f)
		8 LOAD_CONST 			2 (None)
		10 RETURN_VALUE
```

字节偏移量为 0 的指令加载了一个代码对象，该对象存储名称为 f ：我们函数定义使用了 MAKE_FUNCTION 指令。list 5.11 显示了此代码对象的内容。

Listing 5.11: Bytecode instruction disassembly for nested function from listing 5.9

```python
co_argcount = 0
co_cellvars = ()
co_code = b'd\x00d\x01\x84\x00Z\x00d\x02S\x00'
co_consts = (<code object f at 0x1022029c0, file "fizzbuzz.py", line 1>, 'f', No\
ne)
co_filename = fizzbuzz.py
co_firstlineno = 1
co_flags = 64
co_freevars = ()
co_kwonlyargcount = 0
co_lnotab = b''
co_name = <module>
co_names = ('f',)
co_nlocals = 0
co_stacksize = 2
co_varnames = ()
```

就像在模块中预期的那样，与代码对象参数相关的字段全为 0:  (co_argcount, co_kwonlyargcount) 。如 list 5.10 所示，co_code 字段包含了字节码指令。co_consts 字段是一个有趣的字段。字段中的常量是代码对象，名称为： f 和 None。代码对象是函数的对象，值 “f” 是函数的名称，“None” 是函数的返回值。回想一下，python 编译器向没有返回值的代码对象添加了 “return None” 语句。

需要注意的是，在模块编译期间实际上并未创建函数对象。我们所拥有的只是代码对象：函数实际上是在代码对象执行期间创建的，如 list 5.10 所示。检查代码对象的属性表明它也是由其他代码对象组成的，如 list 5.12 所示。

Listing 5.12: Bytecode instruction disassembly for nested function from listing 5.10

```python
co_argcount = 0
co_cellvars = ('a', 'b')
co_code = b't\x00t\x01\x83\x01\x01\x00d\x01\x89\x00d\x02\x89\x01\x87\x00\x87\x01\
f\x02d\x03d\x04\x84\x08}\x00d\x00S\x00'
co_consts = (None, 1, 3, <code object g at 0x101a028a0, file "fizzbuzz.py", line\
5>, 'f.<locals>.g')
co_filename = fizzbuzz.py
co_firstlineno = 1
co_flags = 3
co_freevars = ()
co_kwonlyargcount = 0
co_lnotab = b'\x00\x01\x08\x01\x04\x01\x04\x01'
co_name = f
co_names = ('print', 'c')
co_nlocals = 1
co_stacksize = 3
co_varnames = ('g',)
```

前面的解释在这里也适用，仅在执行代码对象期间创建函数对象。

### 5.3 Code Objects in the VM

VM 中代码对象的实现和对象属性在 python 中的实现非常相似。与大多数内置类型一样，有一些代码类型为代码对象实例定义了代码对象类型和 PyCodeObject 结构。代码类型与前面各节中讨论的其他类型对象相似，因此不再赘述。代码对象的实例如 list 5.13 中的结构所示。

Listing 5.13: Code object implementation in C

```c
typedef struct {
    PyObject_HEAD
    int co_argcount; 			/* #arguments, except *args */
    int co_kwonlyargcount; 		/* #keyword only arguments */
    int co_nlocals; 			/* #local variables */
    int co_stacksize; 			/* #entries needed for evaluation stack */
    int co_flags; 				/* CO_..., see below */
    int co_firstlineno; 		/* first source line number */
    PyObject *co_code; 			/* instruction opcodes */
    PyObject *co_consts; 		/* list (constants used) */
    PyObject *co_names; 		/* list of strings (names used) */
    PyObject *co_varnames; 		/* tuple of strings (local variable names) */
    PyObject *co_freevars; 		/* tuple of strings (free variable names) */
    PyObject *co_cellvars; 		/* tuple of strings (cell variable names) */
    
    unsigned char *co_cell2arg; /* Maps cell vars which are arguments. */
    PyObject *co_filename; 		/* unicode (where it was loaded from) */
    PyObject *co_name; 			/* unicode (name, for reference) */
    PyObject *co_lnotab; 		/* string (encoding addr<->lineno mapping) See
    								Objects/lnotab_notes.txt for details. */
    void *co_zombieframe; 		/* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist; 	/* to support weakrefs to code objects */
    /* Scratch space for extra data relating to the code object.__icc_nan
    	Type is a void* to keep the format private in codeobject.c to force
    	people to go through the proper APIs. */
    void *co_extra;
} PyCodeObject;
```

除了 co_stacksize，co_flags，co_cell2arg，co_zombieframe，co_weakreflist 和 co_extra 这些字段外，其与字段几乎都与 python 代码对象中的字段相同。因此，co_weakreflist 和 co_extra 并不是什么特殊的字段。这里的其余字段几乎具有与代码对象中相同的目的。 co_zombieframe 是为优化目的而存在的字段。这保留了对以前用作执行代码对象上下文的 frame 对象的引用。当这样的代码对象被重新执行时，它被用作执行 frame ，以减少另一个 frame 对象分配内存的开销。

