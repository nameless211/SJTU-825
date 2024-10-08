# Representing Information

## Outline

字母编码：

1. ASCII码

数字编码：

1. 无符号数的二进制表示
2. 有符号数通过补码（two‘s complement）表示
3. 浮点数的表示以及计算

算术运算：

1. 初等算术运算
2. 逻辑运算（按位，逻辑）
3. 位运算
4. 运算的优化

## Storing Info

byte是存储信息的基本单位，bit是计算机存储的基本单元

### 十六进制、二进制、十进制及其转化

![image-20230910101846373](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910101846373.png)

使用短除法使十进制向二进制以及十六进制转换

### 字数据大小

P27 定义

也就是64位和32位的定义

32位和64位下的各种数据类型所占的字节数如下

| 数据类型  | 32bit | 64bit |
| --------- | ----- | ----- |
| char      | 1     | 1     |
| short     | 2     | 2     |
| int       | 4     | 4     |
| long      | 4     | 8     |
| long long | 8     | 8     |
| int32_t   | 4     | 4     |
| int64_t   | 8     | 8     |
| pointer   | 4     | 8     |
| float     | 4     | 4     |
| double    | 8     | 8     |

- int32_t以及int64_t为C99引入的固定大小数据类型，和编译设定无关

- char一般被编译器视为signed，但是需要通过[signed]关键字声明才能确认

- 数据类型的大小按不同平台（32、64）会有所不同，但是C++标准会规定每种数据类型至少得有多大

  ![image-20230910104516301](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910104516301.png)

#### 多字节对象的存储***

big-endian&little-endian

- 取决于最高字节在前面（低内存地址）还是后面（高内存地址）

- 以0x2A4FC971为例

  | 地址                  | 0x100 | 0x101 | 0x102 | 0x103 |
  | --------------------- | ----- | ----- | ----- | ----- |
  | big endian（大端）    | 2A    | 4F    | C9    | 71    |
  | little endian（小端） | 71    | C9    | 4F    | 2A    |

**尤其需要注意在阅读机器代码的时候需要将小端法的数据转换**

### 字符串对象

ACSII码

‘0’\~‘9’：0x30~0x39

'A'：0x41

'a'：0x61

字符串以null（0x00）结尾

## Logos

### Boolean

![image-20230910111242739](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910111242739.png)

基于异或的inplace_swap函数

![image-20230910142234829](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910142234829.png)

#### C中的逻辑运算符

特性：**如果根据第一个运算数就能确定表达式的值，第二个运算数就不参与运算**

### bit operations

移位运算从左至右可结合

1. 左移：右边补0
2. 右移：
   - 逻辑右移：左边补0
   - 算术右移：左边补1
   - 对于有符号数，只能使用算术右移，无符号数只能使用逻辑右移

## Integer&Arithmetic Operations***

### 整数的二进制表示

这种表示被看作是一种双射，二进制数据不能被看作是整数本身，就算是有符号数的表示也一样

#### 无符号数(U)

注意范围是0~2^n^-1

#### 补码(T)

设二进制编码数组
$$
[x_{\omega-1},x_{\omega-2}……,x_0]
$$
则其向对应的整数，也即其补码编码对应的数字转换的方式如下
$$
B2T_\omega(\vec{x})\doteq-x_{\omega-1}2^{\omega-1}+\sum_{i=0}^{\omega-2}x_i2^i
$$
其实就是将最高的$\omega-1$位所对应的位权重从$2^{\omega-1}$换成了$-2^{\omega-1}$

于是补码的范围是
$$
[1000……0]\sim[01111……0]，-2^{\omega-1}\sim2^{\omega-1}-1
$$
并不是对称的，但是是双射

1. 不对称是由于0包含在正数，也即是符号位为0的数里面了

2. 同样位数的二进制表示，U~max~=2T~max~+1

3. 从补码向二进制编码转换

   ![image-20230910151036228](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910151036228.png)

   例如补码为x，则其二进制表示为~x+1

#### 其他有符号数的表示

![image-20230910151401576](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910151401576.png)

#### 补码和无符号数

1. 相互转换

   大部分编译器中二者进行强制类型转换的定义都是同一个二进制编码，分别用补码和无符号数的方式进行转换。如果是一者向另一者转换，就先变成二进制编码再转换。

   ![image-20230910152809678](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910152809678.png)

   补码向无符号数转换（ω位）（或者也可以-1再求补，或者先求补再+1）
   $$
   tx= \left\{
   	\begin{aligned}
       &u   && u \leq TMax_\omega \cr
       &x^2 && u <    TMax_\omega 
       \end{aligned}
   	\right.
   $$
   

   无符号向补码转换（ω位）

2. T+U=2^ω^

   x+~x+1

3. 在C中的一些特殊情况

   赋值，格式化输出等都可能导致隐式的强制类型转换

   在运算中混用两种表示会导致所有的运算数都被强制类型转换为无符号数

### 相关运算

#### 扩展位数

从小位数向大位数转换总是可行

无符号数直接逻辑右移即可

有符号数直接算术右移

同时需要扩展以及强制类型转换时，先扩展再转换

```c++
short sx;
printf("%u",(unsigned)sx);
```

将会先把short转换为int，再将int转换为unsigned

#### 截断位数

![image-20230910161805556](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910161805556.png)

实际上做的操作都是将前面的位数截断掉，然后根据不同的表示来决定其数值

### 算术运算

#### 加法

固定位数的加法运算会导致溢出问题，从而需要进行截断，从而产生数值上的错误

1. 无符号数加法

   得到的值需要进行取模运算来截断

   取模加法构成阿贝尔群（逆元由刚刚溢出的数值决定）

2. 补码加法

   直接将补码视为无符号数进行加法运算（需要截断），再将得到的结果视为补码即可
   $$
   sum=U2T(T2U(x)+^u_{\omega}T2U(y))
   $$
   

   利用了无符号数的取模加法（证明见P63）

   正溢出将会导致结果变为负数（超过TMax），反之同理

   补码同样存在逆元（P66）

#### 乘法

同样是直接截断

**性质：位级等价性**

同样二进制表示的补码和无符号数进行乘法运算，其乘积的二进制表示是相同的（如果有溢出，则在截断后相同）

**利用位级等价性优化乘法**

无符号数与2的幂进行乘法运算等价于左移其幂次的位数。

而由于补码和无符号数的乘法具有位级等价性，因此补码与2的幂进行乘法运算也能使用同样的方法进行运算

只需要将一个乘数转换为二进制，就能利用乘法的分配律将乘法转换为数个移位操作和加法操作结合
$$
x*14=x*1110_2=x*8+x*4+x*2=(x<<3)+(x<<2)+(x<<1)
$$

#### 除法

**整数除法的舍入规则：正数向下舍入，负数向上舍入**

对于2的幂，无符号数直接进行逻辑右移即可，**而负的有符号数进行逻辑右移的结果将会导致向下舍入，不符合规则**

解决的方法是通过加入一个偏置量：
$$
\lceil x/2^k\rceil=(x+(1<<k)-1)>>k
$$
若x的低k位有值，则所加上的这个（1<<k）将不会被-1消掉，相当于在结果上+1，即是向上舍入。而若没有值，这个-1将会使得（1<<k）=2^k^变成2^k^-1，从而会被消掉，结果不变。

然而无法拓展到所有常数

## Floating Points

本质上是使用二进制小数来描述十进制小数

二进制小数的小数位权重为2的负幂次

就好像使用十进制无法精确表示$\frac{1}{3}$一样，二进制也无法精确表示$\frac{1}{10}$

### IEEE标准

为了同时能够做到对小数以及较大的数的表示，IEEE标准将浮点数划分为三块
$$
V=(-1)^s\times M\times 2^E
$$

- s为符号位
- M为尾数，为一个二进制小数，范围是1\~2-ε或0~1-ε
- E为阶码，为浮点数加权，从而使得其可以表示较大的数（E>0），也可以表示较小的数（E<0）

![image-20230910172948693](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910172948693.png)

![image-20230910172959516](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910172959516.png)

实际上的表示有四种情况，根据阶码exp字段的情况而定：（计E的位数为e，M的位数为f）

需要注意的是，浮点数公式中的E、M等只是在Encoding中使用二进制表示，不代表他们等于二进制所直接表示的值

1. 规格化的值（exp字段不全为0或1）

   由于E的取值范围包含正数和负数，因此需要进行偏置得到。
   $$
   E=e-Bias
   $$
   其中e即是exp字段的二进制对应的无符号数，Bias为$2^{e-1}-1$

   二进制小数，也即是尾数M使用frac字段表示，这些二进制编码被解释为f=0.frac，也就是一个二进制小数的小数点后位数。M=f+1，相当于有一个implicit 1

2. 非规格化的值（exp字段全为0）

   E=1-Bias，M=f。

   由于没有implicit 1（为了能够表示0），因此E相应的加上了一个1，从而使得最大的非规格化值和最小的规格化值之间相差较小。
   $$
   ((1+0)\times 2^{1-Bias}- (1-2^{-frac})\times 2^{1-Bias} 
   $$

3. 特殊值（exp字段全为1）

   无穷：frac全为0，根据s的值判断是+∞还是-∞

   NaN：frac不全为0

正浮点数可以使用Unsigned的排序函数进行排序，这是因为它的二进制表示所对应的Unsigned具有相同的大小关系。负浮点数只需要去掉标志位再进行排序即可（注意其Unsigned的升序即是原浮点数的降序）

### 整数向浮点数转换

先将小数点移到最高位，记下左移的位数作为E，将得到的1.frac截断为浮点对应的长度，再将他们转换为浮点的表示。

#### 舍入规则

需要使结果不会造成统计上的偏差（例如平均值的变化等）

采用向偶数舍入的方法：

1. 优先向最近的表示舍入
2. 当数字是两个可能舍入的值的中间值时，向尾数为偶数的舍入（也就是说最低有效位为0的）

### 浮点数计算

![image-20230910181347604](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910181347604.png)

乘法

![image-20230910181334207](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910181334207.png)

![image-20230910181411720](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910181411720.png)

加法

![image-20230910181431934](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910181431934.png)

![image-20230910181503629](https://markdown-zyy.obs.cn-east-3.myhuaweicloud.com/img/image-20230910181503629.png)