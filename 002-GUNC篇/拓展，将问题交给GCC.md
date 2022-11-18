# GNUC 对于 C 语言的扩展
## 语句与声明表达式
使用圆括号（`(...)`）包裹的复合语句会被GCC识别为一个表达式，也就是说，你可以在其中使用循环，switch以及局部变量。
在复合语句中，由大括号（`{...}`）包裹一系列语句，因此，我们可以使用这样一种结构`({...})`，譬如：
```c
({
    int a = foo(x);
    int b;
    if(a > 0) b = a;
    else b = -a;
    b;
})
```
当然，这段代码对于编译器来说是有效的，并且最后会返回`b`的值，同时，我们知道这段代码中`a`和`b`的声明周期仅从声明处到符合语句结束。

我们将上面这段代码补全：
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
int 
foo()
{
    struct timespec st = {0};
    timespec_get(&st, TIME_UTC);
    srand((unsigned int)st.tv_nsec);
    return rand()%10;
}

int
main(int argc, char * args[])
{
    for(int i = 0; i< 5; i++){
        int x = ({
            int a = foo(x);
            int b;
            if(a > 0) b = a;
            else b = -a;
            b;
        });
        printf("x = %d\n",x);
    }
    exit(0);
}
```
编译上面的代码，我们将得到五个`10`以内的随机数。

对于一个复合语句来说，其最后一项应该是一个表达式，当然，如果是其他类型的语句也是可以的，但是这会导致整个结构的类型为`void`，也就不再有值。

这一特性适用于使用宏定义时保证代码的安全性，譬如我们在标准C中通常这样定义一个判定较大值的`max(a,b)`宏函数：
```c
#define max(a,b)    ((a) > (b) ? (a) : (b))
```
但是这种定义方式在一些比较复杂的入参替换时（譬如：`max(x+y%2/10, z%3*5)`），会计算两次`a`或`b`，这显然是比较糟糕的。因此，当我们知道入参的数值类型时，就可以用下面这种方法来避免上面出现的情况：
```c
#define maxint(a,b) \
    ({\
        int _a = (a), _b = (b);\
        _a > _b ? _a : _b;\
    })
```
很显然，这样就可以避免入参`a`和`b`的多次计算。
    
但是显然，在这种声明中应该考虑变量遮蔽的问题，譬如我们使用如下的调用方式：
```c
int _a = 1, _b = 2;
int c = maxint(_a, _b);
```
这段代码在经过预处理后，会被替换成如下代码：
```c
int _a = 1, _b = 2;
int c = 
    ({
        int _a = (_a), _b = (_b);
        _a > _b ? _a : _b;
    });
```
显然，由于变量名相同，导致在外部的变量`_a`和`_b`被复合语句内部的同名变量遮挡，导致最后的数值错误，但从程序语法来说，编译器并没有检查出什么问题。

让我们再将上面的例子变得更复杂一点，例如，我们递归它：
```c
#define maxint3(a, b, c) \
    ({\
        int _a = (a), _b = (b), _c = (c);\
        maxint(maxint(_a, _b), _c);\
    })
```
很显然，这次我们甚至不需要在外面定义`_a`和`_b`：
```c
int n = maxint3(2, 3, 5);
```
将会被替换为：
```c
int n = 
    ({
        int _a = (2), _b = (3), _c = (5);
        ({
            int _a =
            (({
                int _a = (_a), _b =(_b);
                _a > _b : _a : _b;
            })), _b = (_c);
            _a > _b ? _a : _b;
        })
    });
```
显然，由于同名变量导致的变量遮盖，这段代码最后的结果并不会理想。

    在常量表达式（如：初始化静态变量的值、位域宽度、枚举常量）中，不允许嵌入语句。

当我们使用复合表达式语句和函数调用时，需要注意到其中的临时变量的生命周期问题。对于函数调用来说，其实参求值期间引入的临时变量是在所调用函数执行完毕返回时销毁；但是对于符合表达式，其临时变量用完即销毁，并不会留存下来。我们用下面的代码来解释这一现象：
```c
#include <stdlib.h>
#define macro(a) ({int b = (a); b + 3; })
int 
X()
{
    return 5;
}

int 
func(int a) 
{
    int b = a; 
    return b + 3; 
}

int 
main(int argc, char * argv[])
{
    macro(X());
    func(X());
    exit(0);
}
```
显然，对于`macro()`来说，传递的参数`X()->5`会在执行完`int b = (a)`后销毁，但是对于函数调用`func()`来说，传递的参数会被接收到参数`a`，并在函数返回时才进行销毁。

使用`goto`跳转到某个语句或者使用一个在语句表达式外的`switch`语句，但是其某个`case`或`default`标签出现在语句表达式内部，这些都是不允许的。
```c
int
main()
{
    int temp = 0;
    switch(3){
        case 1:
            temp = 3;
            break;
        case 2:
            break;
        ({
            case 3:
            temp = 3;
            break;
        });
        default:
            temp = 4;
    }
    return 0;
}
```
这会导致编译器报错：
```shell
12:13: error: switch jumps into statement expression
   12 |             case 3:
      |             ^~~~error: switch jumps into statement expression
5:5: note: switch starts here
    5 |     switch(3){
      |     ^~~~~~
```
同时，使用[`Computed-goto`](#computed-goto)（我会在后面介绍它）跳转到语句表达式内是一个未定义行为，不过使用`Computed-goto`跳转到语句表达式外是可以的。但是，如果这个语句表达式是另一个较大的语句表达式的一部分，那么不指定这个较大表达式中的其他子表达式是否求值，除非语言定义中要求其中的某个子表达式在语句表达式之前或之后求值。
```c
#include <stdio.h>
#include <stdlib.h>
int foo(){puts("[foo]");return 5;}
int bar(){puts("[bar]");return 6;}
int hack(){puts("[hack]")return 7;}
int func(){puts("[func]");return 8;}
int
main()
{
    printf("valus = %d",({
        foo();
        ({
            bar();
            goto L_1;
            0;
        }) + hack();
        func();
    }));
    L_1:
    exit(0);
}
```
显然，这个程序会在执行到`goto L_1`的时候跳转到`L_1`然后结束程序。

但是仔细思考其中每个函数的调用问题，实际上我们可以发现：`foo()`函数和`bar()`函数一定被调用了，`func()`函数一定没有被调用，但是`hack()`函数是否被调用是取决于编译器的处理方式的，因此`hack()`函数有可能被调用，也有可能没有被调用。

一个存在于`while`、`for`、`do-while`循环、`switch`语句条件、`for`语句初始化或递增表达式中的语句表达式内如果有`break`或`continue`，则会跳转到外层的循环或者`switch`语句（如果外层没有嵌套其他语句，则被认为是错误的，而不是跳转到出现它的循环、`switch`语句、`for`语句的初始化或增量表达式中。

无论如何，就像函数调用一样，语句表达式的求值不会与包含表达式的其他部分的求值交叉。这听起来比较混乱，因此我们还是用两段代码来看看这个问题：
```c
#include <stdio.h>
#include <stdlib.h>
int
main(int argc, char * argv[])
{
    int i = 10;
    for(({i--; break; i;});i>0;)
        printf("i = %d\n",i);
    puts("END");
    exit(0);
}
```
我们知道，如果在`for`语句的初始化部分使用了`break`，且外部没有嵌套其他循环或`switch`语句时，编译器就会报错。上面这段代码在编译时，将会报出以下错误：
```shell
7:16: error: break statement not within loop or switch
    7 |     for(({i--; break; i;});i>0;)
      |                ^~~~~
```
我们对此稍加改动，在外层嵌套上循环语句：
```c
#include <stdio.h>
#include <stdlib.h>
int
main(int argc, char * argv[])
{
    int i = 10;
    for(;;) {
        for(({i--; break; i--;});i>0;) {
            printf("i = %d\n",i);
        }
    }
    printf("i = %d\n",i);
    puts("END");
    exit(0);
}
```
由于包含在内层`for`循环的初始化语句中的语句表达式中使用了`break`，而这是作用在外层`for`循环上的，因此将直接跳出外层`for`循环。最终运行结果为：
```shell
i = 9
END
```

## 局部标签声明

GCC允许你在嵌套的块作用域中声明局部标签。局部标签像普通标签一样，但是你只能在声明它的块作用域中引用它。

一个局部标签的声明方式如下：
```c
__label__ label;
```
或者
```c
__label label1, label2,...;
```
局部标签的声明必须在一个语句表达式的开头，处于任何其他语句之前（毕竟这是一个GCC特性，在开头会更好处理）。
局部标签的特性对于比较复杂的宏定义非常有用，如果一个宏包括了嵌套的循环，一个`goto`语句对于跳出循环很有帮助，如果一个宏在某个函数中多次被使用，那么标签也就会多次被定义，在这种情况下，使用局部标签显然可以避免多次定义标签的问题。我们用一段代码来举例：
```c
#include <stdio.h>
#include <stdlib.h>
#define SEARCH(value, array, target)                \
    do {                                            \
        __label__ end;                              \
        for(int i = 0; i < max; i++)                \
            for(int j = 0; j < max; j++)            \
                if(array[i][j] == (target)) {       \
                    (value) = i;                    \
                    goto end;                       \
                    }                               \
        (value) = -1;                               \
        end:;                                       \
    } while(0)
int
main(int argc, char * argv[])
{
    int array[3][3] = {{1,2,3},{2,3,4},{3,4,5}};
    int max = 3;
    int len;
    SEARCH(len, array, 5);
    if(len != -1)
        printf("5 in len %d\n", len+1);
    exit(0);
}
```
编译运行后，我们可以看到执行结果为：
```shell
5 in len 3
```
当然，我们也可以使用语句表达式重写上面的宏：
```c
#include <stdio.h>
#include <stdlib.h>
#define SEARCH(array, target)                   \
({                                              \
    __label__ end;                              \
    int i, j;                                   \
    for(i = 0; i < max; i++)                    \
        for(j = 0; j < max; j++)                \
            if(array[i][j] == (target))         \
                goto end;                       \
        i = -1;                                 \
        end:                                    \
        i;                                      \
})
int
main(int argc, char * argv[])
{
    int array[3][3] = {{1,2,3},{2,3,4},{3,4,5}};
    int max = 3;
    int len = SEARCH(array, 5);
    if(len != -1)
        printf("5 in len %d\n", len+1);
    exit(0);
}
```
显然，结果是一样的。

## 使标签作为数值
在GNU C中，我们可以获取当前函数（或包含的函数）中定义的标签的地址（实际上，这个地址就是标签后第一条指令的地址）。使用单目运算符`&&`（它通常被用作逻辑与），但在这里，它可以用来获得标签的地址，获取的数值类型为`void * `。这个数值是一个常量并且可以出现在任何允许该类型常量使用的地方。

我们使用一段代码作为示例：
```c
#include <stdio.h>
#include <stdlib.h>
int
main()
{
foo:
    printf("main address: %p\t\tfoo address:%p", main, &&foo);
    return 0;
}
```
在这里不得不提到的一点，标签只能是语句的一部分，因此标签只能放在某个语句前面，但不能放在声明的前面，如果你执意如此，那么我建议你使用一句空语句：
```c
int
main()
{
foo:;
    int i = 1;
    return 0;
}
```
如果你没有在`foo:`后加上分号`;`，那么你只能得到一个报错：
```shell
5:5: error: a label can only be part of a statement and a declaration is not a statement
    5 |     int i = 1;
      |  
```
在64位环境中，使用GCC编译并执行这条语句，得到的`foo`标签地址与`main`函数起始地址差值为`8`，这是因为在C语言中，使用了[栈帧](#什么是栈帧)来解决函数调用。
```asm
main:
.LFB6:
	.cfi_startproc
	endbr64
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
.L2:
	endbr64
	leaq	.L2(%rip), %rdx
	leaq	main(%rip), %rsi
	leaq	.LC0(%rip), %rdi
	movl	$0, %eax
	call	printf@PLT
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE6:
```
其中`.L2`实际上是`foo`标签编译后的替换。可以看到，在`.L2`标签之前，编译器添加了`pushq %rbp`和`movq %rsp, %rbp`指令。两条指令占据了`8`个字节，因此内存地址差值为`8`。

在使用这些标签的值时，你需要确定能够跳转到对应地址。这是通过`Computed-goto`语句完成的。
### Computed-goto
以一个例子来说明：
```c
int
main()
{
    int count = 0;
foo:;
    void * ptr = &&foo;
    if(count < 5){
        count++;
        goto *ptr;
    }
    return 0;
}
```
可以注意到，在这里`goto`并没有像往常一样使用标签来跳转，而是使用一个`void *`型变量值来实现跳转，实际上`goto *ptr`允许任何`void *`类型值。

使用这些常数地址的一种方法是初始化一个静态变量数组用作跳表：
```c
#include <stdio.h>
#include <stdlib.h>
int
main(int argc, char * argv[])
{
    static void * array[] = {&&L_1, &&L_2, &&L_3, &&L_4};
    int input = 0;
    if(argc == 2)
        input = atoi(argv[1]);
    goto *(array[input]);
L_1:
    puts("your didn't input");
    puts("your didn't input");
    goto *(array[3]);
L_2:
    puts("your input: 1");
    goto *(array[3]);
L_3:
    puts("your input: 2");
L_4:
    return 0;
}
```
但是需要注意到，对于C语言来说，是从来不检查下标是否在数组边界内的，因此，在上面的程序中，当你输入了超过`2`的正整数，程序在取得超过`array`边界的数据时并不会报错。

可以看出，这种写法与`switch`语句十分类似，但是有一个显著的差异，`switch`语句实现转移表只能正向转移，但是用`computed goto`实现转移表时可以实现反向转移，但是显然`switch`语句看起来更干净一些。因此，除非在解决某些不适于使用`switch`语句的特殊问题时，否则，使用`switch`语句是更好的选择。

另一中对标签值的使用是在解释器的线程代码中，解释器函数的标签可以存储在线程代码中以实现超快调度。

你最好不要使用这种机制从一个函数的代码中跳转到另一个函数的代码中，否则可能会引起不可预知的情况。最好的解决方案是使用局部作用域变量并且永远不将其作为参数传递。

首先解释为什么不要使用这种跳转方法：
```c
#include <stdlib.h>
void
bar(void * ptr)
{
    goto *ptr;
}

void
foo()
{
    bar(&&label_foo);
    label_foo:;
}
int
main(int argc, char* argv[])
{
    foo();
    exit(0);
}
```
编译并运行上面的函数，将会得到一个`segmentation fault`。这是因为在`bar()`函数中使用了`computed goto`直接跳转到函数`foo()`中而未使用`return`，导致栈帧与函数不再对应。因此我们最好不要使用`computed goto`机制在函数间进行跳转。

我们可以将上面的代码重写，使其更加成熟：
```c
#define GET_OPCODE(i)         i
#include <stdio.h>
#include <stdlib.h>
int
main()
{
    #ifdef USING_SWITCH_PATCH   
    #define dispatch(i)         switch(i)
    #define disp_case(i)        case i:
    #define disp_break          break
    typedef enum{
        OP_INPUT,
        OP_INC,
        OP_DEC,
        OP_ADD,
        OP_SUB,
        OP_EXIT,
        OP_END
    };
    #else
    #define DISTAB_MAX          (sizeof(dispatch_table)/sizeof(dispatch_table[0])-1)
    #define dispatch(i)         goto * (dispatch_table[i]);
    #define disp_case(i)        DO_##i:
    #define disp_break          goto * (dispatch_table[DISTAB_MAX])
    static const void * dispatch_table[] = {
        &&DO_OP_INPUT, 
        &&DO_OP_INC, 
        &&DO_OP_DEC,
        &&DO_OP_ADD,
        &&DO_OP_SUB,
        &&DO_OP_EXIT,
        &&DO_OP_END
    };
    #endif
    int i = 0, op = 0;
    for(int _loop = 0; _loop < 1;){
        int _temp = 0;
        system("clear");
        printf("i = %d\n", i);
        puts("0. reset i\n1. increse\n2. decrese\n3. add a num\n4. sub a num\n5. exit");
        scanf("%d",&op);
        dispatch(GET_OPCODE(op)) {
            disp_case(OP_INPUT){
                printf("input new i:");
                scanf("%d",&i);
                disp_break;
            }
            disp_case(OP_INC){
                i++;
                disp_break;
            }
            disp_case(OP_DEC){
                i--;
                disp_break;
            }
            disp_case(OP_ADD){
                printf("input the num: ");
                scanf("%d", &_temp);
                i += _temp;
                disp_break;
            }
            disp_case(OP_SUB){
                printf("input the num: ");
                scanf("%d", &_temp);
                i -= _temp;
                disp_break;
            }
            disp_case(OP_EXIT){
                _loop = 1;
                disp_break;
            }
            disp_case(OP_END){
            }
        }
    }
    exit(0);
}
```
由于我使用的是Linux系统，所以在实现屏幕清空时使用了`clear`，如果使用windows请自行更改为`cls`。



---
#### 什么是栈帧？
C语言的函数调用机制使用了栈结构提供的Last In First Out策略，每一个栈帧对应了一个尚未运行完的函数。函数的栈帧中保存了该函数的返回地址和局部变量，其中`rbp`寄存器指向当前栈帧的栈底，`rsp`寄存器指向函数栈的栈顶。

一个函数被调用后，首先要将调用它的函数的栈帧的栈底压入栈中作以保存，再将`rsp`寄存器所指向的内存位置作为当前函数的栈帧的栈底，如果在当前函数中还会调用，则会提前对`rsp`中存储的地址进行偏移，在堆栈中预留出保存局部变量的空间，更多的细节我们会在后面讨论。

#### 什么是共享库？
共享库又叫动态函数库，其中的函数在可执行程序启动时被加载，如果一个共享库被正常安装，那么所有的程序在重新运行时都可以自动加载最新的函数库中的函数。

在Linux中共享库遵循下面的命名方式：`lib` + `共享库名称` + `.so.` + `主版本号` + `.` + `次版本号` + `.` + `发行版本号`，譬如：`libread.so.3`。

每个共享函数库都有一个`so name`文件和一个`real name`文件，其中`real name`包含真正库函数代码的文件，而`so name`则是一个链接文件，链接到最新的库代码文件上。

    函数库是由系统建立的有一定功能的函数的集合，库中存放函数名称、对应的目标代码以及连接过程所需要的重定位信息。用户也可以根据自己的需要建立用户函数库。