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



