# #define——一切的起源
脱离了`define`的C代码是丑陋的。`define`对于成熟的C程序员来说，如同鱼之水，雁之风。

`define`以及`#`、`##`被称为替换文本宏（Replacing text macros），将在程序编译过程中的预处理阶段[展开](#展开)。

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

# #if——使用宏对代码进行分支
