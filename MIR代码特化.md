过去的三年里，我在MIR上花了我一半的工作时间，目标是创造一个通用的轻量级即时(JIT)编译器，而项目的基石是一个平台无关的中间表示(MIR)。有关项目更多的内容可以查阅我之前发在Red Hat Developer上的文章。

https://developers.redhat.com/blog/2020/01/20/mir-a-lightweight-jit-compiler-project
https://developers.redhat.com/blog/2021/04/27/the-mir-c-interpreter-and-just-in-time-jit-compiler

近期，我在MIR上的工作重点是做一个给主要的一些目标平台（x86-64 Linux和macOS, aarch64, s390x, riscv64 Linux, ppc64 big- 和 little-endian Linux）生成高质量机器代码的快速JIT编译器。

项目目前的版本是一个基于方法的JIT编译器，对于像C这样的静态语言可以有效地工作。我们已经开发了一个基于C到MIR编译器的C语言JIT编译器。

MIR最初的目标是实现一个更好的Ruby JIT编译器（尤其是我重点投入的由C编写的默认Ruby解释器——CRuby）Ruby是一个无比灵活的动态语言，你甚至可以重定义用于整数加法的方法。

为了让动态语言有更好的性能，你需要跟踪程序的执行，做许多的假设，并生成基于这些假设的代码。例如，你发现特定情况下加法计算时只有整数操作数参与，你可以假设这个情况不会改变，并生成针对整数特化的加法运算代码。

你需要用不同的方式来保证你的假设可靠，比如做一些假设检查，或是证明在特定执行路径下假设总是成立。如果检查发现假设出错，你需要切换到通用的代码上。这个切换过程在讨论JIT时一般称之为去优化。

这篇文章中，我将会介绍我怎么准备在MIR中实现生成特化和去优化的代码，和现阶段已经在MIR中实现的一些东西。

注意：大多数JIT编译器是针对特定语言的：JavaScript的V8，Lua的luajit，PHP的PHP JIT。我对在不同动态语言中支持JIT编译器的特化及去优化的实现更感兴趣。（看看OpenJDK的JIT编译器是怎么提高Java的性能的，可以了解更多JIT编译器中去优化的细节。）


# MIR中的特化及去优化
接下来我们可以具体看看MIR编译器中是怎么做代码特化和去优化的。
我会用以下简化后的代码来描述CRuby虚拟机(VM)中加法指令的实现：
```
    if (FIXNUM_2_P(recv, obj) &&
        BASIC_OP_UNREDEFINED_P(BOP_PLUS, INTEGER_REDEFINED_OP_FLAG)) {
        res = rb_fix_plus_fix(recv, obj);
    } else if (FLONUM_2_P(recv, obj) &&
               BASIC_OP_UNREDEFINED_P(BOP_PLUS, FLOAT_REDEFINED_OP_FLAG)) {
        res = DBL2NUM(RFLOAT_VALUE(recv) + RFLOAT_VALUE(obj));
    } else if (SPECIAL_CONST_P(recv) || SPECIAL_CONST_P(obj)) {
        ...  
    } else if (RBASIC_CLASS(recv) == rb_cFloat && RBASIC_CLASS(obj)  == rb_cFloat &&
               BASIC_OP_UNREDEFINED_P(BOP_PLUS, FLOAT_REDEFINED_OP_FLAG)) {
 ...
    } else if (RBASIC_CLASS(recv) == rb_cString && RBASIC_CLASS(obj) == rb_cString &&
               BASIC_OP_UNREDEFINED_P(BOP_PLUS, STRING_REDEFINED_OP_FLAG)) {
        ...
    } else if (RBASIC_CLASS(recv) == rb_cArray && RBASIC_CLASS(obj) == rb_cArray &&
               BASIC_OP_UNREDEFINED_P(BOP_PLUS, ARRAY_REDEFINED_OP_FLAG)) {
        ...
    } else {
        ... // call of method implemented + for object recv
    }
```

这些代码做了什么呢？首先，检查了操作数是定点数(整数)并且没有重定义整数加法的方法。如果成立，下面的代码将用于定点数的加法计算，否则将检查操作数类型是否是浮点数，字符串或数组。最后，代码调用了Ruby中对象recv实现的加法方法。

CRuby中的定点数都是能够被目标平台高效实现的整数子集，大数字则由GMP库以多精度数字的形式实现。

CRuby中的所有值都有特定的位标记它们的类型。例如，一个定点数总是有1个标志位，一个对象指针总是有3个(在32位平台上2个)（一般情况下为0的）标志位。以下代码是宏`FIXNUM_2_P`的实现：
```
  (recv & obj & 1) == 1
```

如果我们想在定点数加法中检查溢出则像这样：（译者注：表达存疑）
```
  (long) recv + (long) obj - 1
```

注意：为了表达简洁，下一节我将会忽略对加法运算重定义的检查，比如使用宏`BASIC_OP_UNREDEFINED_P`。

# 基于信息收集的代码特化
假设我们已经检查了一个特定加法中操作数的类型，并且发现近期执行的全是定点数加法，那么我们可以生成以下代码：
```
if (!FIXNUM_2_P(v1, v2)) goto general_case;
res = rb_fix_plus_fix(v1, v2)
```

乍一看，类型检查还在，我们并没有优化代码。那么，一系列的加法计算呢？
```
  //v1 + v2 + v3 + v4
  if (!FIXNUM_2_P(v1, v2)) goto general_case;
  res = rb_fix_plus_fix(v1, v2)
  if (!FIXNUM_2_P(res, v3)) goto general_case;
  res = rb_fix_plus_fix(res, v3)
  if (!FIXNUM_2_P(res, v4)) goto general_case;
  res = rb_fix_plus_fix(res, v4)
```

聪明的编译器可以去掉后两个检查，但不幸的是，GCC和Clang都不知道`res`有个标志位，所以它们不知道可以这样做。（GCC的Project Ranger完全实现后可能可以做到。）但如果值由一个两个成员(类型和值)组成的结构体表示，GCC/LLVM就会这么做。

# 扩展基础块上的优化
即便不移除多余的检查，编译器依然可以基于此做去掉多余的读写操作等的其它优化，
所以跑这样特化后的代码会更好。背后的原理是这些代码形成了一些特定的块（扩展基础快，EBBs），编译器可以很好地在这些块上做优化。像这样的代码还具备更好的局部性（可能涉及到多线程优化），以及在分支预测时命中率更高。

# 通用代码
那么我们怎么在定点数的例子中实现通用的部分呢？有三种可能：

* 切换到解释器执行。
* 让JIT编译器移除特化代码，并为VM指令生成通用代码。
* 跳转到一个有所有通用代码的位置上。

CRuby中切换到解释器执行开销很大，更优的方案是同时生成特化和通用的代码，并在特化代码假设不成立的情况下跳转到通用代码所在位置上。执行了一些跳转到通用代码的指令后，我们可以基于此重新生成整个方法的代码。

# 变量特性
MIR JIT编译器不比GCC或Clang聪明，在做前面的那种检查时也会有同样的问题。为了解决这些问题，我计划以C内建函数的形式为变量和MIR指令添加一些特性：
```
__builtin_prop_cond (cond, var1, property_const1, var2, property_const2, ...)
__builtin_prop_set (var, property_const)
```

注意：为了表达简洁，我将会跳过介绍MIR层上特性的实现。

特性都是整数常量。我们可以用`__builtin_prop_set`来为特定执行点上的变量设置特性，并在变量赋值时传递。

当我们无法在特定执行点上获知一个变量的特性时，将变量特性设为0，意味着这是个未知的特性。

接下来，我们可以对加法的代码用新的内建调用进行类型标注：
```
    enum prop { unknown = 0, inttype, flotype, ... };
    if (__builtin_prop_cond (FIXNUM_2_P(recv, obj), recv, intype, obj, intype)) {
        res = rb_fix_plus_fix(recv, obj);
        __builtin_prop_set (res, inttype)
    } else if (__builtin_prop_cond (FLONUM_2_P(recv, obj), recv, flotype, obj, flotype))
        res = DBL2NUM(RFLOAT_VALUE(recv) + RFLOAT_VALUE(obj));
        __builtin_prop_set (res, flotype);
    } else {
        ... // call of method implemented + for object recv
    }
```

下面的伪代码演示了怎么用`__builtin_prop_cond`和`__builtin_prop_set`：
```
 if (recv.prop == intype && obj.prop == inttype
        || (recv.prop == unknown && obj.prop == unknown) && FIXNUM_2_P(recv, obj)) {
        res = rb_fix_plus_fix(recv, obj);
        res.prop = intype;
    } else if (__builtin_prop_cond (FLONUM_2_P(recv, obj), recv, flotype, obj, flotype))
        res = DBL2NUM(RFLOAT_VALUE(recv) + RFLOAT_VALUE(obj));
        __builtin_prop_set (res, flotype);
    } else {
        ... // call of method implemented + for object recv
    }
```

因为我们在代码生成时知道这些特性，所有特性的赋值和比较操作最终都会被去掉。例如，如果我们知道`recv`和`obj`都是整数类型，最后的代码将如下：
```
  res = rb_fix_plus_fix(recv, obj);
```

如果我们在分析时知道`recv`和`obj`都是浮点类型，最后的代码将如下：
```
  res = DBL2NUM(RFLOAT_VALUE(recv) + RFLOAT_VALUE(obj));
```
