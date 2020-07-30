---
layout: post
title:  "Inside The Python Virtual Machine --  8、Intermezzo The abstract.c Module"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第八章介绍部分的中文内容。

<!-- more -->


## 8. Intermezzo: The abstract.c Module

到目前为止，我们已经多次提到 python 虚拟机通常将执行的值 (values for evaluation) 视为 PyObjects。这就留下了一个明显的问题：如何在此类通用对象上安全地执行操作？例如，当执行 (evaluating) 字节码指令 BINARY_ADD 时，会从执行堆栈中弹出两个 PyObject 值，并将其用作加法运算的参数，但是虚拟机如何知道这些值是否实际实现了加法运算所属的协议？

要了解 PyObjects 上的许多操作是如何工作，我们只需要查看 Objects/Abstract.c 模块。该模块定义了许多对实现给定对象协议的对象起作用的函数。这意味着，例如，如果一个对象要加上两个对象，则此模块中的 add 函数将期望两个对象都实现 tp_numbers slots 的 __add__ 方法。解释这个问题的最佳方法是举例说明。

考虑 BINARY_ADD 操作码的情况，当将它应用于两个数字的加法运算时，将调用 Objects/Abstract.c 模块的 PyNumber_Add 函数。list 8.1 中提供了此功能的定义。

Listing 8.1: Generic add function from abstract.c module

```c
PyObject * PyNumber_Add(PyObject *v, PyObject *w){
    PyObject *result = binary_op1(v, w, NB_SLOT(nb_add));
    if (result == Py_NotImplemented) {
        PySequenceMethods *m = v->ob_type->tp_as_sequence;
        Py_DECREF(result);
        if (m && m->sq_concat) {
            return (*m->sq_concat)(v, w);
        }
        result = binop_type_error(v, w, "+");
    }
    return result;
}
```

list 8.1 中 PyNumber_Add 函数的第 2 行对 binary_op1 函数的调用十分有趣。 binary_op1 函数是另一个通用函数，该函数的参数中包扩两个数值类型或数值类型的子类，并将一个二进制函数应用于这两个值。 NB_SLOT 宏将给定方法的偏移量返回到 PyNumberMethods 结构中；回想一下，此结构是处理数值方法的集合。list 8.2 中包含此类 binary_op1 函数的定义，下面的是对该函数的深入说明。

Listing 8.2: The generic binary_op1 function

```c
static PyObject * binary_op1(PyObject *v, PyObject *w, const int op_slot){
    PyObject *x;
    binaryfunc slotv = NULL;
    binaryfunc slotw = NULL;
    
    if (v->ob_type->tp_as_number != NULL)
        slotv = NB_BINOP(v->ob_type->tp_as_number, op_slot);
    if (w->ob_type != v->ob_type &&
        w->ob_type->tp_as_number != NULL) {
        slotw = NB_BINOP(w->ob_type->tp_as_number, op_slot);
        if (slotw == slotv)
            slotw = NULL;
    }
    if (slotv) {
        if (slotw && PyType_IsSubtype(w->ob_type, v->ob_type)) {
            x = slotw(v, w);
            if (x != Py_NotImplemented)
                return x;
            Py_DECREF(x); /* can't do it */
            slotw = NULL;
        }
        x = slotv(v, w);
        if (x != Py_NotImplemented)
            return x;
        Py_DECREF(x); /* can't do it */
    }
    if (slotw) {
        x = slotw(v, w);
        if (x != Py_NotImplemented)
            return x;
        Py_DECREF(x); /* can't do it */
    }
    Py_RETURN_NOTIMPLEMENTED;
}
```

1. 该函数接受三个值，两个 PyObject *v 和 *w 以及一个整数 operation slot，它是在 PyNumberMethods 结构中的偏移量。 
2. 第 3 行和第 4 行定义了两个值 slotv 和 slotw ，它们是表示其类型所建议的二进制函数的结构。

3. 从第 3 行到第 13 行，我们尝试对 v 和 w 取消对 op_slot 参数给出的函数的引用。在第 8 行，会检查两个值是否具有相同的类型，如果两个值具有相同的类型，则不需要取消在 op_slot 中对第二个值的函数的引用。即便两个值不是同一类型，但只要从两个值取消引用的函数是相等的，那么 slotw 值就被清空。 
4. 取消引用二进制函数后，如果 slotv 不为 NULL，然后在会第 15 行中，检查 slotw 是否也为 NULL，并且 w 的类型是否是 v 的子类型，如果上面的结果为 true，则 v 和 w会被应用到 slotw 函数中。发生这种情况的原因就是继承之后的方法就是我们想要使用的方法。如果 w 不是子类型，则会在第 22 行将 slotv 应用于这两个值。
5. 到达第 27 行意味着 slotv 函数为 NULL，因此 slotw 只要不为 NULL，那么我们就对 v 和 w 应用 slotw 引用。 
6. 如果 slotv 和 slotw 都不包含函数，则返回 Py_NotImplemented。 Py_RETURN_NOTIMPLEMENTED 只是一个宏，该宏会在返回 Py_NotImplemented 值之前增加其引用计数。

上面给出的解释的想法是，虚拟机如何能对提供给它的值执行对应操作的蓝图。我们在这里通过忽略可以重载的操作码来简化一些事情，例如 + 符号映射到 BINARY_ADD 操作码，并且可以应用于字符串，数字或序列，但是在上面的示例中，我们只看了适用于数字和数字子类。很难想象如何处理重载操作。在 BINARY_ADD 的情况下，如果查看 PyNumber_Add 函数，则可以看到，如果从 binary_op1 调用返回的值是 Py_NotImplemented，则虚拟机将尝试将这些值视为序列，并尝试取消引用序列连接的方法，然后将它们应用于两个值 (如果它们实现了序列协议) 。回到 ceval.c 中的解释器循环 (interpeter  loop) ，当我们看到对 BINARY_ADD 操作码进行执行 (evaluation) 的情况时，我们会看到以下代码段。

Listing 8.3: ceval implementation of binary add

```c
PyObject *right = POP();
PyObject *left = TOP();
PyObject *sum;
if (PyUnicode_CheckExact(left) &&
    PyUnicode_CheckExact(right)) {
    sum = unicode_concatenate(left, right, f, next_instr);
    /* unicode_concatenate consumed the ref to left */
}
else {
    sum = PyNumber_Add(left, right);
    Py_DECREF(left);
}
```

在讨论解释器循环时，请忽略第 1 行和第 2 行。从其余片段中我们看到的是，当我们遇到 BINARY_ADD 时，调用的第一个操作是检查两个值是否是字符串，以便将字符串连接方法应用于这些值上。如果不是字符串，则将 Objects/Abstract.c 中的 PyNumber_Add 函数应用于这两个值。尽管代码在 Python/ceval.c 中完成的字符串检查以及在 Objects/Abstract.c 中完成的数字和序列检查看起来似乎有些混乱，但是很明显，当我们有一个重载的操作码时，会发生什么。

上面提供的解释是大多数操作码操作的处理方式：检查要计算的值的类型，然后根据需要取消引用该方法并应用于参数值。

