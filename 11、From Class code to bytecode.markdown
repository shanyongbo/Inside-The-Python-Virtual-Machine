---
layout: post
title:  "Inside The Python Virtual Machine --  11、From Class code to bytecode"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true
---

本文是 Inside The Python Virtual Machine 中第11章介绍部分的中文内容。

<!-- more -->



## 11. From Class code to bytecode

我们已经讨论了很多基础知识，讨论了 python 虚拟机或解释器如何执行代码的细节，但是对于像 Python 这样的面向对象的语言，我们实际上忽略了最重要的方法之一：用户定义的类如何精简为字节码并进行执行的基本内容。

通过对 Python 对象的讨论，我们对如何创建新的类有一个粗略的看法，但是直觉可能无法完全清晰的捕捉类创建的整个过程：从用户的定义到创建新类对象的实际字节码的过程；因此本章主要为了弥补这一点，并就此过程的发生方式进行阐述。

通常，我们会从一个非常简单的用户定义的类开始，如 lisitng 11.0 所示。

Listing 11.0: A simple class definition

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

当将包含上述类定义的模块用 dis 模块进行反汇编时，将会输出如 list 11.1 中所示的字节码流。

Listing 11.1: A simple class definition

```python
    0 LOAD_BUILD_CLASS
    2 LOAD_CONST 						0 (<code object Person at 0x102298b70, file "str\
    ing", line 2>)
    4 LOAD_CONST 						1 ('Person')
    6 MAKE_FUNCTION 					0
    8 LOAD_CONST 						1 ('Person')
    10 CALL_FUNCTION 					2
    12 STORE_NAME 						0 (Person)
    14 LOAD_CONST 						2 (None)
    16 RETURN_VALUE
```

我们对字节 0 到字节 12 感兴趣，因为它们是创建新类对象并存储它的实际操作码，以便可以通过其名称 (在本例中为 Person) 进行引用。在这之前，我们在上面的操作码上进行扩展，然后看一下 Python 文档所指定的类创建过程。

文档中对过程的描述虽然非常高级，但是非常清楚。可以从 python 文档中推测出来，类创建的幕后过程大致涉及以下过程，但是没有特定的顺序。

1. 类语句的主体被隔离到一个代码对象中。 
2. 确定用于类实例化的适当元类。 
3. 准备代表该类命名空间的类字典。 
4. 代表类主体的代码对象在此命名空间内执行。 
5. 创建类对象。

在最后一步中，通过实例化 type 类并传入类名称，基类和类字典作为参数来创建类对象。任何 __prepare__钩子都在实例化类对象之前运行。通过在类定义中提供 metaclass 关键字，可以显式指定在类对象创建中使用的元类。如果未提供，则类语句将会检查存在的基类元组中的第一个条目。如果没有使用基类，则搜索全局变量 __metaclass__，如果没有找到此值，Python 将使用默认的元类。

在随后的章节中将讨论有关元类的更多信息。整个类创建过程始于将 __build_class 函数加载到值堆栈上。该功能负责创建新类的所有繁重工作。为此，我们在编译阶段完成了上述创建过程的第一步。在此步骤中，代表类主体的代码对象通过偏移量为 2 的 LOAD_CONST 的指令加载到堆栈上。该代码对象由 MAKE_FUNCTION 操作码包装到函数对象中，并且很快就会明白为什么会发生这种情况。到那时，执行循环到达偏移量为 10 的地方，执行堆栈 (evaluation loop) 看起来会是类似于图 11.0 的结构。

![](./images/image-20191130160314.png)

在偏移量为 10 的地方，CALL_FUNCTION 以其在执行堆栈 (evaluation loop) 上的值作为参数调用__build_class 函数。此函数在 Python/bltinmodule.c 模块中定义。该函数的主要部分专用于完整性检查：检查是否提供了正确的参数，它们是否具有正确的类型等。在进行这些完整性检查之后，该函数必须决定正确的元类。我们阐释了确定正确的元类的规则，正如 python 文档中所述。

1. 如果没有给出基数并且没有给出显式元类，则使用 type()；
2. 如果给出了显式元类并且它不是 type() 的实例，则将其直接用作元类；
3. 如果是 type() 的实例并且显示给出，或者定义了基类，那么使用派生程度最高的元类。

从派生的显式指定的元类 (如果有) 和所有指定的基类的元类 (即 type(cls)) 中选择派生最多的元类。最多的派生元类是所有这些候选元类的子类型。如果没有任何候选元类满足该条件，则该类定义将失败并显示 TypeError。

list 11.2 中显示了用于处理元类解析的 __build_class 函数的实际代码段，并对其进行了注释，以提供更加清晰的理解。

Listing 11.2: A simple class definition

```c
...
    /* kwds are values passed into brackets that follow class name
    e.g class(metalcass=blah)*/
    if (kwds == NULL) {
        meta = NULL;
        mkw = NULL;
    }
    else {
    	mkw = PyDict_Copy(kwds); /* Don't modify kwds passed in! */
        if (mkw == NULL) {
            Py_DECREF(bases);
            return NULL;
        }
        /* for all intent and purposes &PyId_metaclass references the string "me\
        taclass"
        but the &PyId_* macro handles static allocation of such strings */
        meta = _PyDict_GetItemId(mkw, &PyId_metaclass);
        if (meta != NULL) {
        	Py_INCREF(meta);
            if (_PyDict_DelItemId(mkw, &PyId_metaclass) < 0) {
                Py_DECREF(meta);
                Py_DECREF(mkw);
                Py_DECREF(bases);
                return NULL;
            }
            /* metaclass is explicitly given, check if it's indeed a class */
            isclass = PyType_Check(meta);
        }
    }
if (meta == NULL) {
    /* if there are no bases, use type: */
    if (PyTuple_GET_SIZE(bases) == 0) {
        meta = (PyObject *) (&PyType_Type);
    }
    /* else get the type of the first base */
    else {
        PyObject *base0 = PyTuple_GET_ITEM(bases, 0);
        meta = (PyObject *) (base0->ob_type);
    }
    Py_INCREF(meta);
    isclass = 1; /* meta is really a class */
}
...
```

找到元类后，然后 __ build_class 继续检查元类上是否存在任何 __prepare__ 属性；如果存在任何此类属性，则通过执行 __prepare__ 钩子来传递类名，类基和类定义中的任何其他关键字参数，从而准备类命名空间。该钩子可用于自定义类行为。list 11.3 中的示例摘自元类定义和 python 文档使用示例，该示例显示了如何使用 __prepare__ 钩子来实现具有属性顺序的类。

Listing 11.3: A simple meta-class definition

```python
class OrderedClass(type):
    @classmethod
    def __prepare__(metacls, name, bases, **kwds):
    	return collections.OrderedDict()
    
    def __new__(cls, name, bases, namespace, **kwds):
        result = type.__new__(cls, name, bases, dict(namespace))
        result.members = tuple(namespace)
        return result

class A(metaclass=OrderedClass):
    def one(self): pass
    def two(self): pass
    def three(self): pass
    def four(self): pass

>>> A.members
('__module__', 'one', 'two', 'three', 'four')
```

如果在元类上没有定义 __prepare__ 属性，则 __build_class 函数将返回一个空的新字典，但是如果存在一个 __prepare__ 属性，则使用的命名空间是执行 __prepare__ 属性的结果，如 list 11.4 所示。

Listing 11.4: Preparing for a new class

```c
...
    // get the __prepare__ attribute
    prep = _PyObject_GetAttrId(meta, &PyId___prepare__);
    if (prep == NULL) {
        if (PyErr_ExceptionMatches(PyExc_AttributeError)) {
            PyErr_Clear();
            ns = PyDict_New(); // namespace is a new dict if __prepare__ is not defined
        }
        else {
            Py_DECREF(meta);
            Py_XDECREF(mkw);
            Py_DECREF(bases);
            return NULL;
        }
    }
	else {
        /** where __prepare__ is defined, the namespace is the result of executing
        the __prepare__ attribute **/
        PyObject *pargs[2] = {name, bases};
        ns = _PyObject_FastCallDict(prep, pargs, 2, mkw);
        Py_DECREF(prep);
    }
	if (ns == NULL) {
        Py_DECREF(meta);
        Py_XDECREF(mkw);
        Py_DECREF(bases);
        return NULL;
	}
...
```

在处理 __prepare__ 钩子之后，现在该创建实际的类对象了。首先，在上一段创建的命名空间中执行类主体的代码对象。要理解为什么会这样，我们将对 list 11.5 中定义的类主体的代码对象进行反汇编。

Listing 11.5: Disassembly of code object for class body from listing 11.0

```python
1 	0 LOAD_NAME 						0 (__name__)
	2 STORE_NAME 						1 (__module__)
	4 LOAD_CONST 						0 ('test')
	6 STORE_NAME 						2 (__qualname__)
2 	8 LOAD_CONST 						1 (<code object __init__ at 0x102a80660, fi\
le "string", line 2>)
	10 LOAD_CONST 						2 ('test.__init__')
	12 MAKE_FUNCTION 					0
	14 STORE_NAME 						3 (__init__)
	16 LOAD_CONST 						3 (None)
	18 RETURN_VALUE
```

当执行此代码对象时，命名空间将包含类的所有属性，即类属性，方法等。然后，该命名空间将在过程的下一阶段用作对元类的函数调用的参数，如 list 11.6 所示。 

Listing 11.6: Invoking a metaclass to create a new class instance

```c
// evaluate code object for body within namespace
none = PyEval_EvalCodeEx(PyFunction_GET_CODE(func), PyFunction_GET_GLOBALS(func), ns,
                         NULL, 0, NULL, 0, NULL, 0, NULL,
                         PyFunction_GET_CLOSURE(func));
if (none != NULL) {
    PyObject *margs[3] = {name, bases, ns};
    /**
    * this will 'call' the metaclass creating a new class object
    **/
    cls = _PyObject_FastCallDict(meta, margs, 3, mkw);
    Py_DECREF(none);
}
```

假设我们正在使用类型元类，则调用类型意味着在类的 tp_call 插槽 (slot) 中取消引用属性。然后， tp_call 函数反引用 tp_new 插槽 (slot) 中的属性，该属性实际创建并返回我们全新的类对象。然后将返回的 cls 值放回堆栈中，并存储到 Person 变量中。我们已经有了这个类和创建了一个新类的过程，而这实际上就是 Python 所具有的全部功能。

