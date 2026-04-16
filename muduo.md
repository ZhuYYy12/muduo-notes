## 典型的服务器结构
### 服务器该具有的三个特质：
1、高并发

2、高可用：若一个服务器出现了问题，另一个服务器可以立马接替他的工作

3、伸缩性：可以部署在同一个服务器下，也可以是独立的（跨机）

### 逐渐完善的过程
组成：网络I/O（epoll） + 服务器高性能编程 + 数据库

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774883448246-ef223927-f957-4a2f-b841-3fb714ab12e2.jpeg)



#### 优化数据库
数据库可能出现  并发问题（如数据库只能处理N个并发，但又10N同时对其操作）

     超时问题（如数据库只能处理每秒N个请求，10N个请求则需要10s，但应用服务器规定要在5s之内处理完毕，则会丢失5N个操作）

   处理服务器的并发问题：

##### 1、增加队列（队列服务+连接池）
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774883603275-a1e8ff38-60cf-4355-829b-7182929f37b6.jpeg)

##### 2、减少服务器的业务处理逻辑
##### 3、增加缓存 （将热点数据存入缓存）
     更新缓存的方法1：如图数据库更新后立马更新缓存中的数据

  方法2：缓存设定固定更新时间（实时性差）

当内存不够时，进行缓存换页，将不活跃的数据换出内存（先进先出、最近最少使用...）

分布式缓存：可用对数据一致性不高的数据库当缓存（redis、...）

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774884117208-a1fd1bb8-be55-45c9-b580-9306b0830dd8.jpeg)

##### 4、数据库负载均衡（replication机制）读写分离
    一般对数据库的读操作比写操作多，而写操作是对其进行加锁会妨碍读操作

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774885817272-e38cf869-5b01-4b52-93fe-15f3ee76a5af.jpeg)



##### 5、数据分区： 
#####    垂直分区 ： 将不同的表存储在不同的数据库
                     水平分区 ： 所有数据库的表结构相同 ， 多个数据库减少量



#### 优化业务逻辑
增加任务服务器：应用服务器被动接收业务：通过算法，将业务抛给业务处理较少的应用服务器

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774965083431-a7a3c0c6-eae3-4c5a-96e6-dbac9a889cb2.jpeg)

                        应用服务器主动接收业务：空闲的应用服务器取接业务

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774965083289-f815df3a-9029-4127-9294-7d8dbb89f88f.jpeg)



#### 最终结果：
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774965423825-358c8596-1b9e-4b93-8030-3fae69901b32.jpeg)



## 大型网站架构演变过程
### 本质
本质也是C/S

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774961858604-aeaea3f4-1e8a-48f2-88eb-e860ae1c1965.jpeg)



### 1、客户端增加缓存
客户端缓存需要存在本机上

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774963034156-29db37e9-0a30-4bb4-96e8-ec54379ed62f.jpeg)



### 2、前端增加缓存
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774964051239-f0454b73-c9b4-4394-90ea-d00a772ee3c1.jpeg)



### 3、本地数据库缓存
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774964203630-328adc3e-bccb-4435-a257-e50ce92b7f6c.jpeg)

EST：将相对静态的部分做缓存



### 4、前端、后端、数据库做负载均衡
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774964204029-49af6ab5-95a4-4fb1-b550-2fc988ab4169.jpeg)



### 5、多数据中心分布式存储与计算：最终结果
因为已经添加了大量的本地缓存，并发大大减少，所以此架构不需要队列

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1774964204130-567a6871-dd04-48e6-a7c2-3768f0ea2664.jpeg)

为什么使用分布式文件系统而不是操作系统自带的文件系统：自带的文件系统的块小，磁盘寻址时效率低

和缓存的区别： 缓存是内存加速临时用，分布式文件系统是磁盘集群永久存正本。  

## poll
### 基础原理
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775115731781-33b976a0-ed30-40fd-bb20-44c56f1cf889.jpeg)





```cpp
int poll(struct pollfd* fds, nfds_t nfds, int timeout);
```

pollfd为内置的结构体，用于存储被监听的fd数组

nfds为数组数量

timeout为 控制 `<font style="color:rgb(31, 35, 41);background-color:rgba(0, 0, 0, 0);">poll()</font>` 等待事件的时长，若为n（n>0) ，则等待n毫秒

       若<0 ，阻塞等待

       若=0 ， 非阻塞



```cpp
struct pollfd{
                int fd;
                short events;
                short revents;
}
```

fd为所监听的fd

events为所监听的事件类型

revents为实际事件类型



### 为什么fd要设置成非阻塞的？
若为阻塞： 没有accept->死等   没有read（）->死等   write()->死等

若为非阻塞：没有accept->返回-1   没有read（）->返回-1   write()->返回-1

### 使用实例
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775119835905-3053860b-eb16-472e-956a-188a6e75f61c.jpeg)





#### POLLOUT事件
+ 如果关注了pollout事件，因为fd是非阻塞，只要发送缓冲区是空的，就会触发该事件。而如果我们在accept到一个fd时就关注他的epollout事件，因没有要发送的东西，就会处于死循环忙等待
+ 解决方法：在write的时候判断是否能完全写完，若不能写完则将未发完的数据放入应用层缓冲区并关注pollout事件，当数据发送完后取消关注



#### 细节优化点：
+ 忽略pipe信号：client调用close后，server调用write，则在传输层会收到RST，server再次调用write，产生SIGPIPE信号，程序直接关闭
+ 减少服务器进入time_waite状态：server主动调用close后会进入time_waite状态，会在一定时间内保留一些内核信息，浪费资源。所以除了主动colse一些不活跃且不主动断开连接的client（造成资源浪费）外，尽可能使客户端断开连接
+ 使用空fd处理EMFILE问题



## epoll和poll的区别
+ epoll是一个实例对象，占有对应的空间（红黑树、就序列表、容器），只在epoll_ctl时拷贝一次，poll是一个临时函数，不断的调用并将fd拷贝进入内核
+ epoll返回的时就绪的事件，poll返回的时就绪个数，需要遍历寻找

## select/poll/epoll对比
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775295124107-24b3ec91-3aec-4041-9c7e-c2559b953ff3.png)

+ select有fd个数限制，只能管理32*32或32*64个（一个int是32位，电脑是32或64位的处理器）
+ poll没有fd个数限制（是链表管理），依旧是线性扫描，且poll是一个临时函数，会将大量的fd进行复制拷贝入内核，返回时也会拷贝回用户态，造成很大的开销，且每次遍历寻找活跃的fd会有大量的无意义遍历
+ epoll是一个实例对象，维护红黑树和就绪列表，相当于在内核态申请了一块空间，不需要大量的复制，且返回的时活跃的fd，减少了无意义的遍历
+ select和poll最大的区别：select用的是fd标识符，所以有个数限制，而poll是动态数组，所以没限制

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775296357927-193caf7b-8a1d-4ce3-b894-ace678de6dc1.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775296455241-5e8dec33-ce94-4af4-bc44-39f80b01a14d.png)

如果fd较少，且大量fd活跃，则调用callback的开销很大，epoll的效率可能就没有前两者高（ 代码极简、无复杂数据结构、无回调触发、无树管理  ）

## epoll
### 基础原理
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775281501790-963af949-4f73-4650-89d4-3567f1b15fab.jpeg)

#### epoll_create
返回epfd

```cpp
int epoll_create(int size);
int epoll_create1(int flag);
```

+ size : 需要管理fd的个数（只是参考，不会被编译器采用）
+ flag：支持EPOLL_CLOEXEC（在进程切换时（如exec操作）会自动关闭epfd，防止新进程占有epfd却并不适用，造成资源浪费）

#### epoll_ctl
返回1：成功    返回0：失败

```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event)
```

op: 操作类型 （`EPOLL_CTL_ADD`、`EPOLL_CTL_MOD`、`EPOLL_CTL_DEL`）

fd：需要监听的fd

event：需要关注的事件和事件数据



```cpp
struct epoll{
    unit32 event; //
    epoll_data_t data; //fd
}

typedef union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

event：

+ `EPOLLIN`：对应可读事件。
+ `EPOLLOUT`：对应可写事件。
+ `EPOLLET`：设置边缘触发。
+ `EPOLLERR`：对应错误事件。
+ `EPOLLHUP`：对应挂起事件。

unit: 大家共用一块内存（相当于上述类型中只能选择一个）

#### epoll_wait
返回就绪的fd

```cpp
int epoll_wait(int epfd,struct epoll_event *events, int maxevents, int timeout)
```

struct epoll_event *events:存就绪fd的数组

maxevents：数组大小

timeout：如poll的timeout

### epoll的水平触发（LT）和边缘触发（ET）
+ 水平触发：epollout事件：[同poll](#v9A22)
    - epollin触发条件：内核接收缓冲区不为空
    - epollout触发条件：内核中的发送缓冲区未满
+ 边缘触发：在accept后关注直接关注它的epollout事件和epollout事件 
    -  边缘触发的条件：状态发生改变时才会触发（如从无到有，从不可写到可写，从不可读到可读）
    -  只会触发一次，搭配非阻塞，一次性发送完毕/一次性读完（<font style="color:rgb(31, 35, 41);">只有内核发送缓冲区满了才会发送一次，若只有一半也不会触发）</font>
    - <font style="color:rgb(31, 35, 41);">不会触发EMFILE事件，因为死循环并没有改变状态</font>

问题：若前面的事件被漏处理，后续有了新事件也不会改变状态，也就不会被处理（（如accept没有没连接上，那么后面的都不会被连接上）

## 面向对象和基于对象的编程风格
### 面向对象
通过虚函数进行回调，需要复写纯虚函数，以基类指针指向派生类指针

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775482584906-3cf2ab0d-6db1-4da8-af53-99918097650f.jpeg)

### 基于对象
通过bind/function进行回调，无需复写虚函数，

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775482197533-44e572dd-2cad-4229-b6f1-14fdd24207a8.jpeg)

bind的作用：相当于胶水层，将用户所编写的函数转换适配epoll回调所需要的参数（解决如参数个数不同等等）

### 案例
#### thread_2_test中基于对象的回调逻辑
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775488048566-e0c2d64c-065c-4ceb-b9ec-f62fca114fc3.jpeg)

#### 小知识
+ pthread_create(线程id，线程属性，线程调用的函数，函数参数)；
    - 如果线程要调用的函数时成员函数，则必须使用static（因为成员函数隐式的包含this指针，而pthread_create无法识别this指针，则需要用static去掉他）
+ thread（线程调用的函数，函数参数）
    - thread时c++中封装的类，可以识别this指针
+ 需要将run函数设为私有（若是直接使用run函数，则并没有创建单独的线程，而是在主线程中运行）
+ 线程的生命周期和线程对象的生命周期不同，线程在运行完之后并不会释放，所以我们需要动态创建线程（new），并主动释放，或者使用智能指针对其进行管理（若不释放：如线程池，即使线程运行完毕也无法循环利用）

## base
muduo的base部分基本是对系统函数的封装，使得代码更加安全简洁，更像c++

### timestamp：时间戳
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1776145196969-5f2880df-436f-495d-bea2-967c968ab0fb.png)

+ 基类模板：less_than_comparable : 只用实现<操作，剩下的（>,>=,<=)自动补全
+ BOOST_STATIC_ASSERT： 编译期间的断言
+ PRid64 是一个字符串，统一32和64位计算机的longlong类型
+ muduo：：copyable  空基类，指明派生类是值类型（可以深拷贝）

### atomicintegerT:原子性操作
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775537769328-a00633b1-739b-4df7-b33b-2e6ab4522a2f.png)

+ 原子性操作：自增/比较和交换/赋值
+ 比较和赋值  如 bool k = a.compare_exchange_strong(b,c)   a是需要比较的值 b是预期值 c是新值

      若a=b，则a=c，k=true；

+ 实现无锁队列

```cpp
EnQueue{
    q->next =null;
    do{ p=tail }
        while(CAS(p->next,null,q) != true);//将q插入队尾
    CAS(tail,p,q);//将队尾改为q
}
        
```

+ q ： 是准备插入的数据   p: 是尾指针快照   CAS：原子性的比较和交换操作
    - 如果p->next == null 则返回 p->next=q , 且返回true
    - 如果tail == p , 则tail = q ,返回true
+ volative：不许与隐式转换（编译器的自动优化），每次都从内存中取值
+ implicit_cast<to* , from* > ：隐式转换
+ down_cast<form*> 向下转换 ：基类指针转为派生类指针

### exception
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775564433658-99848119-ed26-468b-9c67-08cc9205412a.png)

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775573299559-41a1369b-37be-4134-b9d4-593387c2af9a.jpeg)

### thread
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775569600668-ec7a5c7b-bac1-47bc-a932-b72edb3e5517.png)



![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775572053636-f335c8a8-c612-4528-8c15-c6e23c00ac48.jpeg)

+ 缓存tid（只能由系统调用获取到tid，减少系统开销
+ 判断是不是主线程： pid == tid
+ __thread  表示是线程内部的局部数据（该线程所独有的）
+ boost：：is_same（a，b） 判断a和b的类型是否一致
+ fork（） 克隆当前进程状态，创建一个子进程与父线程一起执行之后的操作（子进程与父进程资源不共享）
+ pthread_atfork:

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775570678464-a20a74ef-e985-41b2-91ae-c3dc8dffbf7a.png)

pthread_atfork的作用： 解决 “多线程 + fork” 带来的【锁状态不一致】导致的死锁问题。  

+ 若直接fork一个进程，可能会导致死锁（在1线程中对某个函数加锁正在处理数据，此时fork出了一个子进程，子进程也需要调用该函数，但此时该函数被锁住，则会造成死锁）
+ 若使用pthread_atfork，在fork之前父进程调用prepare（解锁），在fork成功后父进程调用parent（加锁）（因为原本的1线程处理数据是本该是锁着的，若解锁可能会脏读，则加锁使1线程继续处理数据），而后子进程调用child（初始化锁，使得子进程不发生脏读等）

### mutex
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775627084287-80ec76b9-29bb-4836-8969-9c00b726f6d8.png)

### 无界队列和有界队列
#### 无界队列
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775658836790-cf0e30c4-60a4-4c1b-89a4-7ac0c134b24e.png)

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775659598480-19dea7a6-0f04-47fe-a7ac-40967b25ec43.jpeg)

#### 有界队列
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775662192996-cda25a4e-9a68-4b26-843b-553f1763d51b.png)

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775662105116-bd23d611-0b2b-4f98-9844-9f6f93b206d3.jpeg)

### threadPool
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775802478405-de87eef0-e64a-4e06-8eba-0f093111d22e.png)

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775803034388-4738d163-9db0-48d6-b46a-4a32d28f51ce.jpeg)

### singleton<T>：单例类的封装
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775804250522-3583b6ca-3d14-4ca9-93d4-03043e2f203b.png)

单例的实现逻辑：

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775806516431-7456bcc3-fffd-4c13-abd9-fd54a20939fa.jpeg)

这样只会使得T类无法再通过传入Singleton获得初始化，但无法限制T类直接构造实例

所以T类的构造函数也需要设为私有，使其不能直接构造实例，而是只能通过传入Singleton进行构造

### threadLocal<T>：线程私有数据的封装
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775807700138-1b3be85c-2d2a-4f0a-8eee-562081e3ba08.png)

POD类型的数据可以用双下划线来表示是线程私有的，而其他的需要使用TSD（以及上述所封装的类）

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775807977869-e79762f2-8e0a-43dc-ad93-c5c468641a6d.png)

由key指向数据地址，获取实际数据



### ThreadLocalSingleton
因为TSD的key是全局唯一的，所以ThreadLocal需要是单例类

因threadLocal并不被需要，我们只需要threadlocalSingleton，所以单独封装它的单例类，而不是直接传给singleton管理

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1775808622838-e5065bde-d01f-4f8e-be9c-fb731afdc44e.png)

## net
### 类图
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775884033715-40759f95-0bcb-4d97-a787-7555a8c3caa5.jpeg)

### 顺序图
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775831898834-8eba417d-aad4-4f2f-aaa3-96c3a72c36b1.jpeg)

#### channel中event的读写辨别：
POLLIN与POLLOUT本质上是标识符，前者为0001，后者为0100（这样使我们通过第0为是否为1来判断是否为读时间，第二位是否为1来判断是否为写事件），int event，先初始话为0，再与POLLIN，POLLOUT进行 |=，获得新的标识符（第0位为1表示读，第1位为1表示紧急读事件，第2位为1表示写，第三位为1表示err）。将其注监听，若监听到有事件到来，则返回revent（本质上也是标识符），再通过revent &= POLLIN/POLLOUT来判断是否有读写

+ 从第0为到第六为的值为1所表示的事件：读，紧急，写，err，连接出错，连接断开

### eventLoop
+ eventloop其实本质上是一个线程+无限循环
+ eventloop只负责 wakeupChannel、timerQueueChannel 这两个内置 Channel 的生命周期，而不负责其他的channel
    - 因为唤醒io和定时处理事件属于io线程的底层处理逻辑，而其他如sockedfd，listenfd处理的业务逻辑
    - io逻辑和业务逻辑的区别：前者是一开始便要存在的，且全程存在，而后者是动态的，随时可能销毁

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1775829902378-d24ddfda-000b-4c86-8aa2-29c3c0251ea5.jpeg)



### 定时器
#### linux系统所提供的三个函数
![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1776144306997-5494f768-bd87-49a2-88e5-452ca2337ad1.jpeg)

linux系统所提供的关于定时器的三个函数

```cpp
// 1. 创建定时器 fd
int timerfd_create(int clockid, int flags); 

// 2. 启动 / 停止 / 设置定时器
int timerfd_settime(int fd, int flags, 
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);

// 3. 获取当前定时器的剩余时间
int timerfd_gettime(int fd, struct itimerspec *curr_value);
```

+ **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">clockid</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：时钟类型</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">CLOCK_MONOTONIC</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">单调递增时钟</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（不会被修改时间影响）</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">CLOCK_REALTIME</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：真实世界时间（会被 NTP 修改，不推荐）</font>
+ **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">flags</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：文件描述符属性</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">TFD_NONBLOCK</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：非阻塞</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">TFD_CLOEXEC</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：fork 后自动关闭 fd</font>
    - <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">muduo 用的是：</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">TFD_NONBLOCK | TFD_CLOEXEC</font>**



+ **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">flags</font>**
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">0</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">相对时间</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（从现在开始多久后超时 → muduo 用这个）</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">TFD_TIMER_ABSTIME</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：绝对时间（到某个时间点超时）</font>
    - <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">*new_value：设置超时时间，包含两个成员</font>
    - <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">*old_value：NULL</font>

```cpp
struct timespec {
    time_t tv_sec;  // 秒
    long   tv_nsec; // 纳秒
};  
```



+ **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">curr_value</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">输出参数，返回：</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">it_value</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：距离下次超时还剩多久</font>
    - `<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">it_interval</font>`<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：重复间隔</font>

#### addtimer和loop不一定在同一个线程
<font style="color:rgba(0, 0, 0, 0.85);background-color:rgba(0, 0, 0, 0.04);">addtimer的作用是创建一个定时器，而定时器是要在特定时间执行读写操作，而读写操作要在loop中进行，但addtimer随处可以调用，不一定实在loop所处的线程，如不在，则要将addtimer传给一个loop，即放入loop线程中的队列,而若在，则立即执行</font>

#### <font style="color:rgba(0, 0, 0, 0.85);background-color:rgba(0, 0, 0, 0.04);">唤醒线程</font>
如前面所说，addtimer和loop可能不在一个线程，所以将其投入loop所在线程的队列中后需要唤醒loop所在的线程，即io线程（因为io线程正在whlie中阻塞），使用wakeup唤醒

    - 在eventloop构造时就将wakeupfd注册进poller中监听，其他线程调用wakeup函数会向wakeupfd中写入数据，使得wakeupfd可读，以此唤醒io线程处理wakeupfd，便也能处理队列中的事件
    - <!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/66278721/1776322799728-6bd83939-eac4-4f06-86d0-9d10d98d4089.png)

执行流程

![画板](https://cdn.nlark.com/yuque/0/2026/jpeg/66278721/1776324102678-31378c43-c732-4ae4-aa0e-05ba51077c13.jpeg)



### eventLoopThread
eventLoop和eventLoopThread的区别：后者还包含了线程的创建，为eventLoopthreadPool做准备

### socked
#### 网机地址和通用地址
socked所bind的地址是通用地址（sockaddr），但通用地址需要自己进行拼出二进制写法，所以使用网机地址（sockaddr_in），然后将其强转为通用地址再bind

#### read和readv
 read 单块内存、多次调用、需要手动拼接；readv 多块分散内存、一次调用、无需拼凑、高性能。  

## cmake
编写规则

```cpp
//版本
cmake_minimum_required(VERSION 3.10)
//项目名称
project(muduo_test)
//c++11标准
set(CMAKE_CXX_STANDARD 11)

//头文件和lib库的路径
set(MUDUO_INCLUDE /home/ssmio/muduo)
set(MUDUO_LIBPATH /home/ssmio/build/release-cpp11/lib)

# 头文件路径
include_directories(${MUDUO_INCLUDE})

# 库文件路径
link_directories(${MUDUO_LIBPATH})

# 生成可执行文件
add_executable(test test.cpp)

# 链接静态库 + 线程库
target_link_libraries(test
    ${MUDUO_LIBPATH}/libmuduo_net.a
    ${MUDUO_LIBPATH}/libmuduo_base.a
    pthread
)
```



