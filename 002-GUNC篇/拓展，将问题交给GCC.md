# 表达式中的语句和声明
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
同时，使用`Computed-goto`（我会在后面介绍它）跳转到语句表达式内是一个未定义行为，不过使用`Computed-goto`跳转到语句表达式外是可以的。但是，如果这个语句表达式是另一个较大的语句表达式的一部分，那么不指定这个较大表达式中的其他子表达式是否求值，除非语言定义中要求其中的某个子表达式在语句表达式之前或之后求值。
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
