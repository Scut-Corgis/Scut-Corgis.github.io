---
title: libco协程剖析
date: 2021-12-01 22:30:05
categories:
    - 协程
---
# libco源码阅读笔记

## 1.协程切换汇编

> 这部分引用了文章[libco中的coctx_swap汇编源码分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/42874353)
>
> 是我看过写的最清晰的

**coctx_swap汇编源码**

> 就是将当前协程上下文保存到regs中，然后将要运行的协程切换回来
>
> 充分利用了调用函数进入函数体时的rsp指向函数返回地址，实现了pc/rip寄存器的切换

基础知识：

X86-64有16个64位通用寄存器，分别是：%rax，%rbx，%rcx，%rdx，%esi，%edi，%rbp，%rsp，%r8，%r9，%r10，%r11，%r12，%r13，%r14，%r15。其中：

· %rax 作为函数返回值使用。
· %rsp 栈指针寄存器，指向栈顶
· %rdi，%rsi，%rdx，%rcx，%r8，%r9 用作函数参数，依次对应第1参数，第2参数。。。
· %rbx，%rbp，%r12，%r13，%14，%15 用作数据存储，遵循被调用者使用规则，简单说就是随便用，调用子函数之前要备份它，以防他被修改
· %r10，%r11 用作数据存储，遵循调用者使用规则，简单说就是使用之前要先保存原值

```c++
struct coctx_t
{
#if defined(__i386__)
void *regs[ 8 ];
#else
void *regs[ 14 ];
#endif
size_t ss_size;
char *ss_sp;
};
```

协程上下文中的regs数组与寄存器对应关系：

```c++
// 64 bit

//low | regs[0]: r15 |

// | regs[1]: r14 |

// | regs[2]: r13 |

// | regs[3]: r12 |

// | regs[4]: r9 |

// | regs[5]: r8 |

// | regs[6]: rbp |

// | regs[7]: rdi |

// | regs[8]: rsi |

// | regs[9]: ret | //ret func addr

// | regs[10]: rdx |

// | regs[11]: rcx |

// | regs[12]: rbx |

//hig | regs[13]: rsp |
```

假设调用coctx_swap(cur_ctx, new_ctx), cur_ctx代表当前运行的协程上下文，new_ctx是将要切换的协程上下文。下面开始分析汇编源码：

```assembly
leaq 8(%rsp),%rax //rsp是当前栈顶，8(%rsp)是跳过rip，指向调用coctxswap函数之前的栈顶。注意，coctx_swap的2个参数并没有入栈，而是存放在rdi,rsi寄存器中！

leaq 112(%rdi),%rsp // 112是偏移14个8字节位置，112(%rdi)指cur_ctx.regs[13]，即当前协程的栈顶rsp，将这个值存入rsp寄存器。注意这时栈顶指针改变了，指向数组cur_ctx.regs

pushq %rax //将栈顶存入cur_ctx.regs[13]

pushq %rbx //将寄存器rbx的值存入cur_ctx.reg[12]

pushq %rcx

pushq %rdx

pushq -8(%rax) //将coctx_swap的返回地址存入cur_ctx.reg[9]

pushq %rsi // 将寄存器rsi的值存入cur_ctx.reg[8]

pushq %rdi

pushq %rbp

pushq %r8

pushq %r9

pushq %r12

pushq %r13

pushq %r14

pushq %r15

movq %rsi, %rsp // 修改栈顶，使它指向新协程的数组，即指向new_ctx.regs[0]

popq %r15 // 将new_ctx.regs[0]的值存入r15寄存器

popq %r14

popq %r13

popq %r12

popq %r9

popq %r8

popq %rbp

popq %rdi

popq %rsi

popq %rax // 将返回地址，即new_ctx.regs[9]存入rax寄存器

popq %rdx

popq %rcx

popq %rbx
/* 下面两条是最核心部分，回复栈顶和设置返回地址, 然后跳转到对应协程 */
popq %rsp // 将new_ctx.regs[13]存入rsp，注意这时栈顶被修改了，指向new_ctx上次被yield时的栈顶！

pushq %rax // 上一次new_ctx挂起时的返回地址，入栈！

xorl %eax, %eax // 设置coctx_swap的返回值为0

ret // 返回，会跳到上一次new_ctx被挂起时的位置继续执行
```

其实当协程初始化时，即调用一个刚刚创立的协程需要为它设置 **栈指针**(函数运行时的栈的位置) 和 **返回地址**

```c
int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 )
```

我读完感觉以上两个函数和**ucontext**的函数效果基本没区别

```c
/* 摘录至云风coroutine源码 */
makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, xx, xx);
swapcontext(&S->main, &C->ctx); //切换上下文
```

其实就是以上两个linux提供的函数的具体实现。

据说libco的实现效率比linux的实现高3.6倍。

> 可能是因为libco只支持X86，并且只压了自己要用的寄存器，还不考虑线程掩码，减少了一次系统调用



## 2.网络I/O阻塞时协程的自动切换

在我看来，libco最大的特点和亮点，就是自己封装设计了一个网络I/O，如read、write、poll等等，使阻塞式同步I/O改为了**外表看上去还是阻塞式，但本质上为异步的实现**。

按照我看的一些源码博客，总结而言实现就是这几点

* **每个线程都有自己的主协程**，并有一个线程局部对象存放着协程调用顺序栈，比如A调用B，B又在里面调用C，那么这个线程局部对象就有一个数组存着【A B C】。这点很重要，这样协程在阻塞时，或者指向完毕时，可以找到调用它的协程，切换过去。
* 调用非阻塞I/O时，不进行处理，直接调用C库原函数，调用阻塞I/O时，调用自己封装的`poll`函数，此为整个libco的核心函数，将自己这个协程放入主协程的epoll检测事件红黑树中，注册一个回调函数，回调函数即唤醒这个协程（放入协程就绪队列）。同时主协程维护着一个时间轮，用于timeout计时，防止阻塞I/O时间过长，若超时了，主协程会删除协程对应文件描述符事件，同时也会触发协程就绪（否则协程永远阻塞了，就算超时也得告诉协程你超时了，比如read返回-1）。对于非超时的就绪协程，直接调用`resume`协程恢复执行。

## 3.其他

libco同时提供共享栈和独立栈两种模式。

经过源码阅读，发现共享栈模式和云风coroutine基本一模一样。只是使用了自己的`ucontext`函数。

独立栈模式最大的缺点就是一个栈要用128K空间，内存根本放不了太多协程。



libco也有hook手法，不过我学习完后觉得不是很重要，因此这里不记录了。

还是喜欢云风的代码，代码质量高的多。并且三言两语讲清楚了协程本质。



### **对照着golang的协程栈**

简单翻了翻go语言学习笔记，因为我印象中go语言没有使用共享栈，但是也能扩容，看看怎么实现的。

> go语言学习笔记371页

简单来说，它没有完全百分百解决栈溢出时扩容，但基本上不会发生，它是这样实现的：

* 协程G初始化为2KB.但是却又设置一个警戒线640Byte。所以它超过警戒线就会两倍扩容。

因此golang运行时的栈是在G协程中自己设置的栈中跑的。然后协程切换时，**M**会临时切换到一个叫做g0的协程栈，在G0协程栈中选择待运行的G任务来执行。



推荐博客

> [(30条消息) libco源码解析(5) poll_李兆龙的博客的博客-CSDN博客_libco源码分析](https://blog.csdn.net/weixin_43705457/article/details/106889805)
>
> [(30条消息) libco源码解析(6) co_eventloop_李兆龙的博客的博客-CSDN博客](https://blog.csdn.net/weixin_43705457/article/details/106891077)
>
> [(30条消息) libco源码解析(7) read，write与条件变量_李兆龙的博客的博客-CSDN博客](https://blog.csdn.net/weixin_43705457/article/details/106891691)

我个人觉得看这三篇就够了。然后结合云风的coroutine就能理解有栈协程了。
