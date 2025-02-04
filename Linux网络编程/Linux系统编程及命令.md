
# linux系统编程及基本命令

## 系统编程
> * [按下开机键Linux发生了什么](#按下开机键Linux发生了什么)
> * [进程退出方式及区别](#进程退出方式及区别)
> * [回收进程资源的方式和区别](#回收进程资源的方式和区别)
> * [守护进程](#守护进程)
> * [线程退出方式与线程回收](#线程退出方式与线程回收)
> * [共享内存](#共享内存)
> * [信号量](#进程中的信号量)
> * [信号通知进程](#信号通知进程)
> * [valgrind检查内存泄漏](#valgrind检查内存泄漏)
> * [程序从main函数开始吗](#程序从main函数开始吗)

## 基本命令
> * [Linux基本目录结构](#Linux基本目录结构)
> * [文件操作命令](#文件操作命令)
> * [磁盘及内存命令](#磁盘及内存命令)
> * [进程命令](#进程命令)
> * [网络命令](#网络命令)
> * [字符处理命令](#字符处理命令)
> * [调试命令](#调试命令)
> * [文本处理工具](#文本处理工具)

## [进程退出方式及区别](https://www.cnblogs.com/xiaojianliu/p/8473083.html)
不管哪种退出方式，系统最终都会执行内核的同一代码，这段代码用来关闭进程打开的文件描述符，释放它占用的内存和其他资源

* 退出
    * 正常退出
        * main函数调用return
        * 调用exit()函数
        * 调用_exit()函数
    * 异常退出
        * 调用abort函数
        * 进程收到某个信号，该信号使程序终止

* 已结束进程的状态
    * shell执行 echo $?，保存最近一次运行的进程的返回值
        * 程序中main函数运行结束，保存main函数的返回值
        * 程序调用exit函数结束运行，保存exit函数的参数
        * 程序异常退出，保存异常出错的错误号

* 区别
    * exit和return的区别
        * exit是函数，有参数，exit执行完会把控制权交给系统，exit(0)表示正常终止，其他值表示有错误发生
        * return是函数执行完后的返回，return执行完后把控制权交给调用函数
    * exit和abort的区别
        * exit是正常终止进程
        * abort是异常终止进程
    * exit和_exit函数的区别
        * exit在头文件stdlib.h中声明，_exit是在头文件unistd.h中声明
        * exit是 _exit之上的一个封装， **exit先刷新流数据** ，再调用 _exit函数
            * _exit会关闭进程打开的文件描述符，清理内存，不会刷新流数据
            * linux的库函数，有一种“缓冲IO”的操作，对应每一个打开的文件，在内存中有一片缓冲区，每次读文件，会连续读出若干条记录，下次再读文件的时候，直接从内存的缓冲区中读；同样写文件也是先写入缓冲区，满足一定条件才将缓冲区的内容一次性写入文件。具体可以看printf和write的区别，及行缓冲和全缓冲
            * exit先刷新流数据，将文件缓冲区的内容写回文件，可以保证数据的完整性，_exit会将数据直接丢失

## 回收进程资源的方式和区别
* init进程（进程号为1）会周期性的调用wait系统调用来清除各个僵尸进程
* 僵尸进程：进程终止，父进程尚未回收，子进程残留资源（PCB）存放于内核中，变成僵尸进程。
* wait
    * wait函数的功能
        * 阻塞等待子进程退出
        * 回收子进程残留资源
        * 获取子进程结束状态（退出原因）
    * `pid_t wait (int *status)` status表示子进程的退出状态，成功返回值为子进程进程号，失败为-1
    * 进程一旦调用wait函数，立即阻塞自己， 判断当前进程的某个子进程是否变成僵尸进程
        * 若存在则收集子进程的信息，将它彻底销毁然后返回
        * 若没有，则会一直阻塞，直到出现一个
* waitpid
    * waitpid相当于wait函数的封装，多了两个由用户控制的参数pid和options，可以自定义回收的子进程进程号，并设置是否阻塞
    * `pid_t waitpid(pid_t pid, int * status, int options)`
    * pid
        * pid < -1，等待进程组ID为pid绝对值的任何子进程
        * pid = -1，等待任何子进程，相当于wait
        * pid = 0，等待进程组ID与目前进程相同的任何子进程
        * pid > 0，等待子进程ID为pid的进程
    * options
        * 0，与wait相同，也会阻塞
        * WNOHANG，不会阻塞，如果当前没有可回收的子进程，立即返回0
    * 【注意】：一次wait和waitpid调用只能清理掉一个子进程

## 守护进程
Daemon进程，守护进程，是脱离终端并在后台运行的进程，脱离终端是为了避免进程运行过程中的信息显示在任何终端上，另外进程越不会被任何终端产生的终端信息所打断。

Linux后台的一些系统服务进程，没有控制终端，不能直接与用户交互。不受用户登录、注销的影响，一直在运行着，它们都是守护进程。如：预读入缓输出机制的实现；ftp服务器；nfs服务器等。

### 终端，进程组，会话期，setsid函数
* 终端
    * linux中，每一个系统与用户进行交流的界面称为终端。
    * 每一个从此终端开始运行的进程都依附于此终端，这个终端称为这些进程的控制终端，当控制终端被关闭时，相应的进程会自动关闭。
    * 守护进程可以突破这种限制，从运行开始执行，整个系统关闭才退出

* 进程组
    * 进程组是一个或多个进程的集合
    * 进程组由进程组ID唯一标识，该进程组中的进程除了进程号之外，还有进程组ID的属性
    * 进程组中第一个进程默认为进程组长，其进程号 = 进程组ID

* 会话期
    * 会话期是一个或多个进程组的集合
    * **进程组的组长不能创建会话**，只有组员才能创建会话，因此都是子进程创建会话实现守护进程
    * 一个会话期开始于用户登录，终止于用户退出，在此期间该用户运行的所有进程都属于这个会话期

* setsid函数
    * 用于创建新会话，将调用setsid的进程担任会话期的会长
    * 让进程摆脱原会话的控制
    * 让进程摆脱原进程组的控制
    * 让进程摆脱原控制终端的控制

* 为什么创建守护进程要调用setsid？
    * 一般子进程实现守护进程，子进程fork于父进程，子进程全盘拷贝了父进程的会话期、进程组、控制终端
    * 虽然父进程退出了，但子进程的各个属性都没变，不算真正意义的独立

### 创建守护进程
* 创建子进程，父进程退出
    * 所有工作在子进程中进行，形式上脱离了控制终端
* 子进程中调用setsid创建新会话
    * 使子进程完全独立出来，脱离控制
* 切换工作目录，一般切换为根目录/
    * chdir函数
    * 从父进程继承来的工作目录可能是可卸载的文件系统，当进程没有结束时，其工作目录是不能被卸载的。若不修改工作目录，该文件系统不能卸载 
* 重设文件权限掩码umask
    * `umask`是用来指定"`目前用户在新建文件或者目录时候的权限默认值`"
    * 创建文件或目录时，指定权限mode & ~umask 才是最终权限
    * 由于从父进程继承来的文件权限掩码会屏蔽掉文件权限中的对应位，这给该子进程使用文件带来麻烦
    * 增加守护进程灵活性
    * 通过umask(0),表示用户、用户组和其他用户都有可读可写可执行权限
* 关闭文件描述符
    * 子进程会从父进程继承一些打开的文件描述符，但这些可能永远不会被守护进程使用，但他们一样消耗系统资源，而且会导致所在文件系统无法结束
    * 守护进程已经脱离了所属的控制终端，因此终端的输入，输出和报错也失去了存在的价值。可以使用dup函数将标准输入、输出和错误输出重定向到/dev/null设备上（/dev/null是一个空设备，向其写入数据不会有任何输出）。
    * 可以遍历MAXFILE，然后close所有文件描述符
* 如果想结束守护进程，直接kill -9 pid即可

## 线程退出方式与线程回收
### 线程退出方式
注意：不能使用exit，exit表示退出整个进程

* pthread_exit
    * `int pthread_exit(void *retval);` 
    * 在任何线程中使用，使该线程直接退出
    * 主线程退出而不影响其他线程，只能使用这种方式
* return
    * 子线程中可以使用，主线程不能使用，主线程代表退出整个进程

### 线程回收
* pthread_join
    * `int pthread_join(pthread_t thread, void **retval) `
    * 用来等待一个线程的结束，并回收该线程的资源
    * 一般是主线程调用，用来等待子线程退出，是阻塞的

### 线程分离
* pthread_detach
    * `int pthread_detach(pthread_t thread)` 
    * 分离已经创建的线程，将主线程与子线程分离，子线程结束后，资源自动回收。 
    * 状态分离后，该线程的结束状态不能被该进程中的其他线程得到，因此pthread_join不能调用，否则会出错

## [共享内存](https://blog.csdn.net/hj605635529/article/details/73163513)
共享内存有两种shm和mmap

* IPC通信System V版本的共享内存shm
* 存储映射I/O（mmap函数） 

Linux上使用 `ipcs -m` 查看共享内存

### shm
* 原理
    
    * 多个进程的地址空间映射到同一个物理内存，不同进程可以将同一段共享的内存连接到自己的地址空间中，从而所有进程都可以访问共享内存中的地址
* API
    * `int shmget(key_t key, size_t size, int shmflg);`
        * 功能：在物理内存创建一个共享内存
        * 返回值：成功返回共享内存的标识符（与key相关），失败返回-1
        * 参数
        	* key是一个非0整数，标识共享内存键值，（多个进程要知道他们共享的内存是谁，就是靠key来判断，因此多个进程调用这个函数时应指定key一样）
    	* size表示以字节为单位指定需要的共享内存的容量
        	* shmflag是权限标志位，比如 0666 | IPC_CREAT，其中有效的包括两个：
        		* IPC_CREAT：如果共享内存不存在，则创建一个共享内存，否则获得该共享内存
        		* IPC_EXCL：只有在共享内存不存在的时候，新的共享内存才建立，否则就产生错误
        
    * `void *shmat(int shmid, const void shmaddr,int shmflg);`
        * 功能：进程在使用共享内存之前，需要先将共享内存映射到自己的虚拟地址空间。
        
        * 返回值：成功返回共享内存的起始地址，失败返回-1
        
    * 参数：
        
        	* shmid时共享内存的id，即shmget的返回值
        
        	* shmaddr是共享内u才能在本进程内的起始地址，置为NULL，让系统选择一个合适的地址空间进行挂接
        	* shmflg表示本进程对该内存的操作模式，如果是SHM_RDONLY的话就是只读模式，一般都是取0。
        
    * `void *shmdt(const void* shmaddr);`
        
    * 将共享内存从当前进程中分离，断开用户级页表到共享内存的那根箭头。
        
    * `int shmctl(int shmid, int cmd, struct shmid_ds* buf);`
        * 释放物理内存中的那块共享内存
        * cmd取IPC_RMID表示删除这块共享内存

### mmap
* 原理
    * mmap是映射磁盘上的一个文件，每个进程在自己的逻辑地址空间中开辟一块空间对磁盘上的文件进行映射
    * 内存映射的过程中，并没有实际的数据拷贝，文件没有被载入内存
    * mmap返回一个指针ptr，它指向进程逻辑地址空间中的一个地址，通过ptr就能够操作文件。但是ptr所指向的是一个逻辑地址，要操作其中的数据，必须通过MMU将逻辑地址转换成物理地址，建立内存映射并没有实际拷贝数据，这时，将产生一个缺页中断，会通过mmap()建立的映射关系，从硬盘上将文件读取到物理内存中

* 效率
    * read()是系统调用，其中进行了数据拷贝，它首先将文件内容从硬盘拷贝到内核空间的一个缓冲区，然后再将这些数据拷贝到用户空间，在这个过程中，实际上完成了两次数据拷贝
    * mmap()也是系统调用，mmap()中没有进行数据拷贝，真正的数据拷贝是在缺页中断处理时进行的，由于mmap()将文件直接映射到用户空间，所以中断处理函数根据这个映射关系，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝

* 映射文件
    * 普通文件
        * open系统调用打开一个文件，然后进行mmap操作，得到共享内存，这种方式适用于任何进程之间。 
    * 匿名映射
        * 调用 mmap 时，在参数 flags 中指定 MAP_ANONYMOUS 标志位，并且将参数 fd 指定为 -1 ,用于父子进程之间

### 不同进程访问共享内存
* shm
    * 不同进程通过shmget->shmat函数，将共享内存连接到自己的虚拟内存地址
* mmap
    * 不同进程通过mmap函数创建映射区，将自己的内存虚拟地址映射到磁盘的文件上

### 程序异常退出，共享内存会释放吗？
* 不会
    * Linux中通过API函数shmget创建的共享内存一般都是在程序中使用shmctl来释放的，但是有时为了调试程序，开发人员可能通过Ctrl + C等方式发送中断信号来结束程序，此时程序申请的共享内存就不能得到释放，当然如果程序没有改动的话，重新运行程序时仍然会使用上次申请的共享内存，但是如果我们修改了程序，由于共享内存的大小不一致等原因会导致程序申请共享内存错误。
* 如何释放
    * 如果总是通过Crtl+C来结束的话，可以做一个信号处理器，当接收到这个信号的时候，先释放共享内存，然后退出程序。
    * 不管你以什么方式结束程序，如果共享内存还是得不到释放，那么可以通过linux命令ipcrm shm shmid来释放，在使用该命令之前可以通过ipcs -m命令来查看共享内存。 

### 两者的区别
* 作用
    * mmap系统调用并不完全是为了共享内存来设计的，它本身提供了不同于一般对普通文件的访问的方式，进程可以像读写内存一样对普通文件进行操作
    * IPC的共享内存shm是纯粹为了共享。
* 映射位置
    * mmap是在磁盘上建立一个文件，每个进程地址空间中开辟出一块空间对磁盘上的文件进行映射。
    * shm每个进程映射到同一块物理内存，shm保存在物理内存，这样读写的速度要比磁盘要快，但是存储量不是特别大
* 内容丢失
    * 进程挂了重启不丢失内容，二者都可以做到 
    * 机器挂了重启，mmap把文件存在磁盘上，可以不丢失内容（文件内保存了OS同步过的映像），而 shmget 会丢失 

## [信号量](https://blog.csdn.net/qq_38813056/article/details/85706006)
[参考资料](https://blog.csdn.net/mijichui2153/article/details/84930553)
信号量广泛用于进程或线程间的同步和互斥，信号量本质上是一个非负的整数计数器，它被用来控制对公共资源的访问，分为两种POSIX信号量和SystemV信号量。

1. 二值信号量(binary semaphore)：其值或为0或为1的信号量。这与互斥锁类似，若资源被锁住则信号量值为0，若资源可用则信号量值为1。
2. 计数信号量(counting semaphore)：其值为0和某个限制值(对于Posix信号量，该值必须至少为32767)之间的信号量。该信号量的值就是可用资源数。
3. 计数信号量集（set of counting semaphore）：一个或者多个信号量（构成一个集合），其中每个都是计数信号量。每个集合的信号量数都存在一个限制，一般在25个的数量级。

==Posix信号量==是基于内存的，即信号量值是放在共享内存中的，它是由可能与文件系统中的路径名对应的名字来标识的。

==System v信号量==测试基于内核的，它放在内核里面，相同点都是它们都可以用于进程或者线程间的同步。

**当我们讨论“System v信号量”时，所指的是计数信号量集，而当我们谈论“Posix 信号量”时，所指的是单个计数信号量。**

### POSIX信号量
* ==有名信号量==，其值保存在文件中，可以用于进程间也可以用于线程间同步

    * 有名信号量的特点是把信号量的值保存在文件中。

        这决定了它的用途非常广：既可以用于线程，也可以用于相关进程间，甚至是不相关进程。

    - **有名信号量能在进程间共享的原因**：由于有名信号量的值是保存在文件中的，所以对于相关进程来说，子进程是继承了父进程的文件描述符，那么子进程所继承的文件描述符所指向的文件是和父进程一样的，当然文件里面保存的有名信号量值就共享了。
    - 有名信号量在使用的时候，和无名信号量共享sem_wait和sem_post函数。区别是有名信号量使用sem_open代替sem_init，另外在结束的时候要像关闭文件一样去关闭这个有名信号量。

* ==无名信号量==，其值保存在内存中
    
    * 无名信号量的创建就像声明一般的变量一样简单，如：sem_t sem_id。然后再初始化该无名信号量，之后就可放心使用了。
    * 无名信号量常用于多线程间的同步，同时也用于相关进程间的同步。也就是说，无名信号量必须是多个进程（线程）的共享变量，无名信号量要保护的变量也必须是多个进程（线程）的共享变量，这两个条件是缺一不可的。
    *  常见的无名信号量相关函数：sem_destroy、sem_wait、sem_post
    * 无名信号量也可以用于进程间同步，只需要将sem_init的参数pshared设置为非0即可

### SystemV信号量，用于进程间同步
* 进程中使用共享内存实现进程间通信，但他并不是线程安全的，需要通过信号量进行同步
* 一般说的systemV信号量是计量信号集，可以使用多个信号量进行同步

* API
    * `int semget(key_t key, int nsems, int semflag);` 

## [信号通知进程](http://www.360doc.com/content/16/0804/10/30953065_580685165.shtml)
### 信号的使用
用kill函数发送信号，在接收进程里，通过signal或者signalaction函数调用sighandler，来启动对应的函数处理信号消息。

* 发送信号
    * raise，向本身发送信号
    * kill，向指定进程发送信号
        * pid > 0 ：向进程号为pid的进程发送信号
        * pid = 0 ：向当前进程所在的进程组发送信号
        * pid = -1 ：向所有进程(除PID=1外)发送信号(权限范围内)
        * pid < -1 ：向进程组号为-pid的所有进程发送信号 
* 自定义信号动作/注册捕捉函数
    * signal
        * `typedef void (*sighandler_t)(int);`
        * `sighandler_t signal(int signum, sighandler_t handler);` 
        * signal里面需要设置捕捉的信号signum、自定义回调函数handler
    * sigaction
        * 同样需要设置捕捉信号和回调处理函数 

### 信号处理机制
每个进程之中，都有存着一个表，里面存着每种信号所代表的含义，内核通过设置表项中每一个位来

* 信号的接收
    * 接收信号的任务是由`内核`代理的，当内核接收到信号后，会将其放到对应进程的信号队列中，同时向进程发送一个中断，使其陷入内核态。注意，此时信号还只是在队列中，对进程来说暂时是不知道有信号到来的。

* 信号的检测
    * 进程陷入内核态后，有两种场景会对信号进行检测：
        * 进程从内核态返回到用户态前进行信号检测
        * 进程在内核态中，从睡眠状态被唤醒的时候进行信号检测
    * 当发现有新信号时，便会进入下一步，信号的处理。

* 信号的处理
    * ( **内核** )信号处理函数是运行在用户态的，调用处理函数前，内核会将当前内核栈的内容备份拷贝到用户栈上，并且修改指令寄存器（eip）将其指向信号处理函数。
    * ( **用户** )接下来进程返回到用户态中，执行相应的信号处理函数。
    * ( **内核** )信号处理函数执行完成后，还需要返回内核态，检查是否还有其它信号未处理。
    * ( **用户** )如果所有信号都处理完成，就会将内核栈恢复（从用户栈的备份拷贝回来），同时恢复指令寄存器（eip）将其指向中断前的运行位置，最后回到用户态继续执行进程。

至此，一个完整的信号处理流程便结束了，如果同时有多个信号到达，上面的处理流程会在第2步和第3步骤间重复进行。

### 信号通知进程，为什么通过内核转发？
* 之所以要通过内核来转发，这样做的目的应该也是为了对进程的管理和安全因素考虑。
* 因为在这些信号当中，SIGSTOP和SIGKILL这两个信号是可以将接收此信号的进程停掉的，而这类信号，肯定是需要有权限才可以发出的，不能够随便哪个程序都可以随便停掉别的进程。

### 信号处理示例
* A，B两个进程，A进程发送信号给B进程，信号并不是直接从进程A发送给进程B，而是要通过内核来进行转发。
* A进程发送的信号消息，由内核对B进程相应的表项进行设置。
    * 内核接受到这个信号消息后，会先检查A进程是否有权限对B进程的信号表对应的项进行设置
        * 如果可以，就会对B进程的信号表进行设置
        * 如果不可以，就忽略
        * 信号处理有个特点，就是没有排队的机制，也就是说某个信号被设置之后，如果B进程还没有来及进行响应，那么如果后续第二个同样的信号消息过来，就会被阻塞掉，也就是丢弃。
    * 内核对B进程信号设置完成后，就会发送中断请求给B进程，这样B进程就进入到内核态
    * 进程B根据那个信号表，查找对应的此信号的处理函数，保护现场，跳回到用户态执行信号处理函数，处理完成后，再次返回到内核态，再次保护现场，然后再次返回用户态，从中断位置开始继续执行。
    * 保护现场是在用户态和内核态之间跳转的时候，对堆栈现场的压栈保存。

## valgrind检查内存泄漏
* 检查内存泄露原理
    * 检测内存泄漏的关键是要能截获住对分配内存和释放内存的函数的调用。
    * 截获住这两个函数，我们就能跟踪每一块内存的生命周期，比如，每当成功的分配一块内存后，就把它的指针加入一个全局的list中；每当释放一块内存，再把它的指针从list中删除。这样，当程序结束的时候，list中剩余的指针就是指向那些没有被释放的内存 
* 最常用的是memcheck，用于发现绝大多数的内存错误使用情况
    * 使用未初始化的内存
    * 使用已经释放的内存
    * 内存访问越界
* memcheck原理
    * Memcheck 能够检测出内存问题，关键在于其建立了两个全局表。
        * Valid-Value 表：对于进程的整个地址空间中的每一个字节(byte)，都有与之对应的 8 个 bits；对于 CPU 的每个寄存器，也有一个与之对应的 bit 向量。这些 bits 负责记录该字节或者寄存器值是否具有有效的、已初始化的值。
        * Valid-Address 表：对于进程整个地址空间中的每一个字节(byte)，还有与之对应的 1 个 bit，负责记录该地址是否能够被读写。
    * 检测原理：
        * 当要读写内存中某个字节时，首先检查这个字节对应的 A bit。如果该A bit显示该位置是无效位置，**memcheck则报告读写错误**。
        * 内核（core）类似于一个虚拟的 CPU 环境，这样当内存中的某个字节被加载到真实的 CPU 中时，该字节对应的 V bit 也被加载到虚拟的 CPU 环境中。一旦寄存器中的值，被用来产生内存地址，或者该值能够影响程序输出，则 memcheck 会检查对应的V bits，如果该值尚未初始化，则会**报告使用未初始化内存错误**。 

    
    
## 程序从main函数开始吗
* 程序在main函数开始之前，已经完成了全局变量的初始化，堆栈初始化和系统I/O
* main之前完成全局变量的构造，在main之后完成全局变量的析构

```c++
#include <bits/stdc++.h>
using namespace std;

class T
{
public:
	T() {
		cout << "construct..." << endl;
	}
	~T() {
		cout << "destruct..." << endl;
	}
};
T t;

int main()
{
	cout << "main begin..." << endl;
	return 0;
}

/*
# 输出
construct...
main begin...
destruct...
*/
```

* 另外atexit函数还可以在main函数之后运行，它接受一个函数指针作为参数
	* atexit函数是一个特殊的函数，它是在正常程序退出时调用的函数，我们把他叫为==登记函数==，⼀个进程可以登记若⼲个函数，这些函数由exit⾃动调⽤，这些函数被称为**终⽌处理函数**， atexit函数可以登记这些函数。 exit调⽤终⽌处理函数的顺序和atexit登记的顺序相反（网上很多说造成顺序相反的原因是参数压栈造成的，参数的压栈是先进后出，和函数的栈帧相同），如果⼀个函数被多次登记，也会被多次调⽤。

```c++
#include <stdio.h> 
#include <stdlib.h>
void func1() { 
    printf("The process is done...\n"); 
} 
void func2() { 
    printf("Clean up the processing\n"); 
} 
void func3() { 
    printf("Exit sucessful..\n"); 
} 
int main() 
{ 
    atexit(func1); 
    atexit(func2); 
    atexit(func3); 
    return 0; 
} 
/*
# 输出
Exit sucessful..
Clean up the processing
The process is done...
*/
```

## Linux基本目录结构
* /bin，binaries存放二进制可执行文件
* /usr，unix shared resources用于存放共享的系统资源
* /sbin，super user binaries存放二进制可执行文件，只有root才能访问
* /etc，etcetera存放系统的配置文件
* /boot，存放启动linux和引导文件的目录
* /lib，存放着系统最基本的动态连接共享库
* /dev，存放linux的设备文件，比如显示器，键盘等
* /mnt，用户可以在这个目录下挂在其他临时文件系统
* /media，linux系统会自动识别一些设备，例如U盘、光驱等等，linux会把识别的设备挂载到这个目录下
* [/proc](https://www.cnblogs.com/zydev/p/8728992.html)，proc被称为虚拟文件系统，它是一个控制中心，可以通过更改其中某些文件改变内核运行状态，它也是内核提空给我们的查询中心，用户可以通过它查看系统硬件及当前运行的进程信息。
    * /proc/loadavg，前三列分别保存最近1分钟，5分钟，及15分钟的平均负载。
    * /proc/meminfo，当前内存使用信息
    * /proc/cpuinfo ，       CPU的详细信息
    * /proc/diskstats，    磁盘I/O统计信息列表
    * /proc/net/dev  ，    网络流入流出统计信息
    * /proc/filesystems，  支持的文件系统
    * /proc/cmdline ，     启动时传递至内核的启动参数，通常由grub进行传递
    * /proc/mounts ，    系统当前挂在的文件系统
    * /proc/uptime ，   系统运行时间
    * /poc/version ，    当前运行的内核版本号等信息 
* /opt，这是给主机额外安装软件所摆放的目录。比如你安装一个ORACLE数据库则就可以放到这个目录下。默认是空的。

## 文件操作命令
* ls
    * `ls -lrt` 递归显示文件的详细信息并按照时间排序 
* tail
    * `tail -n 100` 显示文件尾，指定显示行数，默认10行
* chmod
    * `chmod 777 filename` 修改文件的权限为用户，用户组，其他人有所有权限 
* rm
    * `rm -r` 递归删除子目录
    * `rm -f` 强制删除
* vim的三种模式
    * 命令模式(一般模式，通过yy进行赋值)
    * 编辑模式，通过i或者a
    * 末行模式，冒号


## 磁盘及内存命令
* 文件大小和占用空间大小是不一样的，因为要对齐

* 显示每个文件和目录的磁盘使用空间
	* (disk used) `du -h`
* 显示磁盘分区上可以使用的磁盘空间
	* (disk free) `df -h`
* 显示内存使用情况
    * [free -m](https://www.cnblogs.com/ultranms/p/9254160.html)
    * Mem是物理内存的使用情况
    * Swap是交换空间的使用情况
    * total是物理内存和交换空间的总大小
    * used是物理内存和交换空间已经被使用的大小
    * free是物理内存和交换空间可用空间（从内核和系统的角度看，真正尚未被使用的物理内存数量）
    * shared 列显示被共享使用的物理内存大小。
    * buff/cache 列显示被 buffer 和 cache 使用的物理内存大小（其实是内存为缓存磁盘数据设置的缓冲区）。
    * available 列显示还可以被应用程序使用的物理内存大小，当应用程序需要内存时，如果没有足够的 free 内存可以用，内核就会从 buffer 和 cache 中回收内存来满足应用程序的请求，理想来说available  = free + buffer + cache
	
## 进程命令
* ps
    * 当前运行的进程的快照，指定ps命令的那个时刻的那些进程
    * `ps -aux` 查看所有的在内存中的进程信息
    * `ps -ajx` 查看进程组相关信息，可以追踪进程之间的血缘关系
    * `ps -ef` 线城市所有进程信息，并显示程序间的关系
    * `ps -u username` 显示指定用户username信息
* top
    * 实时显示系统中各个进程的资源占用情况，按"q"退出top命令
    * `top -H -p pid`显示对应pid的所有线程资源使用情况
    * `load average` 表示系统最近1min,5min,15min的平均负载，越大表示负载越来越小
    * %MEM物理内存占用比
    * Cpu(s)和%cpu
        * Cpu(s)表示的是所有用户进程占用整个cpu的平均值
        * %CPU显示的是进程占用一个核的百分比，而不是整个cpu（8核）的百分比，有时候可能大于100，那是因为该进程启用了多线程占用了多个核心，所以有时候我们看该值得时候会超过100%，但不会超过总核数*100。
* kill
    * 杀掉进程
    * `kill -9 pid`杀掉指定进程
* lsof
    * 列出当前系统打开文件的工具
    * `lsof -i :8600` 查看8600端口的运行情况
    * `lsof -u username` 查看username打开的文件
    * `lsof -c string` 查看包含指定字符的进程所打开的文件

## 网络命令
* netstat
    * 用于显示与IP、TCP、UDP、和ICMP协议相关的统计数据，用于检验本机各端口的网络连接情况
    
    * `-a` 列出所有端口
    
    * `-p` 显示出进程和PID
    
    * `-n` 将主机、端口和用户名用数字代替
    
    * `-t` 只列出tcp协议的连接
    
    * `-l` 表示过滤state是LISTEN的连接
    
    * `netstat -apn | grep port` 显示指定端口的状态信息和进程信息
    
    * `netstat -tnlp` 显示所有状态为LISTEN的TCP连接
    
    	
* tcpdump
    * `tcpdump host ip` 截获主机发出和收到的数据包
    * `tcpdump port 6666` 截获端口上通过的包
    * `tcpdump -i eth0` 截获某网卡上的包
* ping
    * `ping ip` 用于测试另一台主机是否可达，测试网络是否连通以及时延
    * windows下ping是32比特，默认发送4次数据包结束
    * linux下ping是64比特，默认不停发送数据包，直到手动停止
* host
    * `host 域名` 返回域名的IP地址
    * 用来查询DNS记录
* ifconfig
    * 输出当前系统中所有处于活动状态的网络接口
    * `ifconfig eth0 ip/24` 手工指定网卡的IP地址和广播地址，其中广播地址可以根据掩码计算出来
    * `ifconfig eth0 up` 启动网卡eht0
    * `ifconfig eth0 down` 关闭网卡eht0
* traceroute

## 字符处理命令
* 管道|
* grep

## 调试命令
* gdb
    * `l` 列出函数代码及行数
    * `b 16` 在16行设置断点
    * `b func` 在函数func设置断点
    * `r` 运行程序
    * `n` 单条执行程序
    * `p i` 打印i变量的值
    * `bt` 查看函数堆栈
    * `finish` 退出函数
    * `q` 结束调试
* strace
    * 监控用户空间进程和内核的交互，跟踪系统调用和信号传递
    * `strace -c ./test` 统计./test使用的系统调用
    * `strace -p pid` 跟踪现有进程
* ipcs
    * 用于报告系统的消息队列，信号量和共享内存等使用情况
    * `ipcs -a`用于列出本用户所有相关的ipcs参数
    * `ipcs -q`用于列出进程中的消息队列
    * `ipcs -s`用于列出所有的信号量
    * `ipcs -m`用于列出所有的共享内存信息
    * `ipcs -l`用于列出系统限额，比如共享内存最大限制
    * `ipcs -u`用于列出当前的使用情况
* ipcrm
    * 用于移除一个消息队列，或者共享内存段，或者一个信号集，同时会将与ipc对象相关联的数据也一起移除，只有超级管理员，或者ipc对象的创建者才能这样做
    * `ipcrm -M shmkey`  移除用shmkey创建的共享内存段
    * `ipcrm -m shmid`    移除用shmid标识的共享内存段
    * `ipcrm -Q msgkey`  移除用msqkey创建的消息队列
    * `ipcrm -q msqid`  移除用msqid标识的消息队列
    * `ipcrm -S semkey`  移除用semkey创建的信号
    * `ipcrm -s semid`  移除用semid标识的信号 

## 文本处理工具
### grep

文本搜索工具，根据用户指定的“模式”对目标文本逐行进行匹配检查；打印匹配到的行

模式：由正则表达式字符以及文本字符所编写的过滤条件

### sed

- 行编辑器，sed是一种流编辑器，它一次处理一行内容。处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间”，接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。然后读入下行，执行下一个循环。如果没有使用诸如‘D’的特殊命令，那会在两个循环之间清空模式空间，但不会清空保留空间。这样不断重复，直到文件末尾。文件内容并没有改变，除非你使用重定向存储输出。
- 功能：主要用于自动编辑一个或多个文件，简化对文件的反复操作，编写转换程序等



## 删除文件夹里一天前创建的文件

```bash
find ./ -mtime -1 -exec rm -rf {} \;
```

首先查找文件，用find，-mtime -1表示一天前，-exec 命令 {} \ 是固定组合，"{}"表示前面的内容，也就是“find ./ -mtime -1”得到的内容