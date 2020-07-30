---
layout: post
title:  "Inside The Python Virtual Machine --  4、Python Objects"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第4章介绍部分的中文内容。

<!-- more -->


## 4. Python Objects

在本章中，我们将研究 python 对象以及它们在 CPython 虚拟机中的实现。理解 python 对象如何如何进行组织的对于理解 python 虚拟机的内部结构十分重要。我们可以在 Include/ 和 Objects/ 目录中找到此处讨论的大多数来源。毫不奇怪，用 python 实现对象系统非常复杂，我们尽力避免陷入 C 实现的繁琐细节中。首先，我们先来看看 PyObject 结构 —— python 对象系统的主要部分。

### 4.1 PyObject

粗略查看 CPython 的源码表明 PyObject 结构的随处可见。实际上，正如我们稍后在本节中看到的那样，当解释器循环正在处理执行堆栈上的值时，所有这些值都被视为 PyObjects。如果需要更好的术语，我们将其称为所有 python 对象的超类。实际上，没有任何值被声明为 PyObject，但是可以将指向任何对象的指针强制转换为PyObject。总而言之，任何对象都可以被视为 PyObject 结构，因为所有对象的初始段 (initial segment) 实际上都是 PyObject 结构。

**A word on C structs**

当我们说没有任何值被声明为 PyObject，但是可以将指向任何对象的指针强制转换为 PyObject 时，我们指的是在 C 语言及其如何解释内存位置数据的实现细节。用于表示 python 对象的 C 结构体只是一组字节，我们可以选择以任何方式解释它们。例如，一个 test 结构体，由 5 个短值组成，每个值 2 个字节，总和最多 10 个字节。在 C 语言中，给定 10 个字节的引用，我们可以将这 10 个字节解释为由 5 个短值组成的 test 结构体，而不管这 10 个字节是否真的是定义为 test 的结构体，但是，当你尝试访问该结构体的字段时，输出也许是乱码。这意味着在给定 n 个表示 python 对象数据的 n 个字节 (其中 n 大于 PyObject 的大小) 的情况下，我们可以将前 n 个字节解释为 PyObject。

PyObject 结构如 list 4.0 所示，它由多个字段组成，这些字段都存在才能将它视为对象。

Listing 4.0: PyObject definition

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```

_PyObject_HEAD_EXTRA 现在是 C 中的一个宏，它定义了指向先前分配的对象和下一个对象的字段，这些字段形成所有活动对象的隐式双链表。 ob_refcnt 字段用于内存管理，*ob_type 是指向类型对象的指针，该对象指示对象的类型。正是这种类型决定了数据代表着什么，包含的数据类型以及可以对该对象执行的操作类型。以 list 4.1 中的代码段为例，名称 name，指向一个字符串对象，并且对象的类型为 “str”。

Listing 4.1: Variable declaration in python

```python
>>> name = 'obi'
>>> type(name)
<class 'str'>
```

这里有一个问题，由于类型字段指向类型对象，那么这个类型对象的 *ob_type 字段指向什么？类型对象的 ob_type 实际上指向自身，因此称一个类型对象的类型是类型 (the type of a type is type) 。

**A word on reference counting**

CPython 使用引用计数进行内存管理。这是一种简单的方法，其中只要创建对对象的新引用 (如 list 4.1 中将名称绑定到对象的情况)，对象的引用计数就会增加。反之亦然，每当对一个对象的引用消失 (例如，使用名称上的 del 方法删除该引用) 时，引用计数就会减少。当对象的引用计数变为零时，VM 可以将其释放。在VM 的世界中，Py_INCREF 和 Py_DECREF 用于增加和减少对象的引用计数，它们在我们讨论的许多代码片段中都存在。

VM 中的类型是使用 Objects/Object.h 模块中定义的 _typeobject 数据结构实现的。这是一个 C 结构，其中包含用于大多数功能或每种类型填充的功能集合的字段。接下来我们看一下这个数据结构。

### 4.2 Under the cover of Types

Include/Object.h 中定义的 _typeobject 结构体充当所有 python 类型的基本结构。此数据结构中定义了大量的字段，这些字段大多是指向为 C 函数的指针，这些函数实现给定类型的某些功能。为方便起见，list 4.2 中复制了 _typeobject 结构定义。

Listing 4.2: PyTypeObject definition

```c
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_asyn;

    reprfunc tp_repr;
    
    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;
   
    PyBufferProcs *tp_as_buffer;
    unsigned long tp_flags;
    const char *tp_doc; /* Documentation string */

    traverseproc tp_traverse;

    inquiry tp_clear;
    richcmpfunc tp_richcompare;
    Py_ssize_t tp_weaklistoffset;

    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free;
    inquiry tp_is_gc;
    PyObject *tp_bases;
    PyObject *tp_mro;
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    unsigned int tp_version_tag;
    destructor tp_finalize;
} PyTypeObject;
```

PyObject_VAR_HEAD 字段是上一节中讨论的 PyObject 字段的扩展；此扩展为具有长度概念的对象添加了一个 ob_size 字段。 [python C API 文档]( https://docs.python.org/3.6/c-api/typeobj.html ) 中提供了这个类型对象结构中每个字段的全面说明。需要注意的一点是，结构体中的每个字段都实现了部分类型的行为。部分这些字段可以被我们称为对象接口或协议的一部分，因为它们映射到可以在 python 对象上调用的函数，但是其实际实现方式取决于类型。例如，tp_hash 字段是给定类型的哈希函数的引用，但是如果类型的实例不可哈希，则该字段可以不带值。在该类型的实例上调用 hash 方法时，将调用 tp_hash 字段中的任意函数。类型对象还具有 tp_methods 字段，该类型的引用方法是唯一的。 tp_new 插槽 (slot) 是对创建该类型的新实例的函数的引用，等等。其中某些字段 (例如 tp_init) 是可选的，并不是每种类型都需要运行初始化函数，尤其是当该类型是不可变的 (例如元组) ，但是有些其他字段 (例如tp_new) 是强制性的。

这些字段中还有其他 python 协议的字段，内容如下：

1. 数字协议（Number protocol）：实现此协议的类型将具有 PyNumberMethods *tp_as_number 字段的实现。该字段是对实现类似于数字运算的一组函数的引用，这意味着该类型将支持在 tp_as_number 集合中实现的算术运算。例如，非数值类型在此字段中有一个条目，因为它支持算术运算，例如 -，<= 等。
2. 序列协议（Sequence protocol）：实现此协议的类型将在 PySequenceMethods *tp_as_sequence 字段中具有一个值。这意味着该类型将支持某些或所有序列操作，例如 len，in 等。
3. 映射协议（Mapping protocol）：实现此协议的类型将在 PyMappingMethods *tp_as_mapping 中具有一个值。这样可以使用字典下标语法来设置和访问键-值映射，从而将此类实例视为 python 字典。 
4. 迭代器协议（Iterator protocol）：实现此协议的类型将在 getiterfunc tp_iter 以及 iternextfunc tp_iternext 字段中具有一个值，从而使该类型的实例能够像 python 迭代器一样使用。 
5. 缓冲区协议（Buffer protocol）：实现此协议的类型将在 PyBufferProcs * tp_as_buffer 字段中具有一个值。这些功能将允许访问该类型的实例作为输入/输出缓冲区。

在阅读本章的过程中，我们将更详细地研究构成类型对象的各个字段，但现在，我们将探讨许多不同的类型对象，作为研究“有关如何在实际类型对象中填充这些字段”的具体案例。

### 4.3 Type Object Case Studies

**The tuple type**

我们详细查看元组类型，用来了解如何填充类型对象的字段。我们之所以选择它，是因为考虑到实现的规模较小，它相对容易使用——大约一千多行 C 语言（包括文档字符串）。元组类型的实现如 list 4.3 所示。

Listing 4.3: Tuple type definition

```c
PyTypeObject PyTuple_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "tuple",
    sizeof(PyTupleObject) - sizeof(PyObject *),
    sizeof(PyObject *),
    (destructor)tupledealloc, 					/* tp_dealloc */
    0, 											/* tp_print */
    0, 											/* tp_getattr */
	0, 											/* tp_setattr */
	0, 											/* tp_reserved */
	(reprfunc)tuplerepr, 						/* tp_repr */
	0, 											/* tp_as_number */
	&tuple_as_sequence, 						/* tp_as_sequence */
	&tuple_as_mapping, 							/* tp_as_mapping */
	(hashfunc)tuplehash, 						/* tp_hash */
	0, 											/* tp_call */
	0, 											/* tp_str */
	PyObject_GenericGetAttr, 					/* tp_getattro */
	0, 											/* tp_setattro */
	0, 											/* tp_as_buffer */
	Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
	Py_TPFLAGS_BASETYPE | Py_TPFLAGS_TUPLE_SUBCLASS, /* tp_flags */
	tuple_doc, 									/* tp_doc */
	(traverseproc)tupletraverse, 				/* tp_traverse */
	0, 											/* tp_clear */
	tuplerichcompare, 							/* tp_richcompare */
	0, 											/* tp_weaklistoffset */
	tuple_iter, 								/* tp_iter */
	0, 											/* tp_iternext */
	tuple_methods, 								/* tp_methods */
	0, 											/* tp_members */
	0, 											/* tp_getset */
	0, 											/* tp_base */
	0, 											/* tp_dict */
	0, 											/* tp_descr_get */
	0, 											/* tp_descr_set */
	0, 											/* tp_dictoffset */
	0, 											/* tp_init */
	0, 											/* tp_alloc */
	tuple_new, 									/* tp_new */
	PyObject_GC_Del, 							/* tp_free */
};
```

我们看看在这种类型中填充的字段。 

1.  PyObject_VAR_HEAD 使用类型对象 PyType_Type 作为类型进行初始化。回想一下，类型对象的类型是 Type。查看 PyType_Type 类型对象可发现 PyType_Type 的类型是其自身。 
2.  tp_name 初始化为元组类型的名称。 
3.  tp_basicsize 和 tp_itemsize 是元组对象的大小和包含在元组对象中的元素 (items)，并对它相应地进行填充。 
4.  tupledealloc 是一种内存管理函数，用于在销毁元组对象时对内存进行重新分配。

5. tuplerepr 是使用元组实例作为参数调用 repr 函数时调用的函数。 

6.  tuple_as_sequence 是元组实现的一组序列方法，如元组支持 in, len 等序列方法。 
7. tuple_as_mapping 是元组支持的一组映射方法，在这种情况下，键是整数索引。 
8.  tuplehash 是在需要元组对象的哈希值时调用的函数，当元组用作字典的键或在集合中使用时起作用。 
9.  PyObject_GenericGetAttr 是引用元组对象的属性时调用的通用函数。我们将在后续部分中介绍属性引用。 
10. tuple_doc 是元组对象的文档字符串。 
11.  tupletraverse 是用于遍历元组对象的遍历函数。GC 使用这个功能来帮助检测循环引用。
12.  tuple_iter 是在 tuple 对象上调用 iter 函数时调用的方法。在这种情况下，将返回完全不同的tuple_iterator 类型，因此 tp_iternext 方法没有实现。 
13. tuple_methods 是元组类型的实际方法。 
14. tuple_new 用于创建新的 tuple 类型实例的函数。 
15. PyObject_GC_Del 是另一个引用内存管理功能的字段。

其余具有 0 值的字段保留为空，因为元组的功能不需要它们。以 tp_init 字段为例，元组是不可变的类型，它一旦创建就不能更改，因此除了 tp_new 引用函数中发生的事情除外，不需要任何初始化，因此该字段保留为空。

**The** type **type**

我们要关注的另一种类型是 type 类型。它是所有内置类型和用户定义的普通类型的元类 (用户可以定义新的元类)，注意在 PyVarObject_HEAD_INIT 中初始化元组对象时如何使用此类型的。在讨论类型时，重要的是区分以 type 为类型的对象和以用户定义类型为类型的对象。这在处理对象中的属性引用时非常重要。

此类型定义了使用类型时使用的方法，并且这些字段与之前章节的方法相似。在创建新类型时 (如我们在后续各节中所见) ，将使用此类型。

**The object type**

另一个重要的类型是 object 类型，它与 type 类型非常相似。object 类型是所有用户定义类型的根类型，并提供一些默认值，用于填充用户定义类型的类型字段。由于和以 type 作为其类型的类型相比，用户定义的类型的行为方式是不同的。正如我们将在后续部分中看到的那样，object 类型和 type 类型所提供的关于属性解析算法之类的功能之间有很大不同。

### 4.4 Minting type instances

假定大家对类型的基本类型 (type) 有深刻的理解，那么我们就可以学习类的最基本功能之一，即 使用类创建实例。为了充分理解创建新类型实例的过程，我们要记住，正如我们在区分内置类型和用户定义类型一样，它们两者的内部结构也会有所不同。 tp_new 字段在 python 新类型实例中普遍存在。下面 tp_new 插槽 (slot) 的文档对应该填充的插槽 (slot) 功能进行了很好的描述。

一个可选的指向实例创建函数的指针。如果对于特定类型，该函数为 NULL，那么该类型无法被调用，从而创建出新的实例；还有其他创建实例的方法，例如工厂函数。函数签名是：

```c
PyObject * tp_new(PyTypeObject *subtype，PyObject * args，PyObject * kwds)
```

subtype 参数是要创建的对象的类型。 args 和 kwds 参数表示调用该类型所需的位置参数和关键字参数。subtype 不必等于调用 tp_new 函数的类型。它可能是该类型的子类型 (但不是无关类型) 。 tp_new 函数需要调用subtype _> tp_alloc(subtype，nitems) 为对象分配空间，然后仅在绝对必要的情况下进行更多的初始化。可以安全地忽略或重复进行的初始化应该放在 tp_init 处理程序中。一个好的经验法则是，对于不可变类型，所有初始化都应在 tp_new 中进行，而对于可变类型，大多数初始化应推迟到 tp_init 中进行。

此字段由子类型继承，但不是由 tp_base 为 NULL 或 &PyBaseObject_Type 的静态类型继承。

我们将使用上一节中的元组类型作为内置类型的示例。元组类型的 tp_new 字段引用 list 4.4 中所示的tuple_new 方法，该方法处理新的元组对象的创建。为了创建一个新的元组对象，需要调用这个函数并且取消相关的引用。

Listing 4.4: tuple_new function for creating new tuple instances

```c
static PyObject * tuple_new(PyTypeObject *type, PyObject *args,
                            PyObject *kwds){
    PyObject *arg = NULL;
    static char *kwlist[] = {"sequence", 0};

    if (type != &PyTuple_Type)
        return tuple_subtype_new(type, args, kwds);
    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|O:tuple", kwlist, &arg))
        return NULL;

    if (arg == NULL)
        return PyTuple_New(0);
    else
        return PySequence_Tuple(arg);
}
```

忽略 list 4.4 中创建元组的第一个和第二个条件，我们遵循第三个条件，顺着代码 if (arg == NULL) return PyTuple_New(0) 从而了解其工作原理。忽略 PyTuple_New 函数中的优化，函数中创建新元组对象的部分是 op = PyObject_GC_NewVar( PyTupleObjectl, ＆PyTuple_Type, size ) 调用，该调用基本上为堆上的 PyTuple_Object 结构体的实例分配内存。这是内建类型和用户定义类型的内部表示之间的一个明显区别，内建类型的实例 (例如元组) 实际上是 C 的结构体。这可能是为了提高效率。那么支持元组对象的 C 结构体看起来像什么？可以在 Include/ tupleobject.h 中找到它作为 PyTupleObject 类型定义 ， 如 list 4.5 中所示。

Listing 4.5: PyTuple_Object definition

```c
typedef struct {
    PyObject_VAR_HEAD
    PyObject *ob_item[1];
    
    /* ob_item contains space for 'ob_size' elements.
    * Items must normally not be NULL, except during construction when
    * the tuple is not yet visible outside the function that builds it.
    */
} PyTupleObject;
```

PyTupleObject 是定义为具有 PyObject_VAR_HEAD 和 PyObject 类型的指针数组的结构体：ob_items 。与使用 python 数据结构表示实例相比，这种实现非常高效。

回想一下，对象是方法和数据的集合。在这种情况下，PyTupleObject 提供了空间来保存每个元组对象包含的实际数据，因此我们可以在堆上分配多个 PyTupleObject 实例，但是这些实例都是单个 PyTuple_Type 类型的引用，该类型提供可以对这些数据进行操作的方法。

现在考虑一个用户定义的类，如 list 4.6 所示。

Listing 4.6: User defined class

```python
class Test:
    pass
```

Test 类型是 Type 类型的实例对象。要创建 Test 类型的实例，需要调用 Test 类型：Test() 。和往常一样，我们可以顺其自然的理清类型对象被调用时发生的事情。 Type 类型具有一个函数引用：type_call，它填充在 tp_call  字段内，并且每当在 Type 实例上使用调用符号时，都会取消引用。list 4.7 中展示了 type_call 函数实现的代码片段。

Listing 4.7: A snippet of type_call function definition

```c
...
obj = type->tp_new(type, args, kwds);
obj = _Py_CheckFunctionResult((PyObject*)type, obj, NULL);
if (obj == NULL)
    return NULL;

/* Ugly exception: when the call was type(something),
don't call tp_init on the result. */
if (type == &PyType_Type &&
    PyTuple_Check(args) && PyTuple_GET_SIZE(args) == 1 &&
    (kwds == NULL ||
     (PyDict_Check(kwds) && PyDict_Size(kwds) == 0)))
    return obj;

/* If the returned object is not an instance of type,
it won't be initialized. */
if (!PyType_IsSubtype(Py_TYPE(obj), type))
    return obj;

type = Py_TYPE(obj);
if (type->tp_init != NULL) {
    int res = type->tp_init(obj, args, kwds);
    if (res < 0) {
        assert(PyErr_Occurred());
        Py_DECREF(obj);

        obj = NULL;
    }
    else {
        assert(!PyErr_Occurred());
    }
}
return obj;
```

list 4.7 显示了调用 Type 对象的实例时，就会取消对 tp_new 字段的引用，并调用了所引用的任意函数从而得到一个新的实例。如果 tp_init 存在，就会在新实例上调用它，从而对新实例进行初始化。这个过程为内置类型提供了解释，因为它们已经定义了自己的 tp_new 和 tp_init 函数，但是用户定义的类型呢？大多数情况下，用户不会为新类型定义 __new__ 函数 (在定义时，它会在类创建期间进入 tp_new 字段) 。答案还取决于实现 Type 中 tp_new 字段的 type_new 函数。在创建用户定义的类型时，如自定义类 Test, type_new 函数会去检查是否存在基本类型 (超类/父类) ，如果不存在，则将 PyBaseObject_Type 类型添加为默认基本类型，如 list 4.8 所示。

Listing 4.8: Snippet showing how the PyBaseObject_Type is added to list of bases

```c
...
if (nbases == 0) {
    bases = PyTuple_Pack(1, &PyBaseObject_Type);
    if (bases == NULL)
        goto error;
    nbases = 1;
}
...
```

这个默认的基本类型也是在 Objects/typeobject.c 模块中定义的，其中包含各个字段的一些默认值。这些默认值中包括 tp_new 和 tp_init 字段的值。这些值是在用户定义类时被解释器调用。在用户定义的类实现自己的方法 (例如 __init __ , __ new__ 等) 的情况下，会调用这些值，而不是 PyBaseObject_Type 类型的值。

有人可能会注意到，我们没有提到任何对象结构，例如元组对象结构：tupleobject，并且提问：如果没有为用户定义的类定义对象结构，那么如何处理对象实例以及未映射到该类型插槽中的对象属性在哪里 (if no object structures are defined for a user defined class then how are object instances handled and where do objects attributes that do not map to slots in the type reside) ？这与 tp_dictoffset 字段 (类型对象中的数字字段) 有关。实例实际上是作为 PyObjects 创建的，但是当实例类型中的偏移值非零时，它会指定实例属性字典与实例(PyObject) 本身之间的偏移量，如图 4.0 所示，因此对于 Person 类型的实例来说，可以通过将此偏移量添加到PyObject 原始内存位置来估计属性字典的位置。

![4.0](./images/image-20191109145506.png)

​					Figure 4.0: How instances of user defined types are structured.

例如，如果实例 PyObject 的值为 0x10，偏移量为 16，则包含实例属性的字典可以在 `0x10 + 16` 处找到。正如我们在下一节中看到的一样，这不是实例存储其属性的唯一方法。



### 4.5 Objects and their attributes

面向对象编程的核心就是类及其属性 (变量和方法) 。通常来说，类和实例使用 dict 数据结构存储其属性，但并不是所有的类和实例都是这样，例如有些定义了 __slots__ 的类和实例。如上一节所说的那样，在两个位置中的某个位置找到 dict 数据结构取决于对象的类型。

1. 对于具有 Type 类型的对象，类结构中的 tp_dict 插槽 (slot) 是指向 dict 的指针，该 dict 包含这个类的值，变量和方法。通常意义上来说，我们说类对象的数据结构中的 tp_dict 字段是指向类的 dict 的指针。 
2. 对于具有非 Type 类型的对象 (即用户定义类型的实例) ，该 dict 数据结构 (如果存在) 位于表示该对象的 PyObject 结构后面。对象类型的 tp_dictoffset 值给出了从对象开始到包含实例属性 dict 的偏移量。

做一个简单的字典访问来获取属性似乎很简单，但这还并没有结束。实际上，与检查 Type 实例的 tp_dict 值或用户定义类型的实例的 tp_dictoffset 处的 dict 相比，搜索属性更加复杂。为了更加全面的理解，我们必须讨论描述符协议，这个协议是 python 属性引用的核心。

《Descriptor HowToGuide》是对一个对描述符很好的介绍，因此这里仅提供了对描述符的粗略描述。简而言之，描述符是一个对象，它实现了描述符协议的 __get__， __set__ 或 __delete__ 特殊方法。list 4.9 显示了python 中每种方法的签名。

Listing 4.9: The Descriptor protocol methods

```python
descr.__get__(self, obj, type=None) --> value
descr.__set__(self, obj, value) --> None
descr.__delete__(self, obj) --> None
```

仅实现 __get__ 方法的对象是非数据描述符，因此它们在初始化后表现为只读，而实现 __get__ 和 __set__ 的对象是数据描述符，这意味着此类描述符对象是可写的。我们对描述符及其在表示对象属性中的应用感兴趣。list 4.10 中的 TypedAttribute 描述符是用于表示对象属性的描述符的示例。

Listing 4.10: A simple descriptor for type checking attribute values

```python
class TypedAttribute:
    def __init__(self, name, type, default=None):
        self.name = "_" + name
        self.type = type
        self.default = default if default else type()
        
    def __get__(self, instance, cls):
    	return getattr(instance, self.name, self.default)
    
    def __set__(self,instance,value):
        if not isinstance(value,self.type):
        raise TypeError("Must be a %s" % self.type)
    	setattr(instance,self.name,value)
        
    def __delete__(self,instance):
    	raise AttributeError("Can't delete attribute")
```

对用于表示类的任何属性，TypedAttribute 描述符类会强制执行基本类型检查。要注意的是，描述符仅在类级别而不是实例级别（即 list 4.11 所示的 __init__ 方法）中定义时才有效。

Listing 4.11: Type checking on instance attributes using TypedAttribute descriptor

```python
class Account:
    name = TypedAttribute("name",str)
    balance = TypedAttribute("balance",int, 42)
    
    def name_balance_str(self):
        return str(self.name) + str(self.balance)
    
>> acct = Account()
>> acct.name = "obi"
>> acct.balance = 1234
>> acct.balance
1234
>> acct.name
obi
# trying to assign a string to number fails
>> acct.balance = '1234'
TypeError: Must be a <type 'int'>
```

仔细思考一下，只有在类型级别定义此类描述符才有意义，因为如果在实例级别定义，则对该属性的任何分配都将覆盖该描述符。必须阅读 python vm 源代码，以了解 基本的描述符对于 python 来说是什么。描述符提供了 Python 中的属性，静态方法，类方法和许多其他的功能。为了具体说明描述符的重要性，需要考虑用来从实例定义的实例 b 中解析属性的算法，如 list 4.12 所示。

Listing 4.12: Algorithm for find a referenced attribute in an instance of a user defined type

------

1. type(b).__dict__ 被用来搜索属性名称。如果名称被搜索到了并且其是一个数据描述符，调用描述符的 __get__ 方法会返回相应的结果。如果属性名称没有被搜索到，然后所有的在 \*mro* 中的基类都以相同的方式进行搜索。
2. b.__dict__ 被搜索并且如果属性名称在这里被搜索到了，它就会返回.
3. ,如果从 1 中得到的名称是一个非数据描述符，那么就会返回调用 __get__ 的结果。 
4. 如果名称没有被找到，那么就会抛出 AttributeError 或者 如果是用户定义的类型就会调用 __getattr__().

------

list 4.12 中的算法表明，在属性引用期间，我们首先会去检查描述符对象；它还说明了 TypedAttribute 描述符如何能够表示对象的属性：每当引用诸如 b.name 之类的属性时，都会在 Account 类对象中搜索该属性，在这种情况下，会找到 TypedAttribute 描述符并会调用它的 __get__  方法。 TypedAttribute 示例说明了一个描述符，但是它相当刻意。为了真正了解描述符对于语言核心的重要性，我们会用一些例子来说明如何应用描述符。

请注意，list 4.12 中的属性引用算法与类型为 type 时使用的属性引用算法是不同的。list 4.13 显示了类型为type 的算法。

Listing 4.13: Algorithm to find a referenced attribute in a type

------

1. type(type).__dict__ 用来搜索属性名称。如果名称被找到并且是一个数据描述符，调用描述符的 __get__ 方法会返回相应的结果。如果属性名称没有被搜索到，然后所有的在 \*mro* 中的基类都会以用相同的方式进行搜索。

2. type.__dict__ 以及所有他的基类都是用来搜索属性名称。如果名称被找到并且它是一个描述符，那么返回一个调用它的 __get__ 方法的值；如果它是一个普通的方法，那么返回它自己。

3. 如果一个值在 1 中被发现并且它是一个非数据描述符，那么返回一个调用它的 __get__  方法的值。

4. 如果一个值在 1 中被发现并且不是一个描述符，那么返回它本身。

------

**Examples of Attribute Referencing with Descriptors inside the VM**

描述符在 Python 中的属性引用中起着非常重要的作用。思考一下本章前面讨论的类型数据结构。任何希望被视为描述符的类型实例都可以填充类型数据结构中的 tp_descr_get 和 tp_descr_set 字段。函数对象是展示其工作原理的好地方。

给一个类，如 list 4.11 中的 Account，考虑一下，当我们从 Account 类中引用 name_balance_str 方法，以及从 list 4.14 中所示的实例中引用同样的方法时会发生什么。

Listing 4.14: Illustrating bound and unbound functions

```python
>> a = Account()
>> a.name_balance_str
<bound method Account.name_balance_str of <__main__.Account object at
0x102a0ae10>>

>> Account.name_balance_str
<function Account.name_balance_str at 0x102a2b840>
```

查看 list 4.14 中的代码段，尽管我们似乎引用了相同的属性，但返回的实际对象的值和类型不同。当从Account 类型引用时，返回的值是函数类型，但是从 Account 类型的实例引用时，返回的结果是绑定方法类型。返回不同类型的原因是因为函数也是描述符。list 4.15 显示了函数对象类型的定义。

Listing 4.15: Function type object definition

```c
PyTypeObject PyFunction_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "function",
    sizeof(PyFunctionObject),
    0,
    (destructor)func_dealloc, 					/* tp_dealloc */
    0, 											/* tp_print */
    0, 											/* tp_getattr */
    0, 											/* tp_setattr */
    0, 											/* tp_reserved */
    (reprfunc)func_repr, 						/* tp_repr */
    0, 											/* tp_as_number */
    0, 											/* tp_as_sequence */
    0, 											/* tp_as_mapping */
    0, 											/* tp_hash */
    function_call, 								/* tp_call */
    0, 											/* tp_str */
    0, 											/* tp_getattro */
    0, 											/* tp_setattro */
    0, 											/* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC,	/* tp_flags */
    func_doc, 									/* tp_doc */
    (traverseproc)func_traverse, 				/* tp_traverse */
    0, 											/* tp_clear */
    0, 											/* tp_richcompare */
    offsetof(PyFunctionObject, func_weakreflist), /* tp_weaklistoffset */
    0, 											/* tp_iter */
    0, 											/* tp_iternext */
    0, 											/* tp_methods */
    func_memberlist, 							/* tp_members */
    func_getsetlist, 							/* tp_getset */
    0, 											/* tp_base */
    0, 											/* tp_dict */
    func_descr_get, 							/* tp_descr_get */
    0, 											/* tp_descr_set */
    offsetof(PyFunctionObject, func_dict), 		/* tp_dictoffset */
    0, 											/* tp_init */
    0, 											/* tp_alloc */
    func_new, 									/* tp_new */
};
```

函数对象使用 func_descr_get 函数填充 tp_descr_get 字段，因此函数类型的实例是非数据描述符。list 4.16 显示了 funct_descr_get 方法的实现。

Listing 4.16: Function type object definition

```c
static PyObject * func_descr_get(PyObject *func, PyObject *obj, PyObject *type){
    if (obj == Py_None || obj == NULL) {
        Py_INCREF(func);
        return func;
    }
    return PyMethod_New(func, obj);
}
```

如上一节所述，可以在类型属性解析或实例属性解析期间调用 func_descr_get。当从类中调 func_descr_get 时就是调用 local_get(attribute, (PyObject *)NULL, (PyObject *)type)，而从用户定义的类型的实例属性引用中调用时，调用签名为 f(descr, obj, (PyObject *)Py_TYPE(obj))。仔细阅读 list 4.16 中 func_descr_get 的实现，我们看到如果实例为 NULL，则函数将返回其自身，而当我们将实例传递给函数调用时，则会使用该函数和实例创建一个新的方法对象。这些总结了 python 如何使用描述符为相同的函数引用返回不同的类型。

- 当在类中定义方法时，我们将 self 参数用作任何实例方法的第一个参数，因为实际上，实例方法将实例（按惯例称为self）作为第一个参数。如 b.name_balance_str() 与 type(b).name_balance_str(b) 的调用实际上是相同。之所以能够调用 b.name_balance_str()，是因为 b.name_balance_str 返回的值是一个方法对象，该对象是 name_balance_str 的一个简单封装，实例已经绑定到该方法了。因此，当我们进行诸如 b.name_balance_str() 之类的调用时，该方法使用绑定的实例作为封装函数的参数，从而向我们隐藏这个细节。

关于描述符的重要性有一些其他的示例，如 list 4.17 中的代码片段，该代码片段显示了从内置类的实例和用户定义类的实例访问 __dict__ 属性的结果。

Listing 4.17: Accesing the __dict__ attribute from an instance of the builtin type and an instance of a user defined type

```python
class A:
	pass
>>> A.__dict__
mappingproxy({'__module__': '__main__', '__doc__': None, '__weakref__': <att\
ribute '__weakref__' of 'A' objects>, '__dict__': <attribute '__dict__' of 'A' objec\
ts>})
>>> i = A()
>>> i.__dict__
{}
>>> A.__dict__['name'] = 1
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: 'mappingproxy' object does not support item assignment
>>> i.__dict__['name'] = 2
>>> i.__dict__
{'name': 2}
>>>
```

从 list 4.17 中可以看出，当引用 __dict__ 属性时，两个对象都不返回普通的字典类型。类的实例返回一个支持所有常用字典功能的字典映射，类对象似乎返回了一个我们无法分配的映射代理。因此，这些对象的属性的引用方式有所不同。后面的几节中我们会对属性搜索算法进行回顾。第一步是在对象类型的 __dict__ 中搜索属性，因此我们继续对 list 4.18 中的两个对象执行此操作。

Listing 4.18: Checking for __dict__ in type of objects

```python
>>> type(type.__dict__['__dict__']) # type of A is type
<class 'getset_descriptor'>
type(A.__dict__['__dict__'])
<class 'getset_descriptor'>
```

我们看到两个对象的 __dict__ 属性都是由数据描述符表示的，这就是可以得到不同的对象类型的原因。我们想找出在此描述符的内部发生了什么，如函数和绑定方法。一个很好的切入点就是 Objects/typeobject.c 模块和 type 类的定义。tp_getset 字段中包含了一个 C 结构体组成的数组 (PyGetSetDef 值) ，如 list 4.19 所示。这是描述符对象插到 tpye 类 __dict__ 属性中的值的集合，这是类型对象的 tp_dict 槽 (slot) 指向的映射。

Listing 4.19: Checking for __dict__ in type of objects

```c
static PyGetSetDef type_getsets[] = {
    {"__name__", (getter)type_name, (setter)type_set_name, NULL},
    {"__qualname__", (getter)type_qualname, (setter)type_set_qualname, NULL},
    {"__bases__", (getter)type_get_bases, (setter)type_set_bases, NULL},
    {"__module__", (getter)type_module, (setter)type_set_module, NULL},
    {"__abstractmethods__", (getter)type_abstractmethods,
    (setter)type_set_abstractmethods, NULL},
    {"__dict__", (getter)type_dict, NULL, NULL},
    {"__doc__", (getter)type_get_doc, (setter)type_set_doc, NULL},
    {"__text_signature__", (getter)type_get_text_signature, NULL, NULL},
    {0}
};
```

这些值不是唯一将描述符插入类型 dict 的值，还有其他的值，例如 tp_members 和 tp_methods 值，这些描述符在类型初始化期间创建并插入 tp_dict 。在类上调用 PyType_Ready 函数时，会将这些值插入 dict 中。作为PyType_Ready 函数初始化过程的一部分，将为 type_getsets 中的每个条目创建描述符对象，然后将其添加到tp_dict 映射中：Objects/typeobject.c 中的 add_getset 函数将对此进行处理。回到我们的 __dict__ 属性，我们知道类型初始化之后，__dict__ 属性存在于类型的 tp_dict 字段中，因此让我们看看该描述符的 getter 函数是做什么的。 getter 函数是 list 4.20 中所示的 type_dict 函数。

Listing 4.20: Getter function for an instance of type

```c
static PyObject * type_dict(PyTypeObject *type, void *context){
    if (type->tp_dict == NULL) {
        Py_INCREF(Py_None);
        return Py_None;
    }
    return PyDictProxy_New(type->tp_dict);
}
```

tp_getattro 字段指向该函数，该函数是用于获取任何对象属性的第一个调用入口。对于类对象，它指向type_getattro 函数。这个方法又实现了 list 4.13 中描述的属性搜索算法。__dict__ 属性的 dict 类中的描述符所调用的函数是 list 4.19 中给出的 type_dict 函数。这里的返回值是包含类属性的实际字典的字典代理；这解释了查询类对象的 __dict__ 属性时返回的是 mappingproxy 类型。

那么，用户定义的类 A 的实例又如何解析 __dict_\_ 属性呢？回想一下，A 实际上是 type 类的对象，因此我们在 Object/typeobject.c 模块中搜寻以了解如何创建新的类。 PyType_Type 的 tp_new 插槽 (slot) 包含用于创建新类型对象的 type_new 函数。仔细阅读函数中的所有创建类的代码，如 listing 4.21 。

Listing 4.21: Setting tp_getset field for user defined type

```c
if (type->tp_weaklistoffset && type->tp_dictoffset)
	type->tp_getset = subtype_getsets_full;
else if (type->tp_weaklistoffset && !type->tp_dictoffset)
	type->tp_getset = subtype_getsets_weakref_only;
else if (!type->tp_weaklistoffset && type->tp_dictoffset)
	type->tp_getset = subtype_getsets_dict_only;
else
	type->tp_getset = NULL;
```

假设第一个条件为 true，则 tp_getset 字段将填充 list 4.22 中所示的值。

Listing 4.22: The getset values for instance of type

```c
static PyGetSetDef subtype_getsets_full[] = {
    {"__dict__", subtype_dict, subtype_setdict,
    PyDoc_STR("dictionary for instance variables (if defined)")},
    {"__weakref__", subtype_getweakref, NULL,
    PyDoc_STR("list of weak references to the object (if defined)")},
    {0}
};
```

调用 (*tp _> tp_getattro)(v, name) 时，将会调用 tp_getattro 字段，其包含指向 PyObject_GenericGetAttr 的指针。该函数负责为用户定义的类实现属性搜索算法。对于 __dict__ 属性，在对象类型的 dict 中找到描述符，而描述符的 __get__ 函数是 list 4.21 中在 __dict__ 属性定义的 subtype_dict 函数。list 4.23 显示了 subtype_dict 的 getter 函数。

Listing 4.23: The getter function for __ attribute of a user-defined type

```c
static PyObject * subtype_dict(PyObject *obj, void *context){
    PyTypeObject *base;
    base = get_builtin_base_with_dict(Py_TYPE(obj));
    if (base != NULL) {
        descrgetfunc func;
        PyObject *descr = get_dict_descriptor(base);
        if (descr == NULL) {
            raise_dict_descr_error(obj);
            return NULL;
        }
        func = Py_TYPE(descr)->tp_descr_get;
        if (func == NULL) {
            raise_dict_descr_error(obj);
            return NULL;
        }
        return func(descr, obj, (PyObject *)(Py_TYPE(obj)));
    }
    return PyObject_GenericGetDict(obj, context);
}

```

当对象实例处于继承层次中时，get_builtin_base_with_dict 会返回一个值，因此该实例忽略此函数是没问题的。 PyObject_GenericGetDict 对象被调用。list 4.24 显示了 PyObject_GenericGetDict 和实际获取实例字典的相关的帮助。实际上获取 dict 函数的是 _PyObject_GetDictPtr 函数，该函数查询对象的 dictoffset 并使用该函数计算实例 dict 的地址。在此函数返回空值的情况下，PyObject_GenericGetDict 可以继续向调用的函数返回一个新字典。

Listing 4.24: Fetching dict attribute of an instance of a user defined type

```c
PyObject * PyObject_GenericGetDict(PyObject *obj, void *context){
    PyObject *dict, **dictptr = _PyObject_GetDictPtr(obj);
    if (dictptr == NULL) {
    	PyErr_SetString(PyExc_AttributeError,
    					"This object has no __dict__");
    	return NULL;
    }
    dict = *dictptr;
    if (dict == NULL) {
    	PyTypeObject *tp = Py_TYPE(obj);
        if ((tp->tp_flags & Py_TPFLAGS_HEAPTYPE) && CACHED_KEYS(tp)) {
            DK_INCREF(CACHED_KEYS(tp));
            *dictptr = dict = new_dict_with_shared_keys(CACHED_KEYS(tp));
        }
        else {
        	*dictptr = dict = PyDict_New();
        }
    }
    Py_XINCREF(dict);
    return dict;
}

PyObject ** _PyObject_GetDictPtr(PyObject *obj){
    Py_ssize_t dictoffset;
    PyTypeObject *tp = Py_TYPE(obj);
    dictoffset = tp->tp_dictoffset;
    if (dictoffset == 0)
        return NULL;
    if (dictoffset < 0) {
        Py_ssize_t tsize;
        size_t size;
        
        tsize = ((PyVarObject *)obj)->ob_size;
        if (tsize < 0)
            tsize = -tsize;
        size = _PyObject_VAR_SIZE(tp, tsize);
        
        dictoffset += (long)size;
        assert(dictoffset > 0);
        assert(dictoffset % SIZEOF_VOID_P == 0);
    }
    return (PyObject **) ((char *)obj + dictoffset);
}
```

该解释简要总结了如何根据类型使用描述符来实现自定义属性访问。在整个VM中，对于使用描述符执行属性访问的其他实例，使用上述相同策略。描述符在VM中无处不在。 __slots__，静态方法和类方法，属性只是使用描述符实现的语言功能的进一步示例。

### 4.6 Method Resolution Order (MRO)

在讨论属性引用时，我们已经提到了 mro，但是由于没有进行过多讨论，因此在本节中，我们将对 mro 进行更加详细的介绍。在 python 中，类可以属于多重继承的层次结构，因此当一个类从多个类继承时，需要一种顺序来搜索方法。正如我们在属性参考解析算法中所看到的那样，在搜索其他非方法的属性时，实际上也是使用了这种称为 *Method Resolution Order* (MRO) 的顺序。 “ Python 2.3 Method Resolution Order” 一文是一篇出色且易于阅读的文档，介绍了python中使用的方法解析算法。这里总结了主要要点。
当类型从多个基本类型继承时，Python使用C3⁸算法来构建方法的解析顺序（在此也称为线性化）。清单4.25显示了一些用于解释该算法的符号。

```
C1 C2 ... CN 指示类的列表 [C1, C2, C3 .., CN]

列表的头是它的第一个元素: head = C1

列表的尾部是剩余的所有元素: tail = C2 ... CN.

C + (C1 C2 ... CN) = C C1 C2 ... CN 表示了列表之和 [C] +
[C1, C2, ... ,CN].
```

考虑多重继承层次结构中的类型C，其中 C 继承自基本类型B1，B2，...，BN，则 C 的线性化是 C 的加上父项的线性化与父项的列表的总和。L[C (B1 ... BN) ] = C + merge (L [B1] ... L [BN], B1 ... BN) 。没有父对象的对象类型的线性化是微不足道的，L[object] = object。合并操作是根据以下算法计算的：

- 以第一个列表的开头，即L[B1]\[0]；如果此头不在任何其他列表的尾部，则将其添加到 C 的线性化中，然后从合并中的列表中将其删除，否则，查看下一个列表的 head 并使用它，如果它是一个好的 head 然后重复该操作，直到所有类都被删除，或者不可能找到好的 head 。在这种情况下，不可能进行合并，Python 2.3 将拒绝创建类 C 并且引发异常。

使用此算法无法线性化某些层次结构的类，在这种情况下，VM会引发错误，并且不会创建此层次结构的类。

![](./images/image-20191119224850.png)

假设我们具有如图 4.1 所示的继承层次结构，则创建 mro 的算法将从层次结构的顶部开始依次为 O，A 和 B。O，A 和 B 的线性化很简单：

Listing 4.26: Calculating linearization for types O, A and B from figure 4.1

------

L[O] = O
		L[A] = A O
		L[B] = B O

------

可以将 X 的线性化计算为L[X] = X + merge(AO, BO, AB)

A 是一个很好的 head，因此将其添加到线性化中，然后剩下的就是计算merge(O, BO, B)。 O 不是好的 head，因为它位于 BO 的尾部，因此我们跳到下一个序列。 B是一个很好的 head ，因此我们将其添加到线性化中，然后剩下的就可以计算归并为 O 的merge(O, O)。所得的 X的 线性化L [X] = X A B O。

使用与上述相同的过程， Y 的线性化的计算如 list 4.27 所示：

Listing 4.27: Calculating linearization for type Y from figure 4.1

------

L[Y] = Y + merge(AO, BO, AB)
				= Y + A + merge(O, BO, B)
				= Y + A + B + merge(O, O)
				= Y A B O

------

计算 X 和 Y 的线性化后，我们现在可以计算 Z 的线性化，如 list 4.28 所示。

Listing 4.28: Calculating linearization for type Z from figure 4.1

------

L[Z] = Z + merge(XABO, YABO, XY)
				= Z + X + merge(ABO, YABO, Y)
				= Z + X + Y + merge(ABO, ABO)
				= Z + X + Y + A + merge(BO, BO)
				= Z + X + Y + A + B + merge(O, O)
				= Z X Y A B O

------


