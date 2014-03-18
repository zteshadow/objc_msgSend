objc_msgSend
============

hook objc_msgSend



目标是hook掉系统的objc_msgSend函数，以检测系统调用，方便开发与调试。主要是参考itrace的源代码，做了一捏捏的修改。

https://github.com/emeau/itrace

我的代码地址：


0. 目标：hook objc_msgSend函数

    我们可以很容易的借助mobile substrate的MSHookFunction的功能，替换掉系统的objc_msgSend函数。

typedef id (*objc_msg_function)(id self, SEL op, ...);
objc_msg_function s_originalObjcMsgSend;

MSHookFunction(objc_msgSend, our_msgSend, &s_originalObjcMsgSend);

这样，就用我们的our_msgSend函数替换了系统的objc_msgSend函数。

id our_msgSend(id self, SEL op, ...)
{
    //do our log 
    id result = 调用保存在s_originalObjcMsgSend里面的原函数
    return result.
}

在这里，我们可以看到问题变成了如何把参数正确的传入s_originalObjcMsgSend原函数。

问题1：参数传递
要正确的传递参数，需要补充一下ARM参数传递的基础知识。
ARM参数传递遵循ATPCS（Arm-Thumb Procedure Call Standard），分为定参和变参两种情况。
1. 定参：
ARM系统中使用寄存器（寄存器见附1）传递参数，r0, r1, r2, r3四个寄存器用来传递参数，如果参数超过4个，那么其余的参数由栈传递（入栈顺序与参数位置相反）。上面是每个参数都能用一个寄存器传递的情况，如果参数大小超过寄存器大小（4字节（32位机），8字节（64位机）），则会逐次的用r0，r1, r2, r3来存储参数，再超出的部分用栈来传递。

2. 变参
变参也同样遵循上面的规则，使用r0, r1, r2, r3四个寄存器传递，超出的部分入栈。
比如printf(“%d, %d, %d, %d”, a, b, c, d)
那么r0传递format字符串地址
r1传递a
r2传递b
r3传递c
sp传递d

根据上面的参数传递规则，由于objc_msgSend是变参函数，我们是无法知道到底有多少参数的，因此要想将参数原封不动的传递给原函数s_originalObjcMsgSend，只能保持进入我们函数的寄存器和栈指针都不变，这样才能保证原样参数调用原函数。再回过头来看我们的函数：
id our_msgSend(id self, SEL op, ...)
{
    //do our log 
    id result = 调用保存在s_originalObjcMsgSend里面的原函数
    return result.
}
我们必须得调用原函数，那么lr一定会发生变化（调用过程是设置lr为下一条指令地址：return result，然后跳入原函数），最后问题变成了保存lr。
id our_msgSend(id self, SEL op, ...)
{
    //保存lr
    //do our log
    //恢复所有的寄存器
    id result = 调用保存在s_originalObjcMsgSend里面的原函数
    //恢复lr
    return result.
}

问题2：保存lr
汇编是可以使用lr的，这里使用了inline汇编，注意XCode中支持的汇编是GNU汇编，并且在编译的过程中必须连接到真机进行编译，否则汇编中的r等寄存器会报错。
lr是函数调用时的返回地址，我们使用栈来保存所有的lr，调用入栈，返回出栈。
另外还要考虑线程安全的问题，这里使用了pthread_key_t——线程私有数据。

问题3：多进程使用文件的问题
多个进程使用log文件，是否会有问题？
理论上是有问题的，目前的试验结果是OK的，具体的文件在多进程环境的读写问题需要进一步研究。


附1. ARM寄存器
r0, r1, r2, r3四个寄存器是用来传递参数的
r4, r5, …, r11这些寄存器是通用的，在函数内部可以使用，但是用完需要恢复，所以一般函数里面会先把需要使用的寄存器入栈，比如如要使用r7作为临时变量，那么会有下面的调用：
push {r7, lr}
即把r7和返回地址入栈，等到函数要返回前，再出栈恢复r7寄存器。
pop {r7, lr}
r12
r13：sp栈指针寄存器，ARM使用FD栈，sp指向栈顶数据，且向下增长。
r14：lr保存返回地址——即调用该函数后下一条指令的地址
r15：pc——当前执行的指令地址

附2：ARM汇编与GNU编译属性
GNU汇编与ARM标准汇编是有区别的，带小点的是GNU汇编，比如：
.section .data
.section .bss
.section .text
上面对应数据段，0初始值的数据段和代码段

__attribute__((naked))
__attribute__是GNU C提供的编译器属性，可以设置函数的属性，比如这个naked设置阻止编译器生成函数入口或者退出代码，全部由自己处理。

__attribute__((constructor))
这个相当于构造函数，在加载该动态库的时候就会被调用

__attribute__((destructor))
卸载调用

__attribute__((naked))
naked属性阻止编译器生成任何函数入口，出口代码，需要自己定义入口出口代码的需要这个属性。

stmdb sp!, {r0-r8,r10-r12,lr}
批量存储，相当于
push lr
push r12
push r11
push r10
push r8
…
push r0
存储后，sp指向最后的r0

bl _bi_save_context
相当于lr = 下一条指令
bx  _bi_save_context调用函数

ldmia sp!, {r0-r8,r10-r12,lr}
与上面的stmdb相反，相当于：
pop r0
pop r1
…
pop r8
pop r10
..
pop lr

附3. 线程私有数据

[参考] http://www.cnblogs.com/godjesse/articles/2429762.html

全局变量，在一个进程中的所有所有线程共享，有竞态问题，有时候我们需要一个线程内部的全局变量概念，典型的就是线程自己的用户栈。
Posix线程库提供了这个功能——Thread Specific Data（TSD）.

创建TSD：
int pthread_key_create(pthread_key_t *key, void (*destruct_func)(void *));

附4： 动态库的加载问题
        	在iOS平台上，我们自己生成的dylib（动态库）是每个进程都有一个加载，还是所有的进程共享加载呢？
	这就涉及到了静态库和动态库的区别，所有的静态库，其实就是.o的集合，使用者把静态库里面的obj链接到自己的app可执行文件中，因此这有2个问题：1是体积过大浪费空间，比如说stdio库，所有的app都会用到，那么如果使用静态库，所有的app可执行文件中，都有这部分代码，执行中都需要加载到内存，浪费存储，浪费内存。2是如果这个部分的库需要更新，那么每个app都需要重新编译。为了解决上面的问题，动态库的概念被提出来，在链接时，只把动态库的链接信息放在可执行文件中了——比如动态库的地址等，这样就解决了第一个问题，至于第二个问题，如果动态库更新了，那么只需要操作系统重新加载更新的动态库到内存即可。这里有个问题，既然是所有的进程共享动态库，那么动态库里面的代码和数据该如何处理呢？.text代码段和只读的数据段都可以共享，那么可写的全局变量该怎么办呢？原来OS做了巧妙的设计动态库中只读数据在各个进程间是共享的，物理内存中只有一份拷贝，而可写的部分，设置成了写时拷贝（copy on write）属性，开始是共享的，一旦某个进程要进行修改，则拷贝一份，并映射到自己的内存空间。
	在iOS平台中，越狱的手机是可以加载动态库的，加载的时候，初始化函数会被调用（比如用属性__attribute__((constructor))声明的函数），既然每个链接该动态库的进程都会加载动态库，那么这个初始化函数应该被每个加载他的进程调用（实验证明这个是正确的，每个进程id会各自调用这个函数）。


【参考资料】

https://github.com/emeau/itrace

http://networkpx.blogspot.jp/2009/09/introducing-subjective-c.html


