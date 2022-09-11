# `#define`——一切的起源
脱离了`#define`的C代码是丑陋的。`#define`对于成熟的C程序员来说，如同鱼之水，雁之风。

`#define`以及`#`、`##`被称为替换文本宏（Replacing text macros），将在程序编译过程中的预处理阶段[展开](#展开)。

## 基本的替换
```c
#define identifier replacement-list
```
- identifier：标识符
- replacement-list：替换列表（可选）

标识符的概念我想每一位程序员都是了解的，它是标识某种事物的一串符号组合，用于区分同类型的不同物体，这就像每个人都有一个名字一样。

<div id="展开"></div>

替换列表是在预处理阶段由标识符替换成的东西，我们称这个替换过程为**展开**。为什么称其为展开呢？因为在很多时候，我们期望使用替换文本宏代替大量不需要的重复性输入，譬如一段字符串`"Hello,world!"`。

这时，我们就可以使用文本替换宏：
```c
#include <stdio.h>
#include <stdlib.h>
#define S_HW "Hello,World!"

int 
main( int argc, char * argv[])
{
    puts(S_HW);
    exit(0);
}
```
编译并执行上面的程序，就可以得到输出`Hello,World!`，这是因为在预编译阶段，`S_HW`被替换为了`"Hello,World!"`，对于这样的替换我们可以通过`gcc`参数`-E`来验证。在编译文件时，使用参数`-E -o a.i`生成名为`a.i`的预处理文件，找到`main`函数，将会发现`S_HW`被替换为了如下形式：
```c
int 
main( int argc, char * argv[])
{
    puts("Hello,World!");
    exit(0);
}
```
像这样使用固定标识符对应固定*替换列表*的文本替换宏，我们将其称为*类对象宏（Objection-like Macros）*。

## 当使用函数变成习惯
文本替换宏有一种升级的用法，称之为*类函数宏（Function-like Macros）*。此类宏会将每一个出现它的地方替换为替换列表。
其基本语法如下：
```c
#define identifier( parameters )  repalcement-list
```
- identifier：标识符
- parameters：参数
- replacement-list：替换列表

我想参数的概念不需要我作过多的介绍了，值得注意的是，在类函数宏的写法中，*替换列表*不再是一个可选项，而变成了一个必要项。这就意味着：在进行替换时，不能再对*替换列表*进行省略了。

我想，在这个时候还是写出一个示例代码比较清晰：
```c
#include <stdio.h>
#include <stdlib.h>
#define MAX(a,b)  a>b?a:b

int
main ( int argc, char * argv[])
{
    printf("%d",MAX(2,1));
    exit(0);
}
```
接下来，让我们对它的展开进行预测，根据*类对象宏*展开的经验，我们预测他将会将`MAX(2,1)`替换为`2>1?2:1`，事实也正是如此，在对其进行预处理后，我们会观察到`main`函数内部如下：
```c
int
main ( int argc, char * argv[])
{
    printf("%d",2>1?2:1);
    exit(0);
}
```

## 仅仅如此吗？
从C89/C90标准中，定义了可变参数的概念，并添加了库`<stdarg.h>`，那么类函数宏为和不能也添加这个特性呢？于是在C99中，类函数宏添加上了可变参数语法：
```c
#define identifier( parameters , ... )  repalcement-list
#define identifier( ... )  repalcement-list
```
需要注意的是，这两种语法是从**C99**开始，因此在使用较旧的编译器时可能无法通过编译。

### 可变是一种包容性的方式
可变是一种兼容性的方式，使同一个函数可以接受任意`>=1`个参数。类函数宏也继承了这一思想，同时也诞生了特定标识符`__VA_ARGS__`，标识符`__VA_ARGS__`用以表示在类函数宏声明中以`...`表示的所有参数，参数数量至少大于一个。

语言总是让人觉得描述的不够清楚，所以还是看看下面的代码吧：
```c
#include <stdio.h>
#include <stdlib.h>
#define PRINT(str,...) \
        printf(#str, __VA_ARGS__)

int
main( int argc, char * argv[])
{
    PRINT(2*3=%d\n,2*3);
    PRINT(2+3=%d\n 2-3=%d\n,2+3,2-3);
    exit(0);
}
```
我们先看看它预处理的结果，再进行解释：
```c
int
main( int argc, char * argv[])
{
    printf("2*3=%d\n",2*3);
    printf("2+3=%d\n 2-3=%d\n",2+3,2-3);
    exit(0);
}
```

或许读者会疑惑，`printf`的第一个参数应该是一个字符串`const char * fmt`，那么为什么这里并没有加上双引号呢？让我们来引出在本章一开始说到的`#`与`##`。

### 函数工厂——减少代码量的绝佳办法
当我们期望有一串字符可以被处理为字符串时，就可以使用`#`，它会在预处理时主动为一串字符加上双引号。正如上面的例子中那样，在宏声明中为类函数宏的参数`str`加上了`#`，这会使宏展开时，为替换的文本加上双引号。同时，如果这串字符串本身含有双引号，则将其含有的双引号变为`\"`。

当我们需要对某一种功能书写重复的函数或者代码块时，机械性的劳动无疑使人觉得乏味，并且极有可能出现失误，因此我们需要一种方法来帮助我们减少在代码中的重复性劳动，程序员们灵机一动，不如将代码生成交给预处理阶段，这种通过宏生成函数的方式成为函数工厂（Functions Factory）。

假设我们要处理几组数据，他们分别是几组不同的数据类型，那么我们可以考虑使用一个函数工厂来避免重复性的书写相同的代码：
```c
#include <stdio.h>
#include <stdlib.h>
#define ADD_FUNC_FACT(type)                 \
        type                                \
        add_func_##type(type a,type b)      \
        {                                   \
            return a+b;                     \
        } 
#define CALL_FUNC(type,a,b)                 \
        add_func_##type(a,b)

ADD_FUNC_FACT(int)
ADD_FUNC_FACT(double)
ADD_FUNC_FACT(long)

int 
main(int argc, char * argv[])
{
    printf("2+2=%d\n",CALL_FUNC(int,2,2));
    printf("2.3+2.9=%.1f\n",CALL_FUNC(double,2.3,2.9));
    exit(0);
}
```
`##`会帮助我们辨识类函数宏的参数应该放在哪里，这使得我们可以创造出来功能类似的函数，同时可以避免他们函数名出现重复。在上面的代码中，预处理阶段，对于三个`ADD_FUNC_FACT`宏的使用，分别生成了三个功能类似的函数`add_func_int`、`add_func_double`、`add_func_long`。在`CALL_FUNC`宏中，我也使用了类似的方法，来调用不同的函数，这是一种简单而又高效的办法，可以使你在创建大型项目时有效的减少代码的书写量。

当然，这种方法在现在已经有了替代方案，在C11中增添的泛型宏`_Generic()`，就有类似的功能，且更加强大。它允许通过函数的某一个参数的类型来确定所调用的函数，具体的内容可以通过[这里](https://zh.cppreference.com/w/c/language/generic)查看。

### 宏也要有生命周期
有时候我们需要将一段宏嵌入到某个函数中，但之后将不再需要这个函数，我们就需要在使用完后对宏常量进行处理，以避免在未来对代码进行扩展时，出现宏常量命名冲突的情况。所以我们需要`undef`来协助我们清理掉不再需要的宏常量。

我们先给出一个`undef`的实例作以介绍：
```c
#include <stdio.h>
#include <stdlib.h>

int
main( int argc, char * argv[])
{
    int val = 2;
#define CONST_INT_A 10
    printf("CONST_INT_A/2=%d\n",CONST_INT_A/2);
#undef CONST_INT_A
#define CONST_INT_A 20
    printf("CONST_INT_A/2=%d\n",CONST_INT_A/2);
#undef CONST_INT_A
    exit(0);
}
```
显而易见的，在第一次`#undef CONST_INT_A`后，编译器将删除`CONST_INT_A`与`10`之间的对应，但其后紧接着，第二次定义了`COSNT_INT_A`为`20`，因此第二次展开`CONST_INT_A/2`时不再是`10/2`，而是`20/2`，因此两次输出分别为`5`和`10`。

## 课外小技巧——使用文本替换宏替代结构体声明
这是一种可能存在争议性的写法，但是不得不说，这种写法是比较巧妙的。
```c
#include <stdlib.h>
#include <stdio.h>
#define virtual_A \
	struct {\
		int _1;\
		int _2;\
	}

virtual_A * A_add( virtual_A* a, virtual_A* b){
	virtual_A* res = (virtual_A*)malloc(sizeof(virtual_A));
	res->_1 = a->_1 + b->_1;
	res->_2 = a->_2 + b->_2;
}

static struct {	
	virtual_A *(*add)(virtual_A* , virtual_A*); 
} A_method = {.add=A_add};

int 
main( int argc, char * argv[])
{
	virtual_A a = {1,2};
	virtual_A b = {2,3};
	virtual_A * c = A_method.add(&a,&b);
	printf("%d\t\t%d\n",c->_1,c->_2);
	exit(0);
}
```
我们使用了替换宏而不直接定义结构体类型，这使得我们在进行继承时，可以使用类似与oop的方法：
```c
struct b{
    virtual_A;
    int _3;
};
```
这会直接在内部展开为：
```c
struct b{
	struct {
		int _1;
		int _2;
	};
    int _3;
};
```
显然，作为匿名成员，显然所有类`b`的实现都可以使用`b._1`来直接访问其所继承的类`virtual_A`。

# `#if`——使用宏对代码进行分支
计算机的本质就是一堆布尔逻辑，因此宏也不例外，现在我们来看看如何在预编译中使用宏来提升程序执行速度。

## 基本的逻辑处理
```c
#if expression
// code
#elif expression
// code
#else
// code
#endif
```
如上所示，其中的`expression`应当是一个在预编译阶段就可以确定布尔值的表达式，例如：`0`、`2>1`等等，其中最特殊的应该是`defined(identifier)`表达，如果`identifier`在之前被定义过且没有被取消定义，则表达式`defined(identifier)`结果为`true`。

例如下面的程序：
```c
#include <stdio.h>
#include <stdlib.h>

#define A

#if defined(A)
    #define B 0
#endif

#undef A

#if defined(A)
    #define C 2
#else
    #define C 3
#endif

int
main( int argc, char * argv[])
{
    int b = B;
    int c = C;
    printf("%d,%d\n", b,c);
    exit(0);
}
```
编译后输出可以得到`0,3`。这说明，在预编译阶段`B`被定义为`0`，`C`被定义为`3`。

## 在预编译时对函数实现进行分支
我们可以通过一个具体的例子来观察我们是如何在预编译阶段对函数实现进行分支的：
```c
/**
 *@file: a.h
 *@author:Do1tY5f
 */
#define __USING_INT32 1
```
```c
/**
 *@file: a.c
 *@author:Do1tY5f
 */
#include <stdio.h>
#include <stdlib.h>
#include <limits.h>
#include "a.h"
int 
main(int argc, char* argv[])
{
#if __USING_INT32
    int a = INT_MAX;
#else
    short a = SHRT_MAX;
#endif
    printf("The largest number supported is: %d\n",a);
    exit(0);
}
```
此时我们先对其编译运行一次，可以得到输出：
```shell
The largest number supported is: 2147483647
```
再将`#define __USING_INT32 1`修改为`#define __USING_INT32 0`，再次进行编译运行，可以得到：
```shell
The largest number supported is: 32767
```
让我们来考虑这样一种情况，当我们要完成一个可能运行在不同设备上的任务调度程序时，设备可能是单核的，也可能是多核的，这意味着，任务可能并行也可能并发，这将会是两种不同的调度策略，但我们规定了只提供一个接口`void task_schedualer(pid * )`，那么我们就可以使用如下的方案来完成这个程序：
```c
void
task_schedualer(pid * process_id)
{
#if USING_MULTI_KERNEL
    //多核调度策略的实现
#else
    //单核调度策略的实现
#endif
}
```
显然，只要我们在头文件中定义了`USING_MULTI_KERNEL`为`1`则将在预编译阶段去掉单核调度策略的实现，如定义`USIGN_NULTI_KERNEL`为`0`，则将去掉多核调度策略的实现。

有时，我们希望某些函数不会参与编译，但是又不希望将其删除或注释掉，我们也可以使用`#if`使其在预编译阶段被处理掉，只需要在其首尾分别加上`#if 0`和`#endif`即可。

## `#if defined()`的替代——`#ifdef`与`#ifndef`
在大多时刻，我们希望用更少的代码量表达更多的意思，于是`#ifdef`出现了，其所表达的意思和`#if defined()`完全一致，因此我们可以直接作以替换。

有这样一种情况：当我们在编写一个头文件时，为了避免重复引用，会特别定义一个宏变量来防止这种情况发生。这样我们只需要确定这个宏变量是否被定义，即可防止重复引用的出现。根据上面对`#if defined()`的描述，我们可以如下方法：
```c
#if defined(__SPECIAL_H__)
#else
#define __SPECIAL_H__
// header file content
#endif
```
然而，实际上，我们知道`#if`后面的部分被定义为逻辑表达式，因此可以加上`!`表示`bool`取反。因此我们可以将上面的实现更简单的写成下面的表达：
```c
#if !defined(__SPECIAL_H__)
#define __SPECIAL_H__
// header file content
#endif
```
然而，实际上，除了`#ifdef`外，C标准还提供了`#ifndef`来表示`#if !defined()`的情况，于是我们又可以将上面的代码简写成：
```c
#ifndef __SPECIAL_H__
#define __SPECIAL_H__
// header file content
#endif
```
    小贴士：
    实际上，在C23的标准中新加入了`#elifdef`以及`#elifndef`两个新的*条件包括* 预处理，从其命名就可以看出实际上是`#elif defined()`和`#elif !defined()`的替代，因此不再作过多的介绍。

## 工程实践——`#if`的实际使用

实际上大部分情况下，我们都会使用`#ifdef`和`#ifndef`，但会有一些特殊情形，使用`#if`更加合适。

在完成项目时，我们希望每次debug后可以保留原来的函数方法，而注释本身应该是对代码块的解释，不应该将代码块也注释掉，这会影响代码的可阅读性，因此会采用`#if 0 ... #endif`的写法来保证编译器在预处理阶段将这段代码去掉。例如：

```c
#include <stdio.h>
#include <stdlib.h>

#define PRINT_STR_ONCE      1
#define PRINT_STR_TWICE     2
#define CALL_BACK_MAX       2

typedef struct{
    int cmd;
    void(*call_back_func)(const char *);
}CALLBACK_STRUCT;

void 
print_1( const char * str)
{
    printf("%s\n",str);
}
void 
print_2(const char * str)
{
#if 0
    for (int i = 2; i > 0; i--) {
        printf("%s\n",str);
    }
#endif
    printf("%s\n",str);
    printf("%s\n",str);
}

static CALLBACK_STRUCT call_bk_set[2] = {
    {.cmd = PRINT_STR_ONCE, .call_back_func = print_1},
    {.cmd = PRINT_STR_TWICE, .call_back_func = print_2},
};

void 
call_back(int cmd, const char * str)
{
    for (int i = 0; i < CALL_BACK_MAX; i++) {
        if(cmd==call_bk_set[i].cmd){
            call_bk_set[i].call_back_func(str);
        }
    }
}

int
main(int argc, char * argv[])
{
    call_back(atoi(argv[1]), argv[2]);
    exit(0);
}
```