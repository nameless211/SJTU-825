# Exceptional Control Flow

## Outline

- 异常
  - 异常的种类
  - 异常处理
- 进程
  - 逻辑流和并发
  - 进程的基础
    - 私有地址空间
    - 用户态和内核态、context switch
  - 进程控制
    - 创建，中止，回收进程
    - 程序的运行
- 信号
  - 基本过程
  - 发送和接收信号
  - 处理信号、并发问题
- 非本地跳转

## 异常

![image-20231023144418877](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023144418877.png)

当一个事件发生（可能是由于程序的执行或者是外部信号等），处理器检测到事件，并且通过异常的方式“接管”执行流，对异常进行处理后再返回

P502有详细过程

### 异常处理

1. CPU检测到发生了一个事件，并且确认其对应的异常号
2. CPU触发异常，通过间接过程调用，根据异常号索引到异常表中的对应异常处理程序
3. 处理程序结束后，可选地返回到对应被中断执行地程序，并将控制和数据恢复

**索引过程**

![image-20231023145952105](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023145952105.png)

- 异常表中每一个条目都是一块异常处理函数的起始地址，只要能通过异常号找到对应的异常表条目，就能通过间接过程调用（就是直接跳转到一块地址上，并且将它的内容当成函数来执行）运行处理函数

- 异常表在计算机的启动阶段进行初始化，在各个条目中填上对应地址

- ![image-20231023150236259](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023150236259.png)

  异常号*8（64位系统）再加上异常表基地址寄存器中存储的异常表的起始地址，就能找到对应的项

根据异常处理的过程，实际上比较类似于函数调用，执行流转换到另一个程序，执行后再返回继续执行原先的程序，但是它也有不同之处：

1. 异常处理的返回地址可能是程序的下一条或本条指令
2. 在间接过程调用时可能会将部分处理器状态压入栈中
3. 异常处理程序总是运行在内核态下。如果被中断的函数是用户态函数，那么它的状态会被压入内核栈，并向内核态下陷

### 异常的类型

![image-20231023150822907](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023150822907.png)

一共有四类异常，主要注意其诱因和返回行为

*在这里同步指异常由某一条指令导致，异步指任何时候都可能发生，和特定的指令无关

#### 中断

![image-20231023151427527](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023151427527.png)

![image-20231023151459222](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023151459222.png)

- 通过中断引脚的高电平来通知CPU
- 许多中断的处理程序（如DMA）只是发起了一个请求，而实际的处理程序在中断返回后继续运行，和CPU的指令并行，在这个意义上也是异步的

#### trap&syscall

- 通过一条指令主动发起
- 针对用户程序需要请求敏感功能（read，execve等），又需要和内核隔离设计的特殊调用，称syscall
- syscall所调用的函数处在内核态，能够访问内核栈以及执行特殊指令

#### fault

- fault由错误情况引起，而这个错误情况可能是由一条指令引起（如引用虚拟地址导致缺页异常）
- 异常处理程序会尝试修复这个错误情况
- 如果修复成功，会重新执行那条引起错误的指令
- 失败则直接终止（返回到内核态中的abort例程从而终止程序运行）

#### 终止

无法修正的错误，直接返回到abort例程，终止程序运行。

### Linux/x86-64异常实例

- 异常表中有CPU定义的（除零，缺页等）和操作系统定义的（syscall等）两种异常
- linux系统调用的传参和普通函数不同
- 和call \<function name>不同，linux系统调用将对应异常号放入%rax寄存器，然后直接执行syscall指令即可

## 进程

![image-20231023144230874](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023144230874.png)

操作系统建立在三大基本抽象上，而本章进程即是基本抽象之一，通过进程来管理CPU资源和程序权限

进程的基本作用：

1. 使得程序都好像能够占用所有内存和CPU资源
2. 举例来说，一个执行中程序的实例即是一个进程。这个程序运行在这个进程的上下文（context）中，也即是包含所有程序状态的数据集合（寄存器，代码，栈，PC，环境变量，文件）
3. 创建进程并在其上下文中运行可执行文件是程序运行的基本方式

这部分主要包括进程是如何提供CPU和内存上的假象的

### 逻辑控制流

系统所实际执行的指令序列称作逻辑控制流

![image-20231023162326867](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023162326867.png)

- 途中包含三个逻辑流
- 对于每一个程序而言，它们的执行都是顺序的
- 而对于CPU而言，它所执行的指令来自不同的程序
- 因此对于单个进程而言，它好像是独占处理器，实际上确实轮流执行的

#### 并发流

- 两个逻辑流的运行时间重叠（指A为B子集或B为A子集）即为并发流
- 并发和是否抢占资源等无关，只和运行的时间有关

### 私有地址空间

根据处理器提供的地址位数，每个进程都为程序提供2^n^大小的地址空间。这部分内容在虚拟内存中有详细的介绍

- 它们具有相同的结构
- 程序存储的数据在程序看来就是在这个地址空间中的
- 实际上由操作系统维持一层从这个地址空间到实际内存的映射，即虚拟内存机制。一般进程是不能访问其他进程保存在内存中的数据的，因而将其称为私有的

![image-20231023163052480](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023163052480.png)

### 用户态和内核态

针对隔离的需要，CPU将不同的进程设置为内核或用户态，从而限制它们能访问的地址空间以及能使用的指令

- 通过一个模式位来标记（set-内核）
- 当处于内核态时，进程被允许访问内核地址空间以及执行特权指令（如停止CPU，发起IO等）
- 用户态反之
- 当发生异常时，进行间接调用的同时，进程会陷入内核态，从而允许异常处理程序执行特殊代码。返回时进程回到用户态

linux通过proc文件系统允许进程不陷入内核态也能访问所有进程以及它们的状态

#### context switch

- CPU通过上下文切换来实现进程之间的调度
- 上下文切换的过程中，一个进程的状态会被保存（都是寄存器状态，内存不用，因为只要进程对应的页表仍然存在，就能访问所有进程的私有地址空间中的数据）
- 注意上下文切换和向内核态的下陷不同。虽然向内核态的下陷也需要保存部分寄存器状态，但是其使用的仍然是同一个地址空间，也就是说进程并没有切换。

由于系统调用常常会导致进程阻塞（如DMA等），因此系统调用可能会导致上下文切换

中断也可能导致上下文切换（如时钟中断等）

## 进程控制

### 创建进程

**进程状态**

1. 运行：一个运行的进程不代表是正在执行的进程，还有可能是被挂起而正在等待CPU调度
2. 停止：通过一些特定信号可以让进程停止运行，也叫挂起。这样的进程不会被CPU调度。直到某些信号使之再次启动
3. 终止：进程终止运行。可以通过信号，程序执行完毕或者显式调用exit函数来实现

**进程创建：fork**

1. 创建的子进程拥有和父进程完全相同的虚拟地址空间和file descriptor
2. 调用一次，返回两次：由于fork只会在父进程中被调用，因此只有一次调用。而由于fork返回时已经有两个进程了，因此会在两个进程中各返回一次，父进程中返回进程PID，子进程中返回0
3. 虽然拥有相同的地址空间，它们的地址空间却是独立的
4. 打开的文件是共享的

*创建完子进程后，两个进程并发执行，其执行顺序不定

#### 进程组

- 每个进程都属于某个进程组

- 通过fork创建的子进程和父进程属于同一个进程组

- 可以显式地更改一个进程的进程组

  ```c++
  setpgid(pid_t pid, pid_t pgid)
      //更改pid对应进程的所在进程组
  ```

  

### 回收子进程

子进程终止不代表其所有资源都被清除。只有被回收了的进程才是被清除了

![image-20231023202056291](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023202056291.png)

一个进程如果还没有回收所有子进程就结束了，那么init进程会回收它们

**waitpid**

用于检查子进程的状态并报告

![image-20231023195606612](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023195606612.png)

如果调用waitpid的进程没有子进程，返回-1并设置errno为ECHILD，如果被信号中断则设置errno为ENTR

参数：

- pid：决定waitpid所处理的“等待集合”

  - \>0：由pid所指定的子进程
  - -1：调用者的所有子进程
  - =0：调用者所在进程组中的所有子进程（在调用waitpid时的）
  - \<-1：由pid的绝对值所指定的进程组中的所有子进程

- options：决定在何种情况下waitpid对子进程进行回收

  - =0（默认情况）：waitpid将调用者挂起，直到检测到任何等待集合中的子进程终止（如果在调用时已经有子进程终止了，那么直接返回），返回所有已终止子进程的pid。这个过程也会将这些子进程回收
  - WNOHANG：如果调用时没有已经终止的子进程，不将调用者挂起等待，立刻返回0
  - WUNTRACED：除了等待被终止的子进程外，被停止的子进程也会引起函数返回，并且返回值中会有所有被停止的子进程的pid
  - WCONTINUED：除了等待被终止的子进程外，还等待已经停止的子进程重新开始
  - 这些选项能够使用逻辑连接词组合起来使用

- statusp：如果给出了statusp，那么waitpid在返回之后还会在statusp所指向的地址放入导致返回的子进程的相关信息。这些信息可以用一些宏来解释

  | WIFEXITED    |      |
  | ------------ | ---- |
  | WEXITSTATUS  |      |
  | WIFSIGNALED  |      |
  | WTERMSIG     |      |
  | WIFSTOPPED   |      |
  | WSTOPSIG     |      |
  | WIFCONTINUED |      |

  可以看到它们检测进程导致waitpid返回的原因，并且可以检查信号编号，退出状态等

wait函数和waitpid（-1，&status，0）相同

### 进程休眠

通过调用sleep能够让进程挂起一段指定时间

通过pause能让进程挂起到收到信号

### 加载并运行程序

![image-20231023203109341](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023203109341.png)

加载可执行文件filename，argv为程序的参数列表，envp为环境变量

![image-20231023203247540](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023203247540.png)

- argv[0]总是可执行文件的名称

**开始执行时进程栈的结构**

![image-20231023203444144](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023203444144.png)

- 启动参数和环境变量采用表的方式索引，表头在寄存器中

- 有几个操作环境数组的函数

  ![image-20231023203809626](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231023203809626.png)

  - getenv：查找name对应的value
  - setenv：将name对应value修改为newvalue，那么不存在则添加一个name-newvalue；若overwrite为0则不覆盖已有的name-value，只在需要添加时操作
  - unsetenv：删除name

## 信号

![image-20231024151549891](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024151549891.png)

信号是一种消息，用于通知进程发生的事件。不同于异常会导致进程下陷并使用预定义的处理函数进行处理，进程接收到信号之后的行为是可以自定义的（除了某些特殊信号）

### 信号过程

传送一个信号到目的进程分两步：

1. 发送信号

   - 所谓“发送”其实就是内核直接更改一个进程的上下文状态。
   - 信号的来源可以是内核，因为检测到某种事件从而向进程发送异常（SIGSEGV，SIGCHLD）；也可以是其他进程通过调用kill函数显式地进行信号传递

2. 接受信号

   - 进程收到一个信号后，可以选择忽略，终止或者调用信号处理程序，这个过程叫做接受信号

   - 由用户自定义的信号处理程序运行在用户态，但信号处理本身是在内核态下的

     ![image-20231024152814210](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024152814210.png)

   - 一个进程可能会有收到了但没有接收（也就是没有处理）的信号，称待处理信号（pending）。这样的信号一种类型只能有一个，多出的会被丢掉

   - 一个进程还可以阻塞部分信号，它能够收到这些信号，但不会对他们进行处理（永远pending）

   - 进程使用两个向量组来分别维护待处理和阻塞信号

     ![image-20231024153246222](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024153246222.png)

### 发送信号

1. 使用/bin/kill发送信号

      ```shell
      linux> /bin/kill -信号ID 进程(>0)/进程组(<0)ID
      ```
      代表对某个进程或一个进程组中的所有进程发送对应ID的信号

2. 从键盘发送信号

      由于一行命令行可能需要执行多个进程，Linux shell提供作业（job）抽象来处理这种情况。

      ![image-20231024162350829](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024162350829.png)

      每个作业都有自己独立的进程组

      ![image-20231024162430196](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024162430196.png)

      通过键盘操作能够对作业进行控制，如：

      1. Ctrl+C：发送SIGINT到前台作业进程组，默认终止这个前台作业
      2. Ctrl+Z：发送SIGTSTP，默认挂起它

3. 使用kill函数发送信号

      ![image-20231024162625488](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024162625488.png)

      使用方法和/bin/kill相同

      如果pid等于0，则发送信号给调用进程所在进程组中的每个进程（包括自己）

4. 使用alarm函数发送信号

      ![image-20231024162812898](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231024162812898.png)

      - alarm安排内核在sec秒之后给自己发送SIGALRM信号
      - alarm调用后立即返回
      - 同时只能存在一个alarm，之后设置的闹钟会导致之前还没有触发的闹钟终止，并在本次调用的返回值中告知之前设置闹钟还剩余的时间

### 接受信号

- 内核将进程p从内核态切换到用户态时，对进程pending的信号做检查，如果非空，则依照次序处理所有的信号（linux，后到的先处理）

- 在信号处理程序中，这种切换也会对pending的信号做检查，并且如果发现了有信号，会中断现在的信号处理程序，转而处理新的信号

- 大部分信号对应的处理逻辑都可以自定义

  ![image-20231025085436807](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025085436807.png)

  通过修改handler，定义signum对应信号的行为

  - handler如果是SIG_IGN或SIG_DFL，代表行为是忽略和恢复默认行为

  - 不同信号定义处理逻辑时可以使用同一个handler

  - error handling

    ![image-20231025085656371](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025085656371.png)

### 阻塞/解除阻塞信号

**隐式机制**

任何当前进程正在处理的信号类型的信号都会被阻塞。这意味着就算在处理的过程中有新的此类信号，它们也不会中断当前信号处理程序的运行，直到它返回，而这个信号变为待接收。

**显式机制**

![image-20231025090159577](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025090159577.png)

- how定义函数的行为，set为函数用于改变blocked mask的参数，oldset非空时用于存储函数作用前blocked mask的值

- how参数

  - SIG_BLOCK：将set中的信号添加到blocked
  - SIG_UNBLOCK：将set中的信号取消blocked
  - SIG_SETMASK：将set设置为blocked mask

- set使用一系列函数来进行设置

  ![image-20231025090459785](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025090459785.png)

  | empty    | 初始化为空              |
  | -------- | ----------------------- |
  | fill     | 将每个信号添加至set     |
  | add      | 添加signum              |
  | del      | 删除signum              |
  | ismember | 检查signum是否为set成员 |

### 编写信号处理程序

由于信号可能在主程序执行的任何时候将其中断并开始处理信号，因此可以看作它和主程序并发的执行，而中断类似于一次调度。此时信号处理程序和主程序在同一个进程中，因此具有同样的全局数据结构。它们对全局数据结构的访问是并发的。因此在信号处理的过程中需要注意并发安全性

- G0：编写尽可能简单的信号处理程序
- G1：使用异步信号安全的函数
- G2：保存和恢复errno：因为许多函数都会设置errno，即便它们是异步信号安全的。因此在进入信号处理函数时需要保存主程序的errno（除非信号处理函数不会返回主程序，也就是当它调用_exit终止进程时）
- G3：在访问共享全局数据结构时应该屏蔽所有的信号
- G4：使用volatile关键字
- G5：使用sig_atomic_t声明标志（见显式的等待信号）

\**：异步信号安全的定义：指**可重入**或**不可被信号处理程序中断的函数**。其中可重入指的是可以被打断并且无论被打断后执行了何种操作，它都能继续运行并返回正确结果（代码上就是指它不依赖任何全局数据结构，只使用自己栈上的数据即可）

\**：普通的线程安全的函数并不能保证异步信号安全。这是因为线程安全有时依赖于对其中一个线程的阻塞，而信号函数的阻塞会带来死锁问题

**：volatile关键字：要求编译器不能将其优化为在寄存器中缓存。这避免了处理函数对全局变量的修改对主程序不可见。

\**：sig_atomic_t类型能够保证对标志的原子操作，从而不需要在对它们进行操作时屏蔽信号

**一些安全的函数**

大部分IO函数都不安全（唯一安全的是write），因此提供了一系列被称为SIO的包装函数。它们使用可重入的函数和write共同构建了异步信号安全的IO函数

![image-20231025094017953](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025094017953.png)

#### 正确的信号处理

**not queued**

多个信号接连发送时，很可能导致存在信号丢失的问题。不能认为“收到一个信号处理一个”，而是应该将收到一次信号作为事件开始的标志，直到确保不会再有同类信号之后再结束处理

P537

**portable handling**

在不同的Unix系统中信号处理的语义不同，因此需要针对性编程

或者可以使用sigaction函数来自定义语义

![image-20231025100033666](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025100033666.png)

之前提到的Signal函数除了能够设置handler以外，还指定了信号处理的语义，从而使得其具有可移植性

![image-20231025100210114](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025100210114.png)

#### 处理并发错误

以一个race为示例

![image-20231025100951280](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025100951280.png)

![image-20231025101004953](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025101004953.png)

必须保证addjob在deletejob之前完成，并且addjob和deletejob需要被保护。所以：

1. addjob和deletejob执行的过程中需要block所有信号
2. 在父进程中，子进程创建前到addjob完成之后都需要block SIGCHLD来防止在完成addjob之前子进程就已经完成执行并发送SIGCHLD，导致deletejob被执行
3. 在fork之后任意时间点子进程都可能被调度，因此不能在fork之后再block

#### 显式地等待信号

一种方法是通过一个全局的共享变量，在主程序中不断测试它的值，而在handler中改变它的值。这样能够使得主程序能够感知到其变化。

问题在于循环测试很浪费资源

下面是一个linux shell的模拟程序

![image-20231025103114503](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025103114503.png)

改成pause或sleep：

![image-20231025103145977](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025103145977.png)

- pause在收到信号时会停止休眠。为了防止其他信号（SIGINT）终止休眠，增加了一个循环，这样就算终止了，如果没有修改pid，也就代表不是SIGCHLD，那么就继续休眠
- 需要注意的是，由于在处理完当前任务之前不能终止程序，SIGINT被重定义为一个被接受但不产生实际作用的信号。这对应了shell中在有作业时需要等待它完成回收工作再结束而不是直接结束的行为。但是它也会导致pause的休眠结束。
- 但是如果在pause之前，while之后有SIGCHLD，那么处理完之后就没有信号来唤醒pause，导致永远休眠
- sleep使得程序最长在收到SIGCHLD之后过一秒才能反应过来（在sleep开始之前收到，要等这次sleep结束之后才能再次检查pid）

采用sigsuspend函数

![image-20231025104254972](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025104254972.png)

- 相当于下列代码的原子版本

  ![image-20231025104320989](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025104320989.png)

- 它暂时替换block的信号组，并在收到信号时停止休眠，进行信号处理，再恢复原本block的信号组

- 这样保证了信号总是在pause之后才被收到（因为解除mask和pause是原子的）

![image-20231025104621755](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025104621755.png)

## 非本地跳转

一对setjump-longjump能够使得在后面任意位置调用longjump时从setjump处返回

相当于setjump设置了一个位置，使得后面调用longjump时能够跳转到这个位置

![image-20231025110642333](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20231025110642333.png)

- 用于信号处理函数的版本有一个前缀
- 由于longjump可以跳转到任意代码，因此函数的执行流可能相当混乱。基于这个原因，它们不被认为是异步信号安全的，并且需要在longjump可达的范围内只使用安全的函数



