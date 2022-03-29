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
So that we can look specifically at specialization and deoptimization in the MIR compiler, I will use the following simplified code for the virtual machine (VM) instruction plus in the CRuby implementation:

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
        .. // call of method implementing + for object recv
    }
```