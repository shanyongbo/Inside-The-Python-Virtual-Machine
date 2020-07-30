---
layout: post
title:  "Inside The Python Virtual Machine --  3、Compiling Python Source Code"
categories: "Inside The Python Virtual Machine"
tags: [python, vm] 
author: Shan
toc: true

---

本文是 Inside The Python Virtual Machine 中第三章介绍部分的中文内容。

<!-- more -->


## 3、Compiling Python Source Code

一般来说，尽管 python 不被认为是一种编译语言，但实际上它就是一种编译型语言。在编译期间，一些 python 源代码会被转换为虚拟机可执行的字节码。但是，在 python 中这个编译的过程非常的简单，它并不涉及太多复杂的步骤。一个 python 程序的编译过程会涉及以下步骤：

1. 将 python 源代码解析为解析树。 
2. 将解析树转换为抽象语法树 (AST) 。 
3. 生成符号表。 
4. 从 AST 生成代码对象。此步骤包括：
   	1. 将 AST 转换为控制流程图。 
    2. 从控制流程图中生成代码对象。

将源代码解析为解析树并将该解析树转换为 AST 是一个标准过程，而 python 并没有引入任何复杂而细微区别，因此本章的重点是将 AST 转换为控制流图以及从控制流程图中生成代码对象。如果你对解析树和 AST 生成感兴趣，《dragon book》会提供一些对这两个主题的更加深入的解释。

### 3.1 From Source To Parse Tree

python 的解析器是 LL(1) 解析器，它是基于《Dragon book》中对此类解析器的描述。Grammar/Grammar模块包含了对于 python 来说的 Extended Backus-Naur Form（EBNF）语法规范。list 3.0 中显示了这个规范大致情况。

Listing 3.0: A cross section of the Python BNF Grammar

```c
stmt: simple_stmt | compound_stmt
simple_stmt: small_stmt (';' small_stmt)* [';'] NEWLINE
small_stmt: (expr_stmt | del_stmt | pass_stmt | flow_stmt |
	import_stmt | global_stmt | nonlocal_stmt | assert_stmt)
expr_stmt: testlist_star_expr (augassign (yield_expr|testlist) |
		('=' (yield_expr|testlist_star_expr))*)
testlist_star_expr: (test|star_expr) (',' (test|star_expr))* [',']
augassign: ('+=' | '-=' | '*=' | '@=' | '/=' | '%=' | '&=' | '|=' | '^='
	| '<<=' | '>>=' | '**=' | '//=')

del_stmt: 'del' exprlist
pass_stmt: 'pass'
flow_stmt: break_stmt | continue_stmt | return_stmt | raise_stmt |
	yield_stmt
break_stmt: 'break'
continue_stmt: 'continue'
return_stmt: 'return' [testlist]
yield_stmt: yield_expr
raise_stmt: 'raise' [test ['from' test]]
import_stmt: import_name | import_from
import_name: 'import' dotted_as_names
import_from: ('from' (('.' | '...')* dotted_name | ('.' | '...')+)
	'import' ('*' | '(' import_as_names ')' | import_as_names))
import_as_name: NAME ['as' NAME]
dotted_as_name: dotted_name ['as' NAME]
import_as_names: import_as_name (',' import_as_name)* [',']
dotted_as_names: dotted_as_name (',' dotted_as_name)*
dotted_name: NAME ('.' NAME)*
global_stmt: 'global' NAME (',' NAME)*
nonlocal_stmt: 'nonlocal' NAME (',' NAME)*
assert_stmt: 'assert' test [',' test]

...
```

当在命令行上执行一个传递给解释器的模块时，会调用 PyParser_ParseFileObject 函数解析这个模块。该函数调用标记 (tokenization) 函数 PyTokenizer_FromFile，并将模块的文件名作为参数传递。标记函数将模块的内容分解为合法的 python tokens，或者在发现非法值时引发异常。

### 3.2 Python tokens

Python 源代码由 tokens 组成。例如，return 是关键字 token，2 是数字 token 等等。解析 python 源代码时的第一个任务是标记化源文件，将其分解为 token。 Python 有许多类型的 token，如下所示。

1. 标识符 (identifiers)：这些名称是程序员定义的，包括函数名称，变量名称，类名称等。它们必须符合 python 文档中指定的标识符规则。 
2. 运算符 (operators)：这些是特殊符号，例如 +，\*，它们对数据进行运算并产生结果。 
3. 分隔符 (delimioters)：这组符号用于对表达式进行分组，提供标点符号以及赋值。此类别中的示例包括(，)，{，}，=，*= 等。
4. 字面量 (literals)：这些是为某些类型提供定值的符号。其中有字符串和字节，例如 “Fred”，b“Fred” 和数值，包括整型 (例如2)，浮点型 (例如1e100) 和虚数类型 (例如10j)。 
5. 注释 (comments)：以哈希符号开头的字符串。注释 token 始终在物理行结尾处结束。 
6. NEWLINE：这是一个特殊 token，表示逻辑行的结尾。 
7. INDENT 和 DEDENT：这些 token 用于表示复合语句分组的缩进级别。

一组由 NEWLINE token 划定的 tokens 组成一条逻辑行，因此我们可以说 python 程序由一系列逻辑行组成，每条逻辑行都由 NEWLINE token 划定。这些逻辑行映射到 python 的语句。这些逻辑行均由多个物理行组成，每个物理行以一个行尾序列终止 (an end-of-line sequence)。在 python 中，大多数情况下逻辑行都会映射到物理行，因此逻辑行由行尾字符分隔。如图 3.0 所示，复合语句可以跨越多个物理行。当表达式位于括号，方括号或花括号中时，逻辑行可以隐式连接在一起，也可以使用反斜杠字符将逻辑行显式连接在一起。缩进在 python 语句中也起着核心作用。因此，python 语法中的一行是：simple_stmt | NEWLINE INDENT stmt + DEDENT，因此 python tokenizer 的主要任务之一就是生成 indent 和 dedent tokens，这些 tokens 将会加入到解析树中。tokenizer 使用堆栈来跟踪缩进，并且使用 list 3.1 中的算法来生成 INDENT 和 DEDENT tokens。

Listing 3.1: Python indentation algorithm for generting INDENT and DEDENT tokens

------

使用 0 初始化缩进堆栈。
		对于考虑了行连接的每个逻辑行：
				A. 如果当前行的缩进大于堆栈顶部的缩进
						1.将当前行的缩进添加到堆栈顶部。 
						2.生成一个 INDENT token。 
				B. 如果当前行的缩进小于堆栈顶部的缩进
						1.如果堆栈上没有与当前行匹配的缩进级别，则报错。 
						2.对于每个在堆栈顶部且不等于当前行的缩进。
								a. 从堆栈顶部删除该值。 
								b.生成一个 DEDENT token。 
				C. tokenizer 当前行。 
		对于堆栈上除 0 以外的每个缩进，生成一个 DEDENT token。

------

Parser/parsetok.c 模块中的 PyTokenizer_FromFile 函数从左到右，从上到下扫描 python 源文件，对文件内容进行 tokenize 。除终止符外的空白字符也可当做分隔字符 (delimit token)，但这不是必需的。在有歧义的地方 (例如 2 + 2)，一个 token 由从右到左读取的最长字符串构成一个合法的 token。在此示例中，tokens 是字面量 2，运算符 + 和字面量 2。

将从 tokenizer 生成的 tokens 传递到解析器，解析器尝试根据 list 3.0 中指定 python 语法子集构建解析树。当解析器遇到违反语法的 token 时，将引发 SyntaxError 这个异常。从解析器中输出的是一个解析树。python 的parser 模块对一块 python 代码的解析树提供了有限的访问，list 3.2 展示了使用这些得到了一个完整的解析树的示例。

Listing 3.2: Using the parser module to obtain the parse tree of python code

```python
>>>code_str = """def hello_world():
					return 'hello world'
			  """
>>> import parser
>>> from pprint import pprint
>>> st = parser.suite(code_str)
>>> pprint(parser.st2list(st))
[257,
[269,
[294,
[263,
	[1, 'def'],
	[1, 'hello_world'],
	[264, [7, '('], [8, ')']],
	[11, ':'],
	[303,
	[4, ''],
	[5, ''],
	[269,
	[270,
	[271,
		[277,
		[280,
		[1, 'return'],
		[330,
		[304,
			[308,
			[309,
			[310,
			[311,
				[314,
				[315,
				[316,
				[317,
					[318,
					[319,
					[320,
					[321,
						[322, [323, [3, '"hello world"']]]]]]]]]]]]]]]]]]]],
	[4, '']]],
	[6, '']]]]],
[4, ''],
[0, '']]
>>>
```

只要提供的源代码在语法上是正确的，上面 list 3.2 中的 parser.suite(source) 调用就会从提供的源代码中返回一个 parse tree(ST) 对象，即 parse tree 的 python 中间表示形式。parser.st2list 调用返回以 python 列表形式表示的真正的 parse tree。列表中的第一项是整数，它标识 python 语法中的生产规则。

![3.0](./images/image-20191102143901.png)

Figure 3.0: A parse tree for listing 3.2 (function that returns the ‘hello world’ string)

图 3.0 是一个树形图，显示了 list 3.2 中的相同 parse tree，其中一些 tokens 被去除，并且可以看到部分整数值表示的语法部分。这些生成规则均在 Include/token.h (terminals) 和 Include/graminit.h (terminals) 头文件中指定。

在 CPython 虚拟机中，树这种数据结构用于表示 parse tree。每个生成规则都是树数据结构上的一个节点。list 3.3 中的 Include/node.h 显示了这个节点数据结构。

Listing 3.3: The node data structure used in the python virtual machine

```c
typedef struct _node {
	short 				n_type;
    char 				*n_str;
    int 					n_lineno;
    int 					n_col_offset;
    int 					n_nchildren;
    struct _node 		*n_child;
} node;
```

在遍历 parse tree 时，我们可以查询节点的类型，子节点 (如果有的话)，导致给定节点创建的行号等等。在 Include/node.h 文件中也定义了与 parse tree 节点进行交互的宏。

### 3.3 From Parse Tree To Abstract Syntax Tree

编译过程的下一个阶段是将 python 解析树 (parse tree) 转换为抽象语法树 (AST) 。抽象语法树是独立于python 语法的代码表示形式。例如，解析树包含如图 3.0 所示的冒号节点 (colon node) ，因为它是一种语法结构，但 AST 将不包含如 list 3.4 所示的语法结构。

Listing 3.4: Using the ast module to manipulate the AST of python source code

```python
>>> import ast
>>> import pprint
>>> node = ast.parse(code_str, mode="exec")
>>> ast.dump(node)
("Module(body=[FunctionDef(name='hello_world', args=arguments(args=[], "
'vararg=None, kwonlyargs=[], kw_defaults=[], kwarg=None, defaults=[]), '
"body=[Return(value=Str(s='hello world'))], decorator_list=[], "
'returns=None)])')
```

在文件 Parser/Python.asdl 文件中可以找到各种 Python 的 AST 节点的定义。 AST 中的大多数定义都与特定来源的结构相对应，例如 if 语句或属性查找。与 python 解释器捆绑在一起的 ast 模块为我们提供了操作 python  AST 的能力。诸如 codegen 之类的工具可以在 python 中使用 AST 进行表示，并输出相应的 python 源代码。在  CPython 的实现中，AST 节点由 C 的结构表示，正如在 Include/Python-ast.h 中所定义的一样。这些结构实际上是由 python 代码生成的；Parser/asdl_c.py 模块会根据 AST asdl 定义生成此文件。例如，list 3.5 中展示了部分声明节点（statement node）的定义。

Listing 3.5: A cross-section of an AST statement node data structure

```c
struct _stmt {
	enum _stmt_kind kind;
	union {
        struct {
            identifier name;
            arguments_ty args;
            asdl_seq *body;
            asdl_seq *decorator_list;
            expr_ty returns;
        } FunctionDef;
        
        struct {
            identifier name;
            arguments_ty args;
            asdl_seq *body;
            asdl_seq *decorator_list;
            expr_ty returns;
        } AsyncFunctionDef;

        struct {
            identifier name;
            asdl_seq *bases;
            asdl_seq *keywords;
            asdl_seq *body;
            asdl_seq *decorator_list;
        } ClassDef;
        ...
    }v;
    int lineno;
    int col_offset
}
```

list 3.5 中的联合类型 (union) 是 C 中的类型，它可以表示联合中列出的任何类型。 Python/ast.c 模块中的 PyAST_FromNode 函数处理从给定的解析树生成 AST 的过程。生成 AST 之后，就是从 AST 中生成字节码了。

### 3.4 Building The Symbol Table

生成 AST 后，该过程的下一步是生成符号表 (symbol table) 。就像名字所表达的一样，符号表是代码块中名称的集合，并且这些名称在上下文中被使用了。建立符号表的过程涉及到分析代码块中包含的名称，并为这些名称分配正确的作用域。在讨论符号表生成的复杂性之前，可以先回顾一下 python 中的名称和绑定。

>**Names and Binding**
>
>在 python 中，对象是通过名称引用的。名称类似于 C ++ 和 Java 中的变量，但不完全相同。
>
>x = 5
>
>在上面的示例中，x 是引用对象 5 的名称。将对 5 的引用分配给 x 的过程称为绑定。绑定导致名称与当前正在执行的程序的最里面的对象相关联。绑定可能发生在许多具体的实例中，例如当提供参数有绑定变量时，会在变量分配或者函数/方法调用期间发生绑定。要注意的是，名称只是符号，符号类型和变量类型之间没有关系。名称只是对实际具有类型的对象的引用。
>
>**Code Blocks**
>
>代码块对于 python 程序至关重要，因此了解它们对于理解 python 虚拟机内部至关重要。代码块是一段程序代码，在 python 中作为一个单元执行。模块、函数和类都是代码块的例子。在 REPL 上以交互方式键入的命令，使用 -c 选项运行的脚本命令也是代码块。一个代码块具有许多与之相关的命名空间。例如，模块代码块可以访问全局命名空间，而功能代码块可以访问局部 (local) 命名空间和全局 (global) 命名空间。
>
>**Namespaces**
>
>顾名思义，命名空间是一个上下文，在该上下文中一组给定的名称会绑定到对象上。命名空间在 python 中被实现为字典映射。内置命名空间是命名空间的一个例子，它是一个包含所有内置函数的命名空间，可以通过在终端输入 __builtins__.__dict__来访问这个命名空间 (结果相当巨大) 。解释器可以访问多个命名空间，包括全局命名空间，内置命名空间和局部命名空间。这些命名空间是在不同的时间创建的，并且具有不同的生存期。例如，在调用函数时会创建一个新的局部命名空间，并在该函数退出或返回时将其丢弃。全局命名空间是在模块执行开始时创建的，并在调用解释器，并且包含所有内置名称时，内置命名空间会在模块范围内使用在该命名空间中定义的所有名称。这三个命名空间是解释器可用的主要命名空间。
>
>**Scopes**
>
>scope 是程序的一个区域，在其中一系列绑定名称 (命名空间) 是可见的，并且可以直接使用它们而无需使用任何点符号。在运行时，以下作用域可能是可用的。
>
>1. 具有局部名称的最内部作用域。
>
>2. 如果有的话，闭包函数的作用域 (适用于嵌套函数)。
>
>3. 当前模块的全局作用域。
>
>4. 包含内置命名空间的作用域。
>
>在 python 中使用名称时，解释器将按上述升序搜索范围的命名空间，如果在任何命名空间中均未找到该名称，则会引发异常。 Python 支持静态作用域，也称为词法作用域；这意味着仅检查程序文本即可推断出一组绑定名称的可见性。
>
>**注意**
>
>Python 有一个古怪的作用域规则，该规则防止在局部作用域内修改全局作用域内对对象的引用。这样的尝试将引发 UnboundLocalError 异常。为了在局部作用域内修改全局作用域内的对象，在尝试进行修改之前，必须将 global 关键字与对象名称一起使用，示例如下。
>
>Listing A3.0: Attempting to modify a global variable from a function
>
>```python
>>>> a = 1
>>>> def inc_a(): a += 2
>...
>>>> inc_a()
>Traceback (most recent call last):
>	File "<stdin>", line 1, in <module>
>	File "<stdin>", line 1, in inc_a
>UnboundLocalError: local variable 'a' referenced before assignment
>```
>
>为了在全局作用域内修改对象，如以下代码段所示，使用了 global 语句。
>
>Listing A3.1: Using the global keyword to modify a global variable from a function
>
>```python
>>>> a = 1
>>>> def inc_a():
>... 	global a
>... 	a += 1
>...
>>>> inc_a()
>>>> a
>2
>```
>
>Python 还有 nonlocal 关键字，该关键字在需要从内部作用域中修改外部非全局作用域中的绑定变量时使用。在使用嵌套函数 (也称为闭包) 时非常方便。以下代码片段有效地说明了非局部关键字的正确用法，该片段定义了一个简单的计数器对象，该计数器对象按升序计数。
>
>Listing A3.2: Creating blocks from an AST
>
>```python
>>>> def make_counter():
>... 	count = 0
>... 	def counter():
>... 		nonlocal count # capture count binding from enclosing not global scope
>... 		count += 1
>... 		return count
>... 	return counter
>...
>>>> counter_1 = make_counter()
>>>> counter_2 = make_counter()
>>>> counter_1()
>1
>>>> counter_1()
>2
>>>> counter_2()
>1
>>>> counter_2()
>2
>```

这一系列函数调用 run_mod _> PyAST_CompileObject _>  PySymtable_BuildObject 触发了构建符号表的过程。 PySymtable_BuildObject 函数的两个参数是先前生成的 AST 以及模块的文件名。建立符号表的算法分为两部分。在第一部分中，会访问 AST 的每个节点（作为参数传递给 PySymtable_BuildObject），以建立 AST 中使用的符号的集合。list 3.6 中简单描述了这个过程，当我们讨论构建符号表中所使用的数据结构时，其中使用的术语将会更加的明显。

Listing 3.6: Creating a symbol table from an AST

------

对于给定AST中的每个节点，

​		如果节点是代码块的开始：

​				1.创建新的符号表条目，并将当前符号表设置为此值。 

​				2.将新的符号表压栈到 st_stack。 

​				3.将新符号表添加到先前符号表的子列表中。 

​				4.将当前符号表更新为新符号表。

​				5.对于代码块节点中的所有节点：

​						a. 使用 “symtable_visit_XXX” 函数递归访问每个节点，其中 “XXX” 是节点类型。 

​				6.通过从栈中删除当前符号表条目来退出代码块。 

​				7.从栈中弹出下一个符号表条目，并将当前符号表条目设置为该弹出的值

​		否则：

​				递归地访问节点和子节点。

------

在 AST 上运行算法的第一阶段后，符号表条目包含模块中已使用的所有名称，但是它们没有这些名称的上下文信息。例如，解释器无法判断给定变量是全局变量，局部变量还是自由变量。调用 Parser/symtable.c 中的 symtable_analyze 函数将启动符号表生成的第二阶段。算法的此阶段将分配从第一阶段收集符号的作用域（局部、全局或者自由）。 Parser/symtable.c 中的注释很有参考价值，下面了解第二阶段对符号表构造过程。

符号表需要两遍才能确定每个名称的作用域。第一遍通过 symtable_visit_* 函数从 AST 收集原始 facts ，然后第二遍通过遍历第一遍期间创建的 PySTEntryObjects 来分析这些 facts。

在第二遍输入函数时，父级将传递对其子代可见的所有名称绑定。这些绑定用于确定非局部变量是自由变量还是隐式全局变量。在这组可见的名称中必须存在明确声明为非局部的名称，如果不存在，则会引发语法错误。进行局部分析后，它使用一组更新的绑定名称来分析其每个子块。

全局变量也有两种，隐式和显式。使用全局语句声明显式全局变量。隐式全局变量是一个自由变量，编译器在闭包函数的作用域内没有发现对其的绑定。隐式全局可以是全局的或内置的。

Python 的模块和类使用 xxx_NAME 操作码来处理这些名称，以实现稍微奇怪的语义。在这样的代码块中，名称在被分配之前，将会被视为全局的。然后分配后会将其视为局部的。

子代更新自由变量集。如果将局部变量添加到子自由变量集中，则将该变量将标记为 cell 。定义的函数对象必须为可能超出函数 frame 寿命的变量提供运行时存储。在函数返回其父级之前，会从自由变量集中删除 Cell 变量。

尽管这些讨论试图用清晰的语言解释该过程，但仍存在一些令人困惑的点，例如父级传递了对其子级可见的所有绑定名称的集合，父级和子级分别指的是哪些？为了理解这种术语，我们必须查看在创建符号表的过程中使用的数据结构。

**Symbol table data structures**

符号表生成的两个主要数据结构是：

1. 符号表数据结构。 
2. 符号表条目数据结构。

符号表数据结构如 list 3.7 所示。可以将其视为一张表，由多个条目组成，这些条目保存了给定模块不同代码块中使用的名称的信息。

Listing 3.7: The symtable data structure

```c
struct symtable {
	PyObject *st_filename; 				/* name of file being compiled */
	struct _symtable_entry *st_cur; 	/* current symbol table entry */
	struct _symtable_entry *st_top; 	/* symbol table entry for module */
    PyObject *st_blocks; 				/* dict: map AST node addresses
    					 				to symbol table entries */
    PyObject *st_stack; 				/*list: stack of namespace info */
    PyObject *st_global; 				/*borrowed ref to st_top->ste_symbols*/
    int st_nblocks; 					/* number of blocks used. kept for
    									consistency with the corresponding
    									compiler structure */
    PyObject *st_private; 				/* name of current class or NULL */
    PyFutureFeatures *st_future; 		/* module's future features that
    									affect the symbol table */
    int recursion_depth; 				/* current recursion depth */
    int recursion_limit; 				/* recursion limit */
};
```

python 模块可以包含多个代码块，例如多个函数定义，并且 st_blocks 字段是所有存在的代码块到符号表条目的映射。st_top 是正在编译的模块的符号表条目（模块也是代码块），因此它会包含在模块的全局命名空间中定义的名称。st_cur 代表当前正在处理的代码块的符号表条目。模块代码块中的每个代码块都有自己的符号表条目，其中包含该代码块中定义的符号。

Figure 3.1: A Symbol table and symbol table entries.

![3.1](\./images/image-20191102204156.png)

再次查看 Include/symtable.h 中的 _symtable_entry 数据结构对了解此数据结构的作用有很大的帮助。list 3.8 中展示了此数据结构。

Listing 3.8: The _symtable_entry data structure

```c
typedef struct _symtable_entry {
    PyObject_HEAD
    PyObject *ste_id; 					/* int: key in ste_table->st_blocks */
    PyObject *ste_symbols; 				/* dict: variable names to flags */
    PyObject *ste_name; 				/* string: name of current block */
    PyObject *ste_varnames; 			/* list of function parameters */
    PyObject *ste_children; 			/* list of child blocks */
    PyObject *ste_directives; 			/* locations of global and nonlocal statements */
	
    _Py_block_ty ste_type; 				/* module, class, or function */
	int ste_nested; 					/* true if block is nested */
	unsigned ste_free : 1; 				/*true if block has free variables*/
	unsigned ste_child_free : 1; 		/* true if a child block has free
										vars including free refs to globals*/
	unsigned ste_generator : 1;			/* true if namespace is a generator */
	unsigned ste_varargs : 1; 			/* true if block has varargs */
	unsigned ste_varkeywords : 1; 		/* true if block has varkeywords */
	unsigned ste_returns_value : 1; 	/* true if namespace uses return with
										an argument */
	unsigned ste_needs_class_closure : 1; /* for class scopes, true if a
											closure over __class__
											should be created */
    int ste_lineno; 					/* first line of block */
	int ste_col_offset; 				/* offset of first line of block */
	int ste_opt_lineno; 				/* lineno of last exec or import * */
	int ste_opt_col_offset; 			/* offset of last exec or import * */
	int ste_tmpname; 					/* counter for listcomp temp vars */
	struct symtable *ste_table;
} PySTEntryObject;
```

源代码中的注释说明了每个字段的作用。 ste_symbols 字段是一个映射，其中包含在代码块分析期间遇到的符号/名称；符号映射到的标志是数值，它提供有关使用符号/名称的上下文信息。例如，一个符号可以是函数参数或全局语句定义。list 3.9 中显示了部分在 Include/symtable.h 模块中定义的标志。

```c
/* Flags for def-use information */
#define DEF_GLOBAL 1 		/* global stmt */
#define DEF_LOCAL 2			/* assignment in code block */
#define DEF_PARAM 2<<1 		/* formal parameter */
#define DEF_NONLOCAL 2<<2 	/* nonlocal stmt */
#define DEF_FREE 2<<4 		/* name used but not defined in
							nested block */
```

回到关于符号表的讨论，假设正在编译包含 list 3.10 中所示代码的模块。构建符号表后，将有三个符号表条目。

Listing 3.10: A simple python function

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter
```

第一个条目是闭包模块，它在局部作用域内定义 make_counter。下一个符号表条目将是功能 make_counter 的条目，并将计数和计数器名称标记为局部。最终的符号表条目是内部 counter 函数。这会将 count 变量标记为 free 。需要注意的一件事是，尽管 make_counter 在模块的块符号表条目中定义为局部，但由于 *st_global 指向 *st_top 符号，因此在模块代码块中将其视为全局定义。

### 3.5 From AST To Code Objects

生成符号表后，编译器下一步是结合符号表中包含的信息，从 AST 中生成代码对象。处理此步骤的函数在Python/compile.c 模块中实现。生成代码对象的过程也是包含了多步。第一步，将 AST 转换为 python 字节码指令的基本块。此算法类似于生成符号表中使用的算法，即称为 compile_visit_xx 的函数，其中 xx 是节点类型，用于在访问过程中递归访问每个节点类型，并发出 python 字节码指令的基本块。它们之间的基本块和路径隐式表示一个控制流程图。该图显示了在程序执行期间可以采用的代码路径。在第二步中，使用后深度优先搜索遍历对生成的控制流程图进行展平。将图展平后，然后计算跳转偏移并将其用作字节码跳转指令的指令参数。代码对象是从这组指令中发出的。为了更好地了解此过程，请参考 list 3.11 中的 fizzbuzz 函数。

Listing 3.11: A simple python function

```python
def fizzbuzz(n):
    if n % 3 == 0 and n % 5 == 0:
        return 'FizzBuzz'
    elif n % 3 == 0:
        return 'Fizz'
    elif n % 5 == 0:
        return 'Buzz'
    else:
        return str(n)
```

上面函数的 AST 如下图所示。

Figure 3.2: A very simple AST for listing 3.2

![3.2](./images/image-20191103115002.png)

图 3.2 中的 AST 编译为 CFG (the control flow graph) 时，其图形类似于图 3.3 所示。图中省略了空白块。基本块具有单个入口，但可以具有多个出口。下面将会更加详细地描述这些块。

Figure 3.3: Control flow graph for the fizzbuzz function from listing 3.11. The straight line represent normal straight line execution of code while the curved lines represent jumps.

![3.3](./images/image-20191103115218.png)

在下面的描述中，仅包括实际的指令。为了使我们能够专注于当前的主题，一些需要参数的指令并未包括在内。

1. Block 1此块包含映射到图 3.2 中 AST 的 BoolOp 节点的指令。该块中的指令使用以下十一组指令来实现操作 n％3 == 0 和 n％5 == 0 。

   ```python
   LOAD_FAST
   LOAD_CONST
   BINARY_MODULO
   LOAD_CONST
   COMPARE_OP
   JUMP_IF_FALSE_OR_POP
   LOAD_FAST
   LOAD_CONST
   BINARY_MODULO
   LOAD_CONST
   COMPARE_OP
   ```

   令人惊讶的是，其余的 if 节点（确定是否应执行该子句的实际测试）未包含在此 block 中。在讨论第二个代码块时，我们就会更加清晰的明白这样的原因。如图 3.3 所示，有两种方法可以退出该 block ：直接执行所有操作码或者在执行 JUMP_IF_FALSE_OR_POP 时跳转到代码 block 2。

2. Block 2 此 block 映射到第一个 if 节点，其中封装了 if 测试和后续的子句。第二个 block 中包含以下四个指令。

   ```python
   POP_JUMP_IF_FALSE
   LOAD_CONST
   RETURN_VALUE
   JUMP_FORWARD
   ```

   从后面的章节中可以看出，当解释器为 if 语句执行字节码指令时，它从 value stack 中读取一个对象，并根据该对象的真值，执行下一个字节码指令或跳转到指令集的其他部分，然后从那里继续执行。 POP_JUMP_IF_FALSE 是处理该过程的指令。此操作码有一个参数，该参数指定此跳转的目的地。

   有人可能想为什么 BoolOp 节点的指令以及 if 语句在不同的块中。为了理解这一点，请记住 python 使用短路求值进行布尔运算，因此在这种情况下，如果 n ％ 3 == 0 的计算结果为 false，则不会计算 n ％ 5 == 0。第一次比较后，查看第一个 block 中的指令，您会注意到 JUMP_IF_FALSE_OR_POP 这个指令。该指令是 jump 指令的变体，因此需要一个目标。

   JUMP_IF_FALSE_OR_POP 需要一个目标，当布尔表达式中的第一个表达式由于短路操作而求值为 false 时，将在该目标处继续执行指令，在这种情况下，目标是 if 语句中的 POP_JUMP_IF_FALSE 指令。为了使跳转成为可能，我们需要一个不同 block 并且有 if 语句指令的目标去跳转，然后可以计算出进行跳转的偏移量。如果计算了布尔表达式的所有部分，则将在执行 BoolOp 块中的所有指令之后，以 if 块中的指令继续正常执行。

3. Block 3 映射到第三个 block 的第一个 orElse AST 节点，它包含以下 9 条指令。

   ```python
   LOAD_FAST
   LOAD_CONST
   BINARY_MODULO
   LOAD_CONST
   COMPARE_OP
   POP_JUMP_IF_FALSE
   LOAD_CONST
   RETURN_VALUE
   JUMP_FORWARD
   ```

   可以看到 elif 语句和 n ％ 3 == 0 以及语句主体都在同一个 block 中。进入此 block 的唯一入口就是跳入该 block ，并且如果 if 结果为 false ，则会通过返回指令或跳转来退出该节点。

4. Block 4 是就指令而言的Block 3 的镜像，但指令的参数不同。

5. Block 5 映射到最终的 orElse AST 节点上，并包含以下 4 条指令。

   ```python
   LOAD_GLOBAL
   LOAD_FAST
   CALL_FUNCTION
RETURN_VALUE
   ```
   
   LOAD_GLOBAL 将 str 函数作为参数并将其加载到值堆栈中。 LOAD_FAST 将参数 n 加载到堆栈上，而RETURN_VALUE 返回执行 CALL_FUNCTION 指令后在堆栈上的值，即 str(n)。

与上一节一样，我们将会研究用于构建基本块的数据结构，以便于更好地掌握此过程。

**The compiler data structure**

图 3.4 显示了在生成控制流程图的基本块过程中使用的主要数据结构之间的关系。

![3.4](./images/image-20191103190235.png)

​				Figure 3.4: The four major data structures used in generating a code object.

最顶层是编译器数据结构，它捕获模块全局编译的过程。list 3.12 中定义了此数据结构。

```C
struct compiler {
    PyObject *c_filename;
    struct symtable *c_st;
    PyFutureFeatures *c_future; /* pointer to module's __future__ */
    PyCompilerFlags *c_flags;

    int c_optimize; 			/* optimization level */
    int c_interactive; 			/* true if in interactive mode */
    int c_nestlevel;

    struct compiler_unit *u; 	/* compiler state for current block */
    PyObject *c_stack; 			/* Python list holding compiler_unit ptrs */
    PyArena *c_arena; 			/* pointer to memory allocation arena */
};
```

以下是我们感兴趣的字段。 
1.  *c_st: 对上一部分中生成的符号表进行引用。 
2.  *u: 对编译器单元数据结构进行引用。这封装了使用代码块所需的信息。该字段指向正在操作的当前代码块的编译器单元。 
3.  \*c_stack: 对 compiler_unit 数据结构堆栈的引用。当一个代码块由多个代码块组成时，此字段将在遇到新块时，会对 compile_unit 数据结构的保存和恢复进行处理。输入新的代码块后，创建新的作用域，然后 editor_enter_scope() 将当前的 compuger_unit  \*u 推入堆栈 \*c_stack 中，创建一个新的 compile_unit 对象，并且当遇到新的模块时将其设置为当前状态。当退出该块时，*c_stack 会从堆栈中弹出，以恢复状态。

对于每个要编译的模块，都会初始化一个编译器数据结构；当遍历为模块生成的 AST 时，会为 AST 中遇到的每个代码块生成一个 editor_unit 数据结构。

**The compiler_unit data structure**

如下面 list 3.13 所示，compiler_unit 数据结构展示了生成代码块所需的字节码指令所需的信息。当我们查看代码对象时，将会遇到很多在 compiler_unit 中定义的字段。

Listing 3.13: The compiler_unit data strcuture

```c
struct compiler_unit {
    PySTEntryObject *u_ste;

    PyObject *u_name;
    PyObject *u_qualname; /* dot-separated qualified name (lazy) */
    int u_scope_type;

    /* The following fields are dicts that map objects to
    the index of them in co_XXX. The index is used as
    the argument for opcodes that refer to those collections.
    */
    PyObject *u_consts; 	/* all constants */
    PyObject *u_names; 		/* all names */
    PyObject *u_varnames; 	/* local variables */
    PyObject *u_cellvars; 	/* cell variables */
    PyObject *u_freevars; 	/* free variables */
    
    PyObject *u_private; 	/* for private name mangling */
    
    Py_ssize_t u_argcount; 	/* number of arguments for block */
    Py_ssize_t u_kwonlyargcount; 	/* number of keyword only arguments for block */
    
    /* Pointer to the most recently allocated block. By following b_list
    members, you can reach all early allocated blocks. */
    basicblock *u_blocks;
    basicblock *u_curblock; /* pointer to current block */
    
    int u_nfblocks;
    struct fblockinfo u_fblock[CO_MAXBLOCKS];
    
    int u_firstlineno; 		/* the first lineno of the block */
    int u_lineno; 			/* the lineno for the current stmt */
    int u_col_offset; 		/* the offset of the current stmt */
    int u_lineno_set; 		/* boolean to indicate whether instr
    						has been generated with current lineno */
};
```

u_blocks 和 u_curblock 字段的引用构成正在编译的代码块的基本块。 *u_ste 字段是对正在编译的代码块的符号表条目的引用。其余字段都可以从名称中得到意义。在编译过程中将遍历组成代码块的不同节点，并且根据给定节点类型是否开始基本块，将创建包含节点指令的基本块，或者将节点的指令添加到现有基本块中。块可以开始新的基本块的节点类型包括但不限于以下类型。 

1. 功能节点。 
2.  跳跃到目标。 
3. 异常处理程序。 
4. 布尔操作等等。

**The basic_block and instruction data structures**

基本块数据结构在生成控制流程图的过程中是一个相当有趣的数据结构。基本块是具有一个入口但具有多个出口的指令序列。list 3.14 中显示了 python 虚拟机中使用的 basic_block 数据结构的定义。

Listing 3.14: The basicblock_ data strcuture

```c
typedef struct basicblock_ {
    /* Each basicblock in a compilation unit is linked via b_list in the
    reverse order that the block are allocated. b_list points to the next
    block, not to be confused with b_next, which is next by control flow. */
    struct basicblock_ *b_list;
    /* number of instructions used */
    int b_iused;
    /* length of instruction array (b_instr) */
    int b_ialloc;
    /* pointer to an array of instructions, initially NULL */
    struct instr *b_instr;    struct instr *b_instr;
    /* If b_next is non-NULL, it is a pointer to the next
    block reached by normal control flow. */
    struct basicblock_ *b_next;
    /* b_seen is used to perform a DFS of basicblocks. */
    unsigned b_seen : 1;
    /* b_return is true if a RETURN_VALUE opcode is inserted. */
    unsigned b_return : 1;
    /* depth of stack upon entry of block, computed by stackdepth() */
    int b_startdepth;
    /* instruction offset for block, computed by assemble_jump_offsets() */
    int b_offset;
} basicblock;
```

如前所述，CFG 基本上由基本块和这些基本块之间的连接组成。 *b_instr 字段引用指令数据结构的数组，并且这些数据结构中的每一个都保存一个字节码指令。这些字节码可以在 Include/opcode.h 头文件中找到。指令数据结构如 list 3.15 所示。

Listing 3.15: The instr data strcuture

```c
struct instr {
    unsigned i_jabs : 1;
    unsigned i_jrel : 1;
    unsigned char i_opcode;
    int i_oparg;
    struct basicblock_ *i_target; /* target block (if jump instruction) */
    int i_lineno;
};
```

看一下 fizzbuzz 函数的 CFG，我们可以看到实际上有两种方法可以从 block 1 到 block 2 。第一种是通过正常执行： 执行完 block 1 中的所有指令之后，在 block 2 中继续执行。另一种方法是通过仅存在于第一个比较操作之后的跳转指令进行。这个跳转的目标是一个基本 block，但实际执行的代码对象对基本 block 一无所知，该代码块仅具有字节码流 (stream of bytecodes)，而我们只能通过偏移量对这种 stream 进行索引。我们不得不使用块创建的隐式图作为跳转目标，并将这些 block with offset 替换进指令数组中。这就是基本 block 的组装过程。

**Assembling basic blocks**

生成 CFG 后，基本块现在包含表示 AST 的字节码指令，但这些块不是线性排序的，对于跳转语句，指令仍将基本块作为跳转目标，而不是相对于指令的相对或绝对偏移。assemble 功能处理 CFG 的线性化和从 CFG 创建代码对象。

首先，assemble 函数向没有 RETURN 语句结束的任意 block 添加 return None 语句的指令，这就是为什么可以定义没有 RETURN 语句的方法。接下来是隐式 CFG 的后序深度优先遍历，为了使块平坦化，后序遍历在访问节点本身之前需要先访问子节点。

![3.5](./images/image-20191103191905.png)

在图的后序深度优先遍历中，我们递归地访问图的左子节点，然后依次访问图的右子节点和节点本身。在图3.5 的图形中，当使用后序遍历对图形进行展平时，节点的顺序为 H _> D _> I _> J _> E _> B _> K _> L _> F _> G _> C _> A 。这与先序遍历 A _> B _> D _> H _> E _> I _> J _> C _> F _> K _> L _> G 或者 中序遍历 H _> D _> B _> I _> E _> J _> A _> K _> L _> F _> C _> G 相反。

list 3.3 中给出的 fizzbuzz 函数的 CFG 是一个相对简单的图形，fizzbuzz 的后顺序遍历的结果是：block 5 _> block 4 _> block 3 _> block 2 _> block 1。如果已经线性化（即展平），则可以通过在展平图上调用 assemble_jump_offsets 函数来计算指令跳转的偏移量。

jump 的组装分为两个阶段。在第一阶段，如 list 3.16 中的代码片段所示，计算每个指令到指令数组的偏移量。这是一个简单的循环，从展平数组的末尾开始，从 0 开始建立偏移量。

Listing 3.16: Calculating bytecode offsets

```c
...
totsize = 0;
for (i = a->a_nblocks - 1; i >= 0; i--) {
    b = a->a_postorder[i];
    bsize = blocksize(b);
    b->b_offset = totsize;
    totsize += bsize;
}
...
```

在组装 jump 偏移量的第二阶段，然后如 list 3.17 所示计算跳转指令的跳转目标。这涉及计算相对跳转的相对跳转，并用指令偏移量替换绝对跳转的目标。

Listing 3.17: Assembling jump offsets

```c
...
for (b = c->u->u_blocks; b != NULL; b = b->b_list) {
    bsize = b->b_offset;
    for (i = 0; i < b->b_iused; i++) {
        struct instr *instr = &b->b_instr[i];
        int isize = instrsize(instr->i_oparg);
        /* Relative jumps are computed relative to
        the instruction pointer after fetching
        the jump instruction.
        */
        bsize += isize;
        if (instr->i_jabs || instr->i_jrel) {
            instr->i_oparg = instr->i_target->b_offset;
            if (instr->i_jrel) {
                instr->i_oparg -= bsize;
            }
            instr->i_oparg *= sizeof(_Py_CODEUNIT);
            if (instrsize(instr->i_oparg) != isize) {
                extended_arg_recompile = 1;
            }
        }
    }
}
...
```

计算出跳转偏移后，展平图中包含的指令以相反的后序遍历开始发出。倒置后序是 CFG 的拓扑排序。这意味着对于从顶点 u 到顶点 v 的每个边，在排序顺序中 u 都排在 v 之前。原因很明显，我们希望一个跳转到另一个节点的节点始终位于该跳转目标之前。完成字节码的发送后，可以使用发出的字节码和符号表中包含的信息为每个代码块组合代码对象。生成的代码对象返回到调用函数，标志着编译过程的结束。

