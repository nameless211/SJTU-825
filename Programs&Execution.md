# Programs&Execution

## Outline

汇编程序：

1. 汇编指令
   - 访问信息指令
   - 算术与逻辑指令
   - 控制指令
     - 实现循环，条件控制下的跳转
   - 浮点相关指令
2. 阅读汇编程序

程序执行：

1. 运行过程
   - 运行时数据结构
   - 程序中数据结构的分配和访问
   - 理解程序运行的过程
2. 安全性
   - 应对buffer overflow

## 程序编码

### 机器级代码

汇编程序展现为对一个地址空间中的各种信息做顺序操作，这得益于两个重要抽象：

1. ISA，指令集体系架构。处理器的硬件能够并发的执行这些指令，同时执行的结果表现为顺序执行。
2. 虚拟内存。

编译器有时会插入一些nop指令，从而使得代码块更符合存储器的分块

### 指令集

基于x86-64的指令集

大部分指令，根据其操作数的位数不同，带有b、w、l、q的后缀，分别对应byte，word，long（实际上是double words），quad words

#### 寄存器

x86-64一共有16个通用寄存器

![image-20230912180156854](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912180156854.png)

![image-20230912180141651](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912180141651.png)

- 其中，%rax是返回值，%rsp是指向栈顶的指针。其他寄存器有部分可以用于传参，其他根据调用时的保存情况分成caller-saved和callee-saved
- 64位的寄存器同样支持针对32位及以下的寄存器指令，具体的截断和扩展规则在具体指令处细化

#### 操作数

1. 立即数

   用$来标识

2. 寄存器

   汇编代码中写作%r~a~，其存储的值写作R[r~a~]

3. 内存引用

   通用寻址模式是Imm(r~a~，r~b~，s)，代表M[Imm+R[r~a~]+R[r~b~]$\times$s]，也就是对应地址的内存中的值。同时还有很多部分省略的变种

![image-20230912181643286](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912181643286.png)

#### 数据传送指令

##### mov***

| 指令                 | 效果            | 描述           |
| -------------------- | --------------- | -------------- |
| MOV             S，D | S   $\rarr$   D | 传送           |
| movb                 |                 | 传送byte       |
| movw                 |                 | 传送word       |
| movl                 |                 | 传送双字       |
| movq                 |                 | 传送四字       |
| movabsq     I，R     | I   $\rarr$   R | 传送绝对的四字 |

1. Source可以是一个立即数，寄存器或者内存地址，Destination可以是一个寄存器或者内存地址

2. x86-64限制S、D不能同时为内存地址，这意味着内存地址之间的传值需要两个mov操作来完成（这和硬件实现有关）

3. movl较为特殊，虽然它是32bit的指令，但是它会将目的寄存器的高32位字节设置为0

   **值得注意的是，所有为寄存器生成32位数据的指令都会将其高32位清零，例如movzbl等**

4. 当Source为64位的立即数时，只能使用movabsq指令（同时它也只能用于将64位立即数存储到寄存器）。movq虽然支持对64位的寄存器和内存进行操作，但只支持32位的立即数，并将其进行符号扩展后放入64位的目的位置

**针对需要扩展（也就是从较小的位置到较大的位置）的情况，x86-64提供了符号扩展和零扩展的指令**

**这些指令只用于目的地为寄存器的情况，对源操作数则不予限制**

1. 零扩展

   本来一共有6个，但是由于movl的特殊性，movz只有五条

   另外还有**cltq指令**，它的作用和movslq %eax，%rax相同

   ![image-20230912183105071](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912183105071.png)

2. 符号扩展

   ![image-20230912183245301](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912183245301.png)

##### 栈操作***

由于x86-64是64位的，因此其栈操作都是64位的操作，带q（quad）后缀

| 指令          | 描述       |
| ------------- | ---------- |
| push       S  | 将四字入栈 |
| pop         D | 将四字出栈 |

出入栈操作实际上包含两个部分，即维护栈顶指针以及栈内容。所以入栈等价于：

```assembly
subq $8, %rsp                              	#维护栈顶指针的变化
movq %R, (%rsp)								#将入栈的内容存入栈中
```

注意栈从高地址向低地址增长

出栈则等价于

```assembly
movq (%rsp), %R								#将出栈的内容存入目标寄存器中
addq $8, %rsp                              	#维护栈顶指针的变化
```

注意此时原来栈顶的内容并没有被覆盖

通过基本的mov指令加上%rsp可以任意访问栈中的内容。

#### 算术&逻辑指令

1. 加载有效地址

   ![image-20230912192536127](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912192536127.png)

2. 一元操作

   ![image-20230912192556142](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912192556142.png)

3. 二元操作

   ![image-20230912192621906](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912192621906.png)![image-20230912192640048](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912192640048.png)

4. 移位操作

   ![image-20230912192658719](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912192658719.png)

5. 特殊算术操作

   ![image-20230912195155753](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230912195155753.png)

##### 加载有效地址

```assembly
leaq S, D                #&S（内存引用） → D（寄存器）
```

尽管leaq指令在形式上看起来是将一个内存中的数据移动到寄存器中，就好像movq会做的事情一样，它却不涉及对内存的读取操作。它所作的事情只是将这个内存引用对应的有效地址写入目的寄存器。

leaq指令同样可以被用于一些简单的可以通过通用寻址模式表示的算术运算：

```assembly
#x in %rdx
leaq 7(%rdx,%rdx,4)               #5x+7
```

这个时间效率会比imul更高

##### 一元操作

目的和源操作数相同，做一些自增自减取反取负的操作

##### 二元操作***

第一个是源操作数，第二个是目的操作数，运算的结果被存放于目的操作数中。运算式子为
$$
D\bigoplus S
$$

其中源操作数可以是立即数、寄存器或内存位置，目的操作数可以是寄存器或内存位置。

##### 移位操作

- 注意两个操作数的含义

- 移位量k可以是一个立即数，也可以是存储在特殊单字节寄存器%cl中

- 如果移位量k大于指令所允许的最高位数，则k的高位会被忽略。举例来说：

  sall对32位进行操作，则移位量k将只会保留低5位（最大31位的移位量）

- 由于有两种右移，因此给出了两种右移指令

- 虽然左移只有一种，但是仍然对应的给出了两个左移指令，他们的实现和效果是完全相同的。

##### 特殊算术操作

**他们都是单操作数指令**

由于64位乘法能够得到最高128位（称oct word，八字）的结果，因此增加了特殊的乘法和除法支持：

1. 乘法

   有无符号数和补码版本，都是将结果的128位分别存储在两个寄存器中，%rdx存储高8字节，%rax存储低8字节。

   尽管补码版本的指令名称和64位版本的相同，但是由于操作数的个数不同，汇编器仍然能够识别所需使用的指令。

2. 四字向八字转换

3. 除法

   同样有两个版本。他们取%rdx（高8字节）和%rax（低8字节）中的值作为被除数，而将商存储于%rax，余数存储于%rdx。

   - 由于大多数除法运算也只涉及64位的被除数，此时%rdx会被设置为0或全1（根据除法的类型以及被除数的符号而定）
   - 无符号数版本使用xor %rdx，%rdx将rdx设置为0，补码版本使用cgto，将%rax中的被除数进行符号扩展

#### 控制指令

**控制指令的基本逻辑是：根据一些测试的结果来改变程序运行的控制流。**

这就需要两个部分：对数据的测试以及结果的储存、根据储存的结果改变控制流。

##### 条件码 EFLAGS

条件码寄存器是一组单bit的寄存器

| 条件码      | 作用                       |
| ----------- | -------------------------- |
| CF 进位标志 | 最高位进位                 |
| ZF 零标志   | 结果为0                    |
| SF 符号标志 | 结果为负数                 |
| OF 溢出标志 | 补码溢出（正溢出或负溢出） |

- 所有的条件码针对的都是最近的操作结果

- CF称进位其实有点不准确，其描述是：

  ![image-20230914152753646](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914152753646.png)

  这不仅可以指无符号数溢出，也可以是左移操作（被设置为最后一个被移出的位）

- 所有的算术操作都有可能会隐式的设置这些flag

  ![image-20230914153210645](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914153210645.png)

##### 逻辑指令

不改变其他寄存器，只设置条件码

| 指令            | 行为        | 描述 |
| --------------- | ----------- | ---- |
| CMP S~1~，S~2~  | S~2~ - S~1~ | 比较 |
| TEST S~1~，S~2~ | S~2~ & S~1~ | 测试 |

- CMP和TEST分别有字节到四字四个版本
- 根据CMP不同的FLAG情况可以用来判断大小情况
- TEST可以用于检测是否相等或利用掩码来检测操作数的某些特定位数

##### 访问条件码

条件寄存器和其他整数寄存器不同，不仅不能直接将其他值放入其中，也不能直接取其中的值。用法一般有三种：

1. 根据条件码的组合（从而推断某个操作的结果）来设置某个字节的值
2. 根据条件码的组合跳转到程序的另一个部分
3. 根据条件码的组合传送数据

这就引出了三大类指令：SET，JMP，CMOV

###### SET

当执行了CMP之后，根据条件码可以判断两个数的大小关系，从而将目的操作数设置为0|1。

<img src="https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914160926329.png" alt="image-20230914160926329" />

- 典型的用法如下（判断a<b）：

  ```assembly
  #a in %rdx，b in %rbx
  
  cmpq 	%rbx, %rdx		#比较a、b的大小
  setl 	%al				#根据比较的值设定%al单字节寄存器的值
  movzbl 	%al, %eax		#将%eax（实际上也是%rax）的其他全部字节清零
  ```

  注意比较的顺序

- set的目的操作数是单字节的，意味着它只能是单字节内存位置或者低位单字节寄存器。

  所以set的后缀都是在表明其比较的关系

- sets表示判断cmp的结果是否为负可能有些奇怪（在这里，结果为负还要考虑溢出的情况），但是考虑到cmp实际上就是sub操作也是合理的

- setl、setg是用于有符号数，seta、setb用于无符号数

- 对于有符号的比较比较好理解。对于无符号的比较，以a<b为例

  由于cmp相当于在执行a-b，因此若a-b<0，会设置进位标志

###### JMP***

JMP指令的目标可以是一个Label或者是一个以*加上内存引用格式声明的地址。根据JMP条件的不同，有一系列JMP指令

| 指令             | 条件                  |
| ---------------- | --------------------- |
| jmp              | 无条件跳转            |
| je、jne          | 相等情况              |
| js、jns          | 结果为负数/非负数情况 |
| jg、jge、jl、jle | 用于有符号数的判断    |
| ja、jae、jb、jbe | 用于无符号数的判断    |

- jmp的目标是用PC-relative的方式进行指定的
- jmp的目标并不是Label本身，而是label的下一行，从PC的角度考虑也是挺自然的

###### CMOV

![image-20230914190129061](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914190129061.png)

![image-20230914190624801](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914190624801.png)

### 使用条件指令来实现控制流

基本的条件分支有下列几种：

1. ```c++
   if(expr)
       then-statement
   else(expr)
       else-statement
   ```

2. ```c++
   do
   	body-statement
   while(test-expr)
   ```

3. ```c++
   while(test-expr)
   	body-statement
   ```

4. ```c++
   for(init-expr; test-expr; update-expr)
       body-statement
   ```

5. ```c++
   switch(oper)
   	case val_0:
   		val_0-statement;
       case val_1:
   		val_1-statement;
   	.
       .
       case val_n:
   		val_n-statement;
   ```

我们利用if-goto来模拟cmp-jmp的逻辑，通过模仿汇编的逻辑来理解这些控制逻辑

#### IF语句

![image-20230914190151885](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914190151885.png)

但是由于处理器的pipeline机制，需要在test-expr得出结果之前就继续运行下面的指令，因此这种跳转的方式可能会导致很高的预测失败开销。

为了避免这一点，可以采用条件传送的思路，也就是先将两个statement的结果都算出来，根据test-expr的结果来决定是哪一个结果要被传到%rax里面作为返回值。(由于条件传送需要读取条件码，因此不存在预测的问题)

<img src="https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914190531740.png" alt="image-20230914190531740" style="zoom:50%;" />

<img src="https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914190551651.png" alt="image-20230914190551651" style="zoom:50%;" />

条件传送所存在的问题是：就算条件不满足，两个分支还是都被计算过了。如果必须在条件满足的情况下才能执行分支语句，那么就有可能导致错误。

#### Do-while循环

![image-20230914192136087](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914192136087.png)

![image-20230914192327192](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914192327192.png)

#### while循环

![image-20230914192353207](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914192353207.png)

通过一个无条件跳转使得第一次body-statement在判断了条件之后再执行

**guarded-do**

在一个do-while循环之前加上了一个判断条件，在第一次条件总是通过的时候可以剩下一次JMP操作

![image-20230914193010552](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914193010552.png)

![image-20230914193027973](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914193027973.png)

#### For循环

可以转换为用while循环实现的形式

![image-20230914193236706](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914193236706.png)

- 但是注意，如果在body中有continue的话，这样转换会导致update被跳过。需要使continue跳过到update前面
- while循环的翻译仍然有两种

#### Switch语句

switch语句的主要特点在于其具有多重分支，对应实现为跳转表

**跳转表：用对应的值来索引，表中的内容是对应的标号**

C版本：

![image-20230914195144236](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195144236.png)

跳转表

![image-20230914195204590](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195204590.png)

![image-20230914195212550](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195212550.png)

汇编版本：

![image-20230914195239118](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195239118.png)

- 跳转表，注意是存储在.rodata里面的
- 对于一些不连续的，如101和105，在跳转表中仍然存在，只是对应的label是def

![image-20230914195434861](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195434861.png)

![image-20230914195522328](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230914195522328.png)

对于有fall through的情况，直接没有对应的jmp即可。

## 程序运行

### 栈&调用***

过程的抽象是一种封装代码的方式，通过将一系列代码封装成可调用的代码块，能够通过清晰的接口为其他函数提供调用支持。

为实现过程的抽象，以一个从过程Q到过程P的调用为例，需要提供三个基本机制：

1. 传递控制
2. 传递数据
3. 分配和释放内存

#### 运行时栈

通过栈的先进先出的特性，我们得以实现对每个过程中的数据进行单独管理、在过程返回时释放内存。

**栈帧**

- 不是每个过程都有对应的栈帧（例如不需要栈传参、不需要调用函数，所有局部变量都能存储在寄存器中的过程）
- 大部分过程的栈帧是定长的，在调用时就已经分配好了

#### 对变长栈帧的支持

某些函数中存在大小会变化的数据结构（例如变长数组，每次运行函数根据参数不同大小会变），针对这些情况的栈帧即为变长栈帧

主要问题是，当编译器不知道分配多大的栈帧的时候，就无法确定释放空间时需要对%rsp做怎样的处理。

为此，x86-64使用%rbp作为栈帧寄存器（注意它是callee-saved，意味着它需要被压入栈中保存），保存在分配局部变量之前的栈顶位置。之后直接通过改变%rsp的方法进行内存分配，最后返回到%rbp记录的位置即可（通过leave指令完成）

![image-20230916194835127](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916194835127.png)

#### 控制转移

控制转移包含两个步骤：

1. 从调用者转移到被调用者
2. 从被调用者转移到调用者

从机器的角度来讲，就是需要将PC的值设置为被调用者代码的起始位置，同时还要保存调用者运行到的位置，以便在返回的时候能继续执行调用者过程。

**call-ret指令**

通过调用call指令，将返回地址（call后面一行代码）压入栈中并且将PC设置为被调用过程的起始代码位置

- 值得注意的是，压入栈中的返回地址是调用者栈帧的一部分，原因是在被调用者返回前其栈帧已经被清除（**通过add %rsp指令显式的清除**），调用者栈帧的最后一部分（也就是返回地址）就在栈顶，直接pop出来即可

| 指令          | 描述                             |
| ------------- | -------------------------------- |
| call Label    | 直接调用，目标为标号             |
| call *Operand | 间接调用，目标为操作数指示的地址 |

#### 数据传送

包括将数据作为参数传递给被调用进程和被调用进程返回结果两个部分。

一般传参直接将对应参数放到寄存器里即可，被调用者直接读取他们就能得到对应的值

![image-20230916155319493](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916155319493.png)

返回时固定将返回值放到%rax里

- 如果参数不是64位的，那么可以通过低位寄存器来访问
- 可以看到，通过寄存器最多传递6个参数，多出来的参数需要通过栈来传递

##### 栈传参

调用过程时，如果有多出的参数，会放到调用者的栈帧中

![image-20230916155856997](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916155856997.png)

- 所有的参数都会round-up到8byte，小数据放到低位

  ![image-20230916155955537](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916155955537.png)

- 调用者需要预先准备好给这些参数的空间

- 被调用者通过%rsp来访问这些值，注意返回地址仍然存在，因此参数的最低地址是%rsp+8

当所有参数都准备好了（存储到寄存器、push到栈上），就可以调用call将控制转移到被调用者了

#### 栈上的内存分配

一般程序的变量都存储在寄存器中，而某些情况下需要在内存中分配空间来存储变量，一般有如下情况：

- 寄存器不够了
- 变量需要通过引用来访问，因此需要一个内存地址
- 数组或结构等必须使用引用来访问

其中，局部数据结构一般是存储在栈上；而通过malloc，new分配的数据结构存储在堆中，不过其引用仍然可能作为一个局部变量被分配到栈上。

**这里需要注意的是，栈中所分配的局部数据结构不能过大，否则可能超出栈的大小限制**

![image-20230916170220106](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916170220106.png)

- 和传递参数类似，局部变量在栈上的存储也是通过类似的方式完成的。在空间的分配和释放上也遵循类似的规律
- 但是局部变量没有align
- 访问它们时使用的地址和%rsp相关

![image-20230916170000627](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916170000627.png)

- 被调用后，马上存储callee-saved的状态‘
- 由于一般栈的定长特性，就算还没有用到这个局部变量，其空间也需要事先分配好。这是在完成寄存器保存之后紧接着需要完成的
- 因此局部变量会在保存的寄存器下面

#### 寄存器状态在调用中的保存

由于所有过程共享一组寄存器状态，因此需要在调用时保证寄存器不被覆盖。

**callee-saved（被调用者保存）**

![image-20230916165815629](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916165815629.png)

在被调用后马上进行保存

![image-20230916163530104](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916163530104.png)

- 如果P调用了Q，那么Q就需要保持这些寄存器的值不变。
- 这意味着Q或者根本不去使用这些寄存器，或者将原值push到自己的栈帧中，在**返回前**再将他们恢复到栈帧中保存的状态
- 这同时也意味着P在调用返回之后能够直接使用这些寄存器中的值
- 一般当P有需要保存的寄存器中的值时，可以转移到callee-saved寄存器中（**注意需要保存callee-saved寄存器中原本的值，保存到栈上**）

**caller-saved（调用者保存）**

![](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916165742765.png)

在调用前进行保存

![image-20230916163857707](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916163857707.png)

- 所有寄存器除%rsp外或者是callee-saved或者是caller-saved
- 如果P调用了Q，那么P需要自己保存好这些寄存器的值来在调用返回时恢复寄存器状态
- P当然是不想由自己来做这种事情的，因此所有P中需要保存的局部变量值都会优先存储到callee-saved里面，存满了再自己保存

#### 递归

由于一个过程的所有信息或者由它自己，或者由它所调用的过程保存，递归调用成为可能。

#### 总结

![image-20230916170537830](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916170537830.png)

栈的总体结构

### C的数据结构

除开普通的变量，C中还有数组，union，struct等较大且不规则的数据结构。将他们分配到栈上显然会导致一个相当臃肿的运行时栈结构，这是不被推荐的。因此这些数据结构将会被放到内存空间中，并且通过指针的方式进行访问。

#### 同质

由于数组中的所有元素都是同样的数据类型，因此数组是一种同质的数据结构。这种特性让它能够紧凑的被分配在一块连续的内存空间上，从而使得访问数组元素变得简单。

通过x86-64提供的基本寻址模式能够简单的访问数组元素：
$$
(\%r_a,\%r_b,n)=array[(\%r_b)]
$$
其中n=size（array_element）

![image-20230916182801342](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916182801342.png)

##### 多维数组

多维数组在内存中的存储也是顺序的，可以直接用数组指针加偏置的方法访问

![image-20230916182837408](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916182837408.png)

**针对定长多维数组的优化**

![image-20230916183041488](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916183041488.png)

![image-20230916183101673](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916183101673.png)

- 将对连续的数组元素的访问优化为用指针进行的访问
- 不用对数组的第一维再寻址

##### 变长数组

这里的变长指的并不是动态分配的数组，而是数组大小在编译时无法确定的数组。只要数组大小在创建数组时能够确认即可

#### 异质

##### struct

类似数组，结构也是将一系列对象连续的存储再内存中的数据结构。但是由于这些对象的类型可能不同，结构需要保存对应不同的位置索引，并且为它们提供align。

值得注意的是，访问结构中的不同元素这部分完全由编译器对结构体指针进行偏移来实现

##### union

union是一个可以被看作不同类型的数据结构。虽然如此，但一般来说对一个union类型的访问只在把它当成一个特定的数据类型时才合法。

对于一些字段较多，字段的使用又互斥的数据结构而言，union可以节省存储空间

##### 数据对齐

由于内存访问的接口有规定大小（4、8、16等），因此将内存对象朝对应大小对齐能够节省读取它所需要的访问次数

![image-20230916190405178](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916190405178.png)

![image-20230916190442312](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230916190442312.png)

要求起始地址与其对应大小的倍数对齐即可

### 栈的数据安全

#### Buffer Overflow

通过越界的内存访问，非法程序可以修改函数的返回地址，并且在修改后的返回地址植入攻击代码

#### Code Reuse

通过插入一系列返回地址，可以调用内存中的可用代码（一般是库函数），从而达成攻击目的

#### 防护

1. 栈随机化

   比较早期的方法通过在程序运行开始时分配一段随机空内存空间，即可保证每次运行的时候栈地址发生变化。

   在现在的机器上程序的所有部分都有随机化，它们将会被加载到不同的内存空间中，从而防止buffer overflow攻击

   应对这种随机化的方法是：通过插入一段足够长的nop指令序列，使得只要固定的返回地址落到这段序列中就能够开始攻击。通过不断枚举是很可能攻破一般的随机化防御

2. 栈破坏检测

   通过在返回地址之前插入一段随机生成的值，并且在返回前检测这个值是否被改变，能够防止错误的返回地址。用于比较的副本存在内存中并且为只读模式，不会被改写。

3. 限制可执行代码区域

   由于老版本的CPU只支持读写两种访问限制，因此所有可读代码也是可执行的。而栈显然必须是可读的，因此也将是可执行的。现代CPU加入了标志执行与否的标签。通过只把编译得到的代码所在的内存页标记为可执行，实现了对代码区域的限制

## 浮点代码

