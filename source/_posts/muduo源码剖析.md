---
title: muduo源码剖析
date: 2021-12-01 22:30:05
categories:
    - others
---
# Muduo源码剖析笔记

# 前言

注意本笔记只适用于**阅读过**陈硕的书籍《linux多线程服务端编程》的读者。

本笔记只讲书籍中没有提到的，或者提到但是书中描写寥寥的，为在阅读实际源码的基础上整理出来的笔记。

因此**若你对其书籍中的知识还不够理解，请回去阅读书籍**，若已经掌握书中思想了，那么本笔记便很适合你进一步了解其实际源码实现的，但书籍中没有说明的重要部分。

> 建议github下载muduo源码跟着一起学习

让我们开始吧！



## 1. 日志系统

主要模型为：多线程多级异步日志，实现日志前后端分离，并高效的解决了临界区问题

**muduo**日志库分为前端和后端两个部分，前端用于生成日志消息并传送到后端，后端则负责将日志消息写入本地日志文件。日志系统的前后端之间只有一个简单的回调函数作为接口

```c++
Logger::OutputFunc g_output
typedef void (*OutputFunc)(const char* msg, int len);
/* 注意muduo默认不打开异步日志，因此需要进行设置，即声明一个AsyncLogging类对象，并设置outputFunc回  调 */
Logger::OutputFunc g_output = defaultOutput; //defaultOutput是往STDOUT写
```
比如可以这样修改源码设置, 这样在第一次调用日志接口时，便生成日志后端线程，并开始工作

```c
static AsyncLogging* AsyncLogger_;
void once_init() {
    AsyncLogger_ = new AsyncLogging(Logger::getLogFileName());
    AsyncLogger_->start();
}
void output(const char* msg, int len) {
    pthread_once(&once_control_, once_init);
    AsyncLogger_->append(msg, len);
}
Logger::OutputFunc g_output = output;
```

其中msg是一条完整的日志消息，包含时间戳、日志等级、线程id、日志内容和位置信息六个部分

> 2021-11-28 11:24:23 DEBUG 131880  WAIT FOR 3 SECONDS -- test.cc:42

在多线程网络服务器程序中，各个线程的功能必然有所区分，那怎样将其他非日志线程产生的日志消息高效地传输到后端日志线程中呢。这就要求设计一个高效的日志库，它对外提供一个统一的接口（**muduo**库中提供的对外接口为LOG宏），这样其他非日志线程只需对这个接口进行简单的调用就能实现所需的日志功能。

这是一个典型的多生产者-单消费者的问题，对生产者（前端）而言，要尽量做到低延迟、低CPU开销、无阻塞；对消费者（后端）而言，要做到足够大的吞吐量，并尽量占用较少的资源。

**基本日志功能实现**

日志模块前端部分的调用时序为：

`Logger`  =>  `Impl` =>  `LogStream`  =>  `operator<<FixBuffer`  =>  `g_output`  =>  `AsyncLogging:: append`

```c++
#define LOG_TRACE if (irono::Logger::LogLevel() <= irono::Logger::TRACE) \
    irono::Logger(__FILE__, __LINE__, irono::Logger::TRACE).stream()
#define LOG_DEBUG if (irono::Logger::LogLevel() <= irono::Logger::DEBUG) \
    irono::Logger(__FILE__, __LINE__, irono::Logger::DEBUG).stream()
```
使用LOG宏时会创建一个匿名Logger对象（其中包含一个Impl类型的成员变量），并调用stream()函数得到一个LogStream对象的引用，而LogStream重载了<<操作符，可以将日志信息存入LogStream的buffer中。这样LOG语句执行结束时，匿名Logger对象被销毁，在Logger的析构函数中，会在日志消息的末尾添加LOG语句的位置信息（文件名和行号），最后调用g_output()函数将日志信息传输到后端，由后端日志线程将日志消息写入日志文件。

```c++
Logger::~Logger() {
    impl_.stream_ << " -- " << impl_.filename_ << ":" << impl_.line_ << '\n';
    const LogStream::Buffer& buf(stream().buffer());
    g_output(buf.data(), buf.length());
}
```
强调一下，这里将Logger设置为匿名对象是一个非常重要的技巧，因为匿名对象是一使用完就马上销毁，而对于栈上的具名对象则是先创建的后销毁。也就是说，如果使用具名对象，则后创建的Logger对象会先于先创建的Logger对象销毁，这就会使得日志内容反序（更准确的说是一个作用域中的日志反序）。使用匿名Logger对象的效果就是：LOG_*这行代码不仅仅包含日志内容，还会马上把日志输出（并不一定会立即写到日志文件中，具体原因见多线程异步日志部分）。

到这里，基本的日志功能已经实现了（只实现日志消息的生成，但还没有将其传输到后端），但这还不是异步的。

**多线程异步日志**

与单线程程序不同，多线程程序对日志库提出了新的要求——线程安全，即多个线程可以并发的写日志而不会出现混乱。简单的线程安全并不难办到，用一个全局的mutex对日志的IO进行保护或是每个线程单独写一个日志文件就可以办到。但是前者会造成多个线程争夺mutex，后者则可能使得业务线程阻塞在磁盘操作上。

其实可行的解决方案已经给出，即用一个背景线程负责收集日志消息并写入日志文件（后端），其他的业务线程只负责生成日志消息并将其传输到日志线程（前端），这被称为“异步日志”。

在多线程服务程序中，异步日志（叫“非阻塞日志”似乎更准确些）是必须的，因为如果在网络IO或业务线程中直接往磁盘上写数据的话，写操作可能由于某种原因阻塞长达数秒之久。这可能使得请求方超时，或是耽误心跳消息的发送，在分布式系统中更可能造成多米诺骨牌效应，例如误报死锁引发failover（故障转移）。因此，在正常的实时业务处理流程中应该彻底避免磁盘IO。

即AsyncLogging::append部分，也就是前端如何将日志消息发送到后端。

**muduo**日志库采用了**双缓冲技术**，即预先设置两个buffer（currentBuffer和nextBuffer），前端负责往currentBuffer中写入日志消息，后端负责将其写入日志文件中。具体来说，当currentBuffer写满时，先将currentBuffer中的日志消息存入buffers，再将nextBuffer（std::move）移动赋值currentBuffer，这样前端就可以继续往currentBuffer_中写入新的日志消息，最后再调用`notify_all()`通知后端日志线程将日志消息写入日志文件。注意，移动的实际是Buffer的unique_ptr,即unique_ptr实现了移动而禁止了赋值。

用两个buffer的好处是在新建日志消息时不必等待磁盘文件操作，也避免了每条新的日志消息都触发后端日志线程。换句话说，前端不是将一条条日志消息分别发送给后端，而是将多条日志消息合成为一个大的buffer再发送给后端，相当于批处理，这样就减少了日志线程被触发的次数，降低了开销。

具体见`/base/AsyncLogging.cc`

前端在生成一条日志消息的时候会调用`AsyncLogging::append()`。在这个函数中，如果currentBuffer剩余的空间足够容纳这条日志消息，则会直接将其拷贝到currentBuffer中，否则说明currentBuffer已经写满，因此将currentBuffer移入buffers，并将nextBuffer移用为currentBuffer，再向currentBuffer中写入日志信息。一种特殊的情况是前端日志信息写入速度过快，一下子把currentBuffer和nextBuffer都用光了，那么只好分配一块新的currentBuffer_作为当前缓存。

后端日志线程首先准备两块空闲的buffer，以备在临界区内交换。在临界区内，等待条件触发，这里的条件有两个，其一是超时，其二是前端写满了currentBuffer_。注意这里是非常规的condition variable用法，没有使用while循环，为了实现定时flush。

当条件满足时，先将currentBuffer移入buffers，并立即将空闲的newBuffer1移为currentBuffer。注意这段代码位于临界区内，因此不会有任何的race condition。接下来交换bufferToWrite和buffers，这就完成了将记录了日志消息的buffer从前端到后端的传输，后端日志线程慢慢进行IO即可。临界区最后干的一件事就是将空闲的newBuffer2移为nextBuffer_，这样就实现了前端两个buffer的归还。

**重要细节总结：**

1. `FileUtil.h`实现的类AppenFile做为最底层与文件打交道的类,因为只有日志后端线程唯一的拥有这样一个类的unique_ptr,所以理论上写文件不用加锁,可以用效率高的多`fwrite_unlocked()`,而且使用了`setbuffer()`为文件创立了在用户栈上缓冲区,只有每3次append才会fflush,减少了磁盘IO次数.

2. 日志前端即使用宏LOG_TRACE等输出日志信息的线程,会进入`AsyncLogging`类,使用其成员函数append,并用了`AsyncLogging`类的数据成员currentBufeer和nextBuffer作为前端日志缓冲区,日志前端需要全程加锁,不过临界区很短,nextBuffer和currentBuffer_的切换也采用的移动语义,一但当前的currentBuffer写满了则会切换,并在临界区结束前唤醒日志后端线程.

3. 日志后端线程全程运行在`AsyncLogging`类的`threadFunc()`中,并时刻提供了两个全新的缓冲区buffer,用来快速替换日志前端的缓冲Buffer,并且使用了临时量vector交换方法,将前后端对应临界区降至最低,基本上只有几次移动替换缓冲区的过程,最大限度的降低的前后端的竞争.

4. 日志后端的逻辑,大致为前端写满了一个Buffer就通过条件变量唤醒,或者超时唤醒.Logging线程全程存在于程序的运行期.

5. `CountDownLatch`,我称之为记数门阀,是一种线程同步手段,其中前面的线程完成了某些条件便会减少门阀计数,当计数为0则使用条件变量唤醒等在门前的后面的线程.

6. 整个日志逻辑: 生成`LogStream`类,其中有个临时Buffer,一行"<<"的都写在这个临时Buffer里面,然后这行结束,类自动析构,析沟时会往日志前端写日志,接着则是前后端日志的逻辑了.这样对外提供了非常简洁易用的接口,而隐藏了复杂的日志内部细节.
   
7. 日志后端线程于首次使用日志宏LOG时创建，并全局创建一次。
   

**疑难问题**
1. 日志前后端速度不匹配会怎样？比如日志前端实时产生了大量日志，而后端无法处理的情况。
   
   *答*：若瞬时产生了大量日志信息，即前端因为写完了两个缓冲Buffer，则会不断申请新的Buffer写并填入buffers,理论上每次写完了一个buffer都会通知后端日志线程写I/O，但若某次buffers里的存量过多，磁盘I/O写日志的速度跟不上产生了日志的速度，则会在buffers产生大量积累，本项目日志后端线程在磁盘I/O前会检查buffers中的buffer数量，若超过了25，则简单丢弃后面所有日志buffer只写前两个，以防止内存爆炸的情况发生。
   
   

## 2. 时间轮

时间轮陈硕书中其实说的很详细，但是其中有一些微妙的细节，在我阅读源码时发现很重要，因此记录在此。

**circular_buffer**

陈硕采用了boost::circular_buffer的实现，我尝试的阅读其源码，发现过于晦涩只能放弃，但实际上完全可以不用circular_buffer，这里提供一种实现，简洁又不失优雅。

总的来说，通过封装std::List，初始化其listnode数量为桶数量，时间轮指针移动，对应list尾端桶全部析构，在前端插入新的空桶，就完成了一个circurlar_buffer

```c++
template<typename T>
class circular_buffer {
    public:
    circular_buffer(int bucketSize) : bucketSize_(bucketSize) {
        int n = bucketSize;
        while (n--) {
            buffers_.push_back(T());
        }
    }
    // 时间轮指针移动，对应list尾端桶全部析构，在前端插入新的空桶
    void Tick() {
        buffers_.pop_back();
        buffers_.push_front(T());
    }
    int size() const{
        int res = 0;
        for (auto iter = buffers_.begin(); iter != buffers_.end(); iter++) {
            res += (*iter).size();
        }
        return res;
    }
    typename std::list<T>::iterator begin() {
        return buffers_.begin();
    }
    typename  std::list<T>::iterator end() {
        return buffers_.end();
    }
    private:
    std::list<T> buffers_;
    int bucketSize_;
};
```

**boost:any<>和 any_cast**

但时间轮实际上还有一个问题，当来新连接时，如何找到原来连接entry的weak_ptr,你实际上只拥有函数回调时的TcpConnetionPtr，它是无状态的没有保存任何的之前的信息。

更重要的是，使用框架的用户很有可能需要把自己的用户数据存在这个TcpConnetion对象里面，这样每次回调时就能得到其用户数据进行相应的操作，因此**muduo**源码里TcpConnection类里存在一个数据成员context_

```c++
class TcpConnection {
  boost::any context_;	                          
}
```

可以将任何类转换为any类，相当于go的interface{}, 其实现是通过any类的内部嵌入了一个模板子类，并拥有其指针。`any_cast`使用了`type_id`的手法

>  源码：[(36条消息) [C++\] C++中boost::any的使用_Glemontree_的博客-CSDN博客_boost::any](https://blog.csdn.net/u010216743/article/details/77772078)
>
>  之所以any的成员用基类指针，实际存的子类holder数据，是因为不想让any表现为模板类，从而让any只是拥有了一个模板构造函数，却不是模板类，不然无法使用，因为没法提供any的模板参数

这样在连接到来时或断开时，时间轮就可以注册自己的用户数据了

```c
    /* 连接建立时回调 */
	if (conn->connected()) {
        EntryPtr entry(new Entry(conn));
        (*connectionBuckets_.begin()).insert(entry);

        WeakEntryPtr weakEntry(entry);
        conn->setContext(weakEntry); //将自己的weakEntry放进去存着
    }
	/* 连接断开时回调,当然收到网络信息回调也是这样做的 */
    else {
        // 把存进去的取出来
        WeakEntryPtr weakEntry(boost::any_cast<WeakEntryPtr>(conn->getContext()));
    }
```



## 3.高水位回调

书中提到的输出缓冲区高水位回调，防止对方迟迟不收信息或者网络问题等导致输出缓冲区膨胀，内存爆炸。

```c++
/* 源码没有设置，只提供了一个接口，因此使用muduo库的用户需要自己设置 */
void setHighWaterMarkCallback(const HighWaterMarkCallback& cb, size_t highWaterMark)
  { highWaterMarkCallback_ = cb; highWaterMark_ = highWaterMark; }

void TcpConnection::sendInLoop(const void* data, size_t len) {
    ...
     /* 发送时触发了高水位，则执行回调函数 */
    if (oldLen + remaining >= highWaterMark_
        && oldLen < highWaterMark_
        && highWaterMarkCallback_)
    {
      loop_->queueInLoop(std::bind(highWaterMarkCallback_, shared_from_this(), oldLen + remaining));
    }
}
```

## 4.三种聊天室程序

陈硕在书中提到了三种聊天室程序，性能依次提高，并在最后一种手法中达到了非常高的性能。

`server.threaded.cc`使用多线程TcpServer,采用互斥锁保护共享数据.

`server_threaded_efficient.cc`借用智能指针`shared_ptr`实现了`copy-on-write`，可作为读写锁的更高效的替代。

`server_threaded_highperformance.cc`采用thread local变量，线程单例手法，实现多线程的高效转发。


前两种在最外层的Server对象中维护了一个总体Tcp连接的集合，这样每次收到了一个Tcp连接发送的信息，经过解码器解码出一条完整信息后，就遍历维护着所有Tcp连接的集合并将信息发送给其所有连接成员，这属于临界区，需要加锁，理论上只要没有用户进出聊天室，那么这里就不会成为临界区，因为没有数据结构的改变，但是为了实现用户的出入功能，该维护所有Tcp连接的集合需要动态的增加减少用户数，即涉及到了写操作，则读写互斥，成为了临界区，一般来说用户进出的频率远不如聊天室中发送信息的频率，因此读事件远远多于写事件，读写锁是最好的方案。但是写操作很容易产生饥饿，给聊天室想进入的人极大的延迟，体验很差，因为聊天室人一多，它就需要等待更久才能加入聊天室，因此采用copy-on-write.

前两种方法，维护着所有Tcp连接的集合，临界区过大。第三种只维护着I/O线程即eventloop的集合体，当收到一个聊天室消息，只需要将这个消息发送给每个eventLoop，而I/O线程数量最多不到10个，临界区非常小，然后可以绑定一个回调事件，即发送这个信息给自己eventloop中的所有Tcp连接成员。

但这里有一个问题，muduo框架eventloop是不会维护Tcp连接集合的，只有外层TcpServer会，因此采取thread local变量，每个I/O线程实现了这样一个线程独有的集合，但名字却相同以使代码可以复用逻辑更加清晰。


```c++
/* 线程单例类 */
template<typename T>
class ThreadLocalSingleton : noncopyable {
public:
    // 设置为delete，不准生成实例
    ThreadLocalSingleton() = delete;
    ~ThreadLocalSingleton() = delete;
	// 无实例时生成实例，存在实例时获得该实例(线程单例)的指针
    static T& instance() {
        if (!t_value_) {
            t_value_ = new T();
            deleter_.set(t_value_);
        }
        return *t_value_;
    }

    static T* pointer() {
        return t_value_;
    }

private:
    static void destructor(void* obj) {
        assert(obj == t_value_);
        typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
        T_must_be_complete_type dummy; (void) dummy;
        delete t_value_;
        t_value_ = 0;
    }
	...
	//静态变量，都是编译期就放到.data段了
    static __thread T* t_value_; //__thread 变量，线程独有
};
// 初始化为空指针
template<typename T>
__thread T* ThreadLocalSingleton<T>::t_value_ = 0;
```

## 5. **function/bind转换**

读muduo源码发现了一个bind funciton转换，百思不得其解。
```c++
//frist step
acceptChannel_.setReadCallback( bind(&Acceptor::handleRead, this) );
//对应的hanleRead声明
void Acceptor::handleRead()
//由上见，bind应该是产生了一个 void()的可调用对象，但是不可思议的是
typedef function<void(Timestamp)> ReadEventCallback;
void setReadCallback(const ReadEventCallback& cb)
{ readCallback_ = cb; }
//setReadCallb函数参数竟然是void(Timestamp)类型的函数对象，这其中发生了转换？
```
进一步我自己写了个测试
```c++
class A {
public:
    A(int a) {
        a_ = a;
    }
private:
    int a_;
};
typedef function<void(A)> voidACallback;

void fk(const voidACallback& h) {
    h( A(3)); 
}
void print() {
    cout<<"it's ok.";
}

int main() {
    voidACallback h = bind(&print);
    fk(h)
}
```
以上没有报任何错误！所以void()类型能转换成void(xxx)类型？amazing!

因为有以上转换，完全可以增加一个新的`typedef function<void(Timestamp)> ReadEventCallback;`
其参数传入poller返回时的时间辍，却不影响之前的所有设计函数，因为需要用Timestamp自然可以用，不需要的继续调用自己的空参数函数就好，
就是因为存在以上可调用对象的转换。
