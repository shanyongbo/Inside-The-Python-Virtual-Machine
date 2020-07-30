---
layout: post
title:  "Inside The Python Virtual Machine -- 1、Introduction"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true
---

本文是 Inside The Python Virtual Machine 中第一章介绍部分的中文内容。

<!-- more -->



## 1、Introduction

python这门语言已经存在有一段时间了。Guido Van Rossum 于 1989 年开始对第一版进行开发，此后 python 成为最受欢迎的语言之一，被广泛用于图形界面，财务和数据分析应用中。  

本文旨在深入介绍 Python 解释器，并提供有关 python 程序如何执行的概念性概述。在撰写本文时， CPython 是 Python 最受欢迎的实现，并被视为标准。

python 程序的执行一般分为以下两个或三个主要阶段，区分方式具体取决于解释器的调用方式。在本文中，下面这些包含在不同的部分中：

1. Initialization：这涉及到 python 进程所需的各种数据结构的建立。这可能仅在通过解释器 shell 非交互地执行程序时才会有这一步骤。

2. Compiling：这涉及诸如解析源代码以构建语法树 (syntax trees) ，创建抽象语法树 (abstract syntax trees) ，构建符号表 (ymbol tables) 以及生成代码对象 (code objects) 之类的活动。

3. Interpreting：这涉及在某些上下文中实际执行生成的代码对象。  

从源代码生成解析树和抽象语法树的过程与语言无关，因此适用于其他语言的相同方法也适用于 Python ；所以这里没有涉及这个方面的主题。另一方面，从“抽象语法树“构建符号表和代码对象的过程是编译阶段中比较有趣的部分，该过程或多或少地会以 python 特定的方式处理。本文还介绍了编译后的代码对象以及该过程中使用的所有数据结构。这将涉及包括但不限于构建符号表和生成代码对象，Python 对象， frame 对象，代码对象，函数对象，python 操作码，解释器循环，生成器和用户定义类的过程等。