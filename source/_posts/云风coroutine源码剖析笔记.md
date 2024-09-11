---
title: 云风coroutine源码剖析笔记
date: 2022-04-09 22:30:00
categories:
    - 协程
---
# 云风coroutine源码剖析笔记

## 1. ucontext.h

linux实现了ucontext结构，使用户可以操作代码上下文，相当于封装了底层汇编语言操作寄存器指令，让用户获得了代码跳转能力。

ucontext_t 中保存的上下文主要包括以下几个部分：

* 运行时各个寄存器的值
* 运行栈
* 信号

```c
typedef struct ucontext {
                /* pointer to the context that will be resume when this context returns */
               struct ucontext *uc_link;
                /* the set of signals that are blocked when this context is active */
               sigset_t         uc_sigmask;
                /* the stack used by this context */
               stack_t          uc_stack;
                /* a machine-specific representation of the saved context */
               mcontext_t       uc_mcontext;
               ...
           } ucontext_t;
```

其中：

* `uc_link`：当前上下文结束时要恢复到的上下文，其中上下文由 `makecontext()` 创建；
* `uc_sigmask`：上下文要阻塞的信号集合；
* `uc_stack`：上下文所使用的 stack；
* `uc_mcontext`：其中 `mcontext_t`类型与机器相关的类型。这个字段是机器特定的保护上下文的表示，包括协程的机器寄存器；

**4个函数**

```c
#include <ucontext.h>

// 用户上下文的获取和设置
int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);

// 操纵用户上下文
void makecontext(ucontext_t *ucp, void (*func)(void), int argc, ...);
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
```

### `getcontext()`

将当前的 context 保存在 `ucp` 中。成功返回 0，错误时返回 -1 并设置 errno；

### `setcontext()`

恢复用户上下文为 `ucp` 所指向的上下文，成功调用**不用返回**。错误时返回 -1 并设置 errno。 `ucp` 所指向的上下文应该是 `getcontext()` 或者 `makecontext()` 产生。 如果上下文是由 `getcontext()` 产生，则切换到该上下文后，程序的执行在 `getcontext()` 后继续执行。比如下面这个例子每隔 1 秒将打印 1 个字符串：

```c
int main(void)
{
    ucontext_t context;

    getcontext(&context); //得到此处的上下文
    printf("Hello world\n");
    sleep(1);
    setcontext(&context); //跳到保存的上下文里面去，就是跳到printf之前去
    return 0;
}
```

### `makecontext()`

`makecontext`调用可以修改（*modify*）`getcontext`得到的上下文——也就是说make之前，你要先get。
`makecontext`可以做下面几件事：

- 指定运行栈
- 指定swap或者set调用后运行的函数（指定为func）
- 指定上面那个函数运行完后，要切换到的那个上下文

如果`setcontext()`上下文是由 `makecontext()` 产生，切换到该上下文，程序的执行切换到 `makecontext()` 调用所指定的第二个参数的函数上。当函数返回后，如果 `ucp.uc_link` 为 NULL，则结束运行；反之跳转到对应的上下文。

```c
void foo(void)
{
    printf("foo\n");
}
    
int main(void)
{
    ucontext_t context;
    char stack[1024];
       
    getcontext(&context);
    context.uc_stack.ss_sp = stack;
    context.uc_stack.ss_size = sizeof(stack);
    context.uc_link = NULL;
    makecontext(&context, foo, 0);
       
    printf("Hello world\n");
    sleep(1);
    setcontext(&context);
    return 0;
}
```

以上输出 `Hello world` 之后会执行 `foo()`，然后由于 `uc_link` 为 NULL，将结束运行。

下面这个例子：

```c
#include <stdio.h>
#include <ucontext.h>
#include <unistd.h>
void foo(void)
{
    printf("foo\n");
}
    
void bar(void)
{
    printf("bar\n");
}
    
int main(void)
{
    ucontext_t context1, context2;
    char stack1[1024];
    char stack2[1024];
       
    getcontext(&context1);
    context1.uc_stack.ss_sp = stack1;
    context1.uc_stack.ss_size = sizeof(stack1);
    context1.uc_link = NULL;
    makecontext(&context1, foo, 0);
       
    getcontext(&context2);
    context2.uc_stack.ss_sp = stack2;
    context2.uc_stack.ss_size = sizeof(stack2);
    context2.uc_link = &context1;
    makecontext(&context2, bar, 0);
        
    printf("Hello world\n");
    sleep(1);
    setcontext(&context2);
        
    return 0;
}
```

### `swapcontext()`

保存当前的上下文到 `ocup`，并且设置到 `ucp` 所指向的上下文。成功返回 0，失败返回 -1 并设置 errno。

如下面这个例子所示：

```c
#include <stdio.h>
#include <ucontext.h>
    
static ucontext_t ctx[3];
    
static void
f1(void)
{
    printf("start f1\n");
    // 将当前 context 保存到 ctx[1]，切换到 ctx[2]
    swapcontext(&ctx[1], &ctx[2]);
    printf("finish f1\n");
}
    
static void
f2(void)
{
    printf("start f2\n");
    // 将当前 context 保存到 ctx[2]，切换到 ctx[1]
    swapcontext(&ctx[2], &ctx[1]);
    printf("finish f2\n");
}
    
int main(void)
{
    char stack1[8192];
    char stack2[8192];
    
    getcontext(&ctx[1]);
    ctx[1].uc_stack.ss_sp = stack1;
    ctx[1].uc_stack.ss_size = sizeof(stack1);
    ctx[1].uc_link = &ctx[0]; // 将执行 return 0
    makecontext(&ctx[1], f1, 0);
    
    getcontext(&ctx[2]);
    ctx[2].uc_stack.ss_sp = stack2;
    ctx[2].uc_stack.ss_size = sizeof(stack2);
    ctx[2].uc_link = &ctx[1];
    makecontext(&ctx[2], f2, 0);
    
    // 将当前 context 保存到 ctx[0]，切换到 ctx[2]
    swapcontext(&ctx[0], &ctx[2]);
    return 0;
}   
```

此时将输出：

```fallback
start f2
start f1
finish f2
finish f1
```

## 2.coroutine

### 源码注释

> 因为源码很短，所以直接贴过来

![](image-20220409151929569.png)

#### coroutine.h

```c
#ifndef C_COROUTINE_H
#define C_COROUTINE_H

#define COROUTINE_DEAD 0
#define COROUTINE_READY 1
#define COROUTINE_RUNNING 2
#define COROUTINE_SUSPEND 3

// 协程调度器

// 为了ABI兼容，这里故意没有提供具体实现

struct schedule;

typedef void (*coroutine_func)(struct schedule *, void *ud);

// 开启一个协程调度器
struct schedule * coroutine_open(void);

// 关闭一个协程调度器
void coroutine_close(struct schedule *);

// 创建一个协程
int coroutine_new(struct schedule *, coroutine_func, void *ud);

// 切换到对应协程中执行
void coroutine_resume(struct schedule *, int id);

// 返回协程状态
int coroutine_status(struct schedule *, int id);

// 协程是否在正常运行
int coroutine_running(struct schedule *);

// 切出协程
void coroutine_yield(struct schedule *);

#endif
```
#### coroutine.c

```c
#include "coroutine.h"
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <stddef.h>
#include <string.h>
#include <stdint.h>
#include <ucontext.h>

#define STACK_SIZE (1024*1024)
#define DEFAULT_COROUTINE 16

struct coroutine;

/* 协程调度器 */
struct schedule {
	char stack[STACK_SIZE];	// 运行时栈
	ucontext_t main; // 主协程的上下文
	int nco; // 当前存活的协程个数
	int cap; // 协程管理器的当前最大容量，即可以同时支持多少个协程。如果不够了，则进行扩容
	int running; // 正在运行的协程ID
	struct coroutine **co; // 一个一维数组，用于存放协程 
};

/* 协程 */
struct coroutine {
	coroutine_func func; // 协程所用的函数
	void *ud;  // 协程参数
	ucontext_t ctx; // 协程上下文
	struct schedule * sch; // 该协程所属的调度器
	ptrdiff_t cap; 	 // 已经分配的内存大小
	ptrdiff_t size; // 当前协程运行时栈，保存起来后的大小
	int status;	// 协程当前的状态
	char *stack; // 当前协程的保存起来的运行时栈
};
/*
* 新建一个协程
* 主要做的也是分配内存及赋初值
*/
struct coroutine * _co_new(struct schedule *S , coroutine_func func, void *ud) {
	struct coroutine * co = malloc(sizeof(*co));
	co->func = func;
	co->ud = ud;
	co->sch = S;
	co->cap = 0;
	co->size = 0;
	co->status = COROUTINE_READY; // 默认的最初状态都是COROUTINE_READY
	co->stack = NULL;
	return co;
}
/**
* 删除一个协程
*/
void _co_delete(struct coroutine *co) {
	free(co->stack);
	free(co);
}
/*
* 创建一个协程调度器 
*/
struct schedule * coroutine_open(void) {
	// 这里做的主要就是分配内存，同时赋初值
	struct schedule *S = malloc(sizeof(*S));
	S->nco = 0;
	S->cap = DEFAULT_COROUTINE;
	S->running = -1;
	S->co = malloc(sizeof(struct coroutine *) * S->cap);
	memset(S->co, 0, sizeof(struct coroutine *) * S->cap);
	return S;
}
/**
* 关闭一个协程调度器，同时清理掉其负责管理的
* @param S 将此调度器关闭
*/
void coroutine_close(struct schedule *S) {
	int i;
	// 关闭掉每一个协程
	for (i=0;i<S->cap;i++) {
		struct coroutine * co = S->co[i];
		if (co) {
			_co_delete(co);
		}
	}
	// 释放掉
	free(S->co);
	S->co = NULL;
	free(S);
}
/**
* 创建一个协程对象
* @param S 该协程所属的调度器
* @param func 该协程函数执行体
* @param ud func的参数
* @return 新建的协程的ID
*/
int coroutine_new(struct schedule *S, coroutine_func func, void *ud) {
	struct coroutine *co = _co_new(S, func , ud);
	if (S->nco >= S->cap) {
		// 如果目前协程的数量已经大于调度器的容量，那么进行扩容
		int id = S->cap;	// 新的协程的id直接为当前容量的大小
		// 扩容的方式为，扩大为当前容量的2倍，这种方式和Hashmap的扩容略像
		S->co = realloc(S->co, S->cap * 2 * sizeof(struct coroutine *));
		// 初始化内存
		memset(S->co + S->cap , 0 , sizeof(struct coroutine *) * S->cap);
		//将协程放入调度器
		S->co[S->cap] = co;
		// 将容量扩大为两倍
		S->cap *= 2;
		// 尚未结束运行的协程的个数 
		++S->nco; 
		return id;
	} else {
		// 如果目前协程的数量小于调度器的容量，则取一个为NULL的位置，放入新的协程
		int i;
		for (i=0;i<S->cap;i++) {
			/* 
			 * 为什么不 i%S->cap,而是要从nco+i开始呢 
			 * 这其实也算是一种优化策略吧，因为前nco有很大概率都非NULL的，直接跳过去更好
			*/
			int id = (i+S->nco) % S->cap;
			if (S->co[id] == NULL) {
				S->co[id] = co;
				++S->nco;
				return id;
			}
		}
	}
	assert(0);
	return -1;
}
/*
 * 通过low32和hi32 拼出了struct schedule的指针，这里为什么要用这种方式，而不是直接传struct schedule*呢？
 * 因为makecontext的函数指针的参数是int可变列表，在64位下，一个int没法承载一个指针
*/
static void mainfunc(uint32_t low32, uint32_t hi32) {
	uintptr_t ptr = (uintptr_t)low32 | ((uintptr_t)hi32 << 32);
	struct schedule *S = (struct schedule *)ptr;
	int id = S->running;
	struct coroutine *C = S->co[id];
	C->func(S,C->ud);	// 中间有可能会有不断的yield
	_co_delete(C);
	S->co[id] = NULL;
	--S->nco;
	S->running = -1;
}
/**
* 切换到对应协程中执行
* @param S 协程调度器
* @param id 协程ID
*/
void coroutine_resume(struct schedule * S, int id) {
	assert(S->running == -1);
	assert(id >=0 && id < S->cap);
    // 取出协程
	struct coroutine *C = S->co[id];
	if (C == NULL)
		return;
	int status = C->status;
	switch(status) {
	case COROUTINE_READY:
	    //初始化ucontext_t结构体,将当前的上下文放到C->ctx里面
		getcontext(&C->ctx);
		// 将当前协程的运行时栈的栈顶设置为S->stack
		// 每个协程都这么设置，这就是所谓的共享栈。（注意，这里是栈顶）
		C->ctx.uc_stack.ss_sp = S->stack; 
		C->ctx.uc_stack.ss_size = STACK_SIZE;
		C->ctx.uc_link = &S->main; // 如果协程执行完，将切换到主协程中执行
		S->running = id;
		C->status = COROUTINE_RUNNING;
		// 设置执行C->ctx函数, 并将S作为参数传进去
		uintptr_t ptr = (uintptr_t)S;
		makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));
		// 将当前的上下文放入S->main中，并将C->ctx的上下文替换到当前上下文
		swapcontext(&S->main, &C->ctx);
		break;
	case COROUTINE_SUSPEND:
	    // 将协程所保存的栈的内容，拷贝到当前运行时栈中
		// 其中C->size在yield时有保存
		memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
		S->running = id;
		C->status = COROUTINE_RUNNING;
		swapcontext(&S->main, &C->ctx);
		break;
	default:
		assert(0);
	}
}
/*
* 将本协程的栈内容保存起来
* @top 栈顶 
*/
static void _save_stack(struct coroutine *C, char *top) {
	// 这个dummy很关键，是求取整个栈的关键
	// 这个非常经典，涉及到linux的内存分布，栈是从高地址向低地址扩展，因此
	// S->stack + STACK_SIZE就是运行时栈的栈底
	// dummy，此时在栈中，肯定是位于最底的位置的，即栈顶
	// top - &dummy 即整个栈的容量
	char dummy = 0;
	assert(top - &dummy <= STACK_SIZE);
	if (C->cap < top - &dummy) {
		free(C->stack);
		C->cap = top-&dummy;
		C->stack = malloc(C->cap);
	}
	C->size = top - &dummy;
	memcpy(C->stack, &dummy, C->size);
}
/**
* 将当前正在运行的协程让出，切换到主协程上
* @param S 协程调度器
*/
void coroutine_yield(struct schedule * S) {
	// 取出当前正在运行的协程
	int id = S->running;
	assert(id >= 0);
	struct coroutine * C = S->co[id];
	assert((char *)&C > S->stack);
	// 将当前运行的协程的栈内容保存起来
	_save_stack(C,S->stack + STACK_SIZE);
	// 将当前栈的状态改为 挂起
	C->status = COROUTINE_SUSPEND;
	S->running = -1;
	// 所以这里可以看到，只能从协程切换到主协程中
	swapcontext(&C->ctx , &S->main);
}

int coroutine_status(struct schedule * S, int id) {
	assert(id>=0 && id < S->cap);
	if (S->co[id] == NULL) {
		return COROUTINE_DEAD;
	}
	return S->co[id]->status;
}
/*
* 获取正在运行的协程的ID
* @param S 协程调度器
* @return 协程ID 
*/
int coroutine_running(struct schedule * S) {
	return S->running;
}
```
#### main例子

```c
#include "coroutine.h"

#include <stdio.h>

struct args {
	int n;
};

static void foo(struct schedule * S, void *ud) {
	struct args * arg = ud;
	int start = arg->n;
	int i;
	for (i=0;i<5;i++) {
		printf("coroutine %d : %d\n",coroutine_running(S) , start + i);
		// 切出当前协程
		coroutine_yield(S);
	}
}

static void test(struct schedule *S) {
	struct args arg1 = { 0 };
	struct args arg2 = { 100 };
	// 创建两个协程
	int co1 = coroutine_new(S, foo, &arg1);
	int co2 = coroutine_new(S, foo, &arg2);
	printf("main start\n");
	while (coroutine_status(S,co1) && coroutine_status(S,co2)) {
		// 使用协程co1
		coroutine_resume(S,co1);
		// 使用协程co2
		coroutine_resume(S,co2);
	} 
	printf("main end\n");
}

int main() {
	// 创建一个协程调度器
	struct schedule * S = coroutine_open();
	test(S);
	// 关闭协程调度器
	coroutine_close(S);
	return 0;

}
```

### 笔记心得

其实每个协程都有自己的栈，然后还设置了一个共享栈，即调度器中的设置的共享栈。

当调度到某个协程时，即调用`coroutine_resume();`，就将其原来的栈内容复制到共享栈中，在共享栈中跑函数代码。

当需要将协程挂起时，即调用`coroutine_yield()`; 将共享栈中的内容再保存到自己私有栈中，切换回主协程。主协程在云风代码中其实就是main函数，然后main函数会再次调用`coroutine_resume()`,切换进别的协程，换句话说，

共享栈的手法得以使调度器对所有协程进行统一管理，

> 注意此共享栈和操作系统的栈不一样，协程所有栈都是在堆上的，只有main函数在栈上
>
> 	coroutine_resume() {
> 			makecontext(&C->ctx, (void (*)(void)) mainfunc);
> 			// 将当前的上下文放入S->main中，并将C->ctx的上下文替换到当前上下文
> 			swapcontext(&S->main, &C->ctx);
> 	}
> 		//这里就是保存了main函数的上下文，当yield的时候就能找回了

云风的协程库需要用户手动的yield和resume。

> 有趣的是，我用gdb调试不了ucontext函数足，也许也就是修改了底层pc，rsp寄存器的原因吧。

共享栈对标的是非共享栈，也就是每个协程的栈空间都是独立的，固定大小。好处是协程切换的时候，内存不用拷贝来拷贝去。坏处则是**内存空间浪费**.

因为栈空间在运行时不能随时扩容，为了防止栈内存不够，所以要预先每个协程都要预先开一个足够的栈空间使用。当然很多协程用不了这么大的空间，就必然造成内存的浪费。

> [ucontext 函数族的使用及协程库的实现 - 晒太阳的猫 (zhengyinyong.com)](https://zhengyinyong.com/post/ucontext-usage-and-coroutine/)



