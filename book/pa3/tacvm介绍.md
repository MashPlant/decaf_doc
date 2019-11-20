# tacvm简述

## 简介

[tacvm](https://github.com/MashPlant/tacvm)是用来执行tac的解释器。

tacvm从文本中parse出tac语句，然后经过一定转换后执行。目前我们还没有将tacvm中的tac与框架中的tac统一化，也就是说目前仍然无法通过编写tac文本来直接构造可供编译器后续处理的tac。这的确是一个很重要的feature，例如llvm的ir就有内存形式，bit code形式，文本ir形式这三种形式，三者是完全一致，彼此一一对应的，这给予了llvm ir很大的灵活性和通用性。只是我们的tac的设计初期就没有怎么考虑这个问题，现在要统一起来工作量较大，希望在将来能够实现这个feature。

剩余的内容将介绍tacvm支持的格式，以及我们的框架中生成对应的文本的接口。

## 基本结构

程序有两种顶层元素，即`VTBL<id>`和`FUNC<id>`，两者的格式差不多，都是在尖括号内放名字，后续的花括号内放以换行符为分隔符的内容。

tac的parser对除了换行符之外的空白字符都不敏感，但对换行符敏感(花括号内不允许出现空白的一行)。

所有的标识符都定义为：`[0-9A-Za-z._]+`，包括上面的`id`和tac指令中用到的各种"名字"。

tacvm运行的入口是名字为`main`的函数。

## tac格式

大体上来说tac的格式与java/scala框架采用的tac解释器的格式是差不多的，不过还是有一定差别。

以下约定：

1. rᵢ表示一个虚拟寄存器；在tac文本中，**它只能是`_T`后接非负整数的格式**，不能任意起名字
2. oᵢ表示一个虚拟寄存器或者一个**整数**常量(o代表operand)
3. lᵢ表示一个标号；在tac文本中，**它只能是`_L`后接非负整数的格式**，不能任意起名字
4. 下表中的"额外信息"一列，含义分别为
   1. `Bin` / `Un`：`op`字段
   2. `Jif`：`z`字段
   3. `Call`：`kind`字段
5. 下表中`[]`表示可选的，即对应tac文本中可以有对应内容或者没有(不存在其它框架中的`<empty>`这种东西)

| 名字 | 额外信息 | 输出 | 含义 |
|-----|---------|---------------------------|-----|
| Assign |  | rᵢ = oⱼ | 把j的值赋给i |
| LoadVTbl |  | rᵢ = VTBL&lt;C&gt; | 把名字为C的虚表的地址赋给i |
| LoadFunc |  | rᵢ = FUNC&lt;f&gt; | 把函数f的地址赋给i，它不能是intrinsic函数 |
| LoadStr |  | rᵢ = "str" | 把字符串"str"的地址赋给i |
| Un | Neg | rᵢ =  - oⱼ | 把j的相反数赋给i |
| Un | Not | rᵢ =  ! oⱼ | 把j逻辑非赋给i |
| Bin | Add | rᵢ = (oⱼ + oₖ) | 把j和k的和赋给i |
| Bin | Sub | rᵢ = (oⱼ - oₖ) | 把j和k的差赋给i |
| Bin | Mul | rᵢ = (oⱼ * oₖ) | 把j和k的积赋给i |
| Bin | Div | rᵢ = (oⱼ / oₖ) | 把j除以k的商赋给i中 |
| Bin | Mod | rᵢ = (oⱼ % oₖ) | 把j除以k的余数赋给i |
| Bin | Eq | rᵢ = (oⱼ == oₖ) | 若j等于k则i为1，否则为0 |
| Bin | Ne | rᵢ = (oⱼ != oₖ) | 若j不等于k则i为1，否则为0 |
| Bin | Lt | rᵢ = (oⱼ < oₖ) | 若j小于k则i为1，否则为0 |
| Bin | Le | rᵢ = (oⱼ <= oₖ) | 若j小于等于k则i为1，否则为0 |
| Bin | Gt | rᵢ = (oⱼ > oₖ) | 若j大于k则i为1，否则为0 |
| Bin | Ge | rᵢ = (oⱼ >= oₖ) | 若j大于等于k则i为1，否则为0 |
| Bin | And | rᵢ = (oⱼ && oₖ) | 把j和k逻辑与操作的结果赋给i |
| Bin | Or | rᵢ = (oⱼ \|\| oₖ) | 把j和k逻辑或操作的结果赋给i |
| Jmp |  | branch lᵢ | 无条件跳转到标记i所表示的地址 |
| Jif | true | if (oᵢ == 0) branch lⱼ | 如果i为0则跳转到j所表示地址 |
| Jif | false | if (oᵢ != 0) branch lⱼ | 如果i不为0则跳转到j所表示地址 |
| Ret |  | return [oᵢ] | 结束函数，函数返回值为i或者**未定义** |
| Param |  | parm oᵢ | 将i作为调用的参数传递 |
| Call | Virtual(oⱼ) | [rᵢ =] call oⱼ | 调用地址为j的函数，结果放到i或者丢弃 |
| Call | Static(id) / Intrinsic(i) | [rᵢ =] call f | 调用名字为f函数，结果放到i或者丢弃 |
| Load |  | rᵢ = *(oⱼ - 4) | 把地址为j-4的单元的内容加载到i |
| Store |  | *(oᵢ + 4) = oⱼ | 把j保存到地址为i+4的内存单元中 |
| Label |  | lᵢ: | 定义一个标记i(局部的，仅本函数内可以跳转) |

所有的虚拟寄存器的都是局部有效的，即不同函数之间的虚拟寄存器的值不会互相干扰。

tac函数的参数传递过程是这样的：
  - 调用者使用`parm`指令把参数存入一个临时的空间中
  - `call`指令时，将临时空间中的内容复制到被调用的函数的前若干个连续的虚拟寄存器中，之后清空临时空间

## 虚表格式

`VTBL`中可以出现以下内容
  - `"str"`，填入对应的字符串地址
  - `int`，填入对应整数
  - `FUNC<id>`，填入名字为`id`的函数的地址
  - `VTBL<id>`，填入名字为`id`的虚表的地址

`VTBL`的内容和顺序并不要求符合我们的编译器框架中的内容和顺序，即编译器的虚表中只用到了tacvm中的虚表的有限的功能。

## 库函数

纯粹的表示运算，跳转，调用decaf函数等功能的tac是没办法实现诸如io，停止程序等需要和外部世界打交道的功能的。早在语法分析阶段就强行设定了几个"函数"，如`ReadInteger()`之类的，现在它们也不可能只用前面这些功能来实现，必须允许tac调用一些不属于decaf的函数，也就是decaf的运行时库函数，我们用`Intrinsic`来表示它们(这种函数似乎叫外部函数或者内部函数都挺合理的，想表达的意思都是语言本身无法实现或无法高效实现的函数)：

框架中所有intrinsic函数定义如下：

```rust
pub enum Intrinsic { 
  _Alloc, _ReadLine, _ReadInt, _StringEqual, _PrintInt, _PrintString, _PrintBool, _Halt 
}
```

**它们在生成的tac中的名字就是各个variant的名字**。它们的功能如下：

- `_Alloc`
  - 功能：分配内存
  - 参数：为要分配的内存块大小(单位为字节)
  - 返回值：该内存块的首地址
- `_ReadLine`
  - 功能：读取一行字符串
  - 参数：无
  - 返回值：读到的字符串首地址
- `_ReadInt`
  - 功能：读取一个整数
  - 参数：无
  - 返回：读到的整数
- `_StringEqual`
  - 功能：比较两个字符串
  - 参数： 两个，分别为要比较的两个字符串的首地址
  - 返回值：表示两个字符串相等的bool值
- `_PrintInt`
  - 功能：打印一个整数
  - 参数：要打印的数字
  - 返回值：无
- `_PrintString`
  - 功能：打印一个字符串
  - 参数：要打印的字符串首地址
  - 返回值：无
- `_PrintBool`
  - 功能：打印一个布尔值
  - 参数：要打印的布尔变量
  - 返回值：无
- `_Halt`
  - 功能：结束程序
  - 参数：无
  - 返回值：无，且它会终止tacvm的运行

## 错误检测

在开始运行之前parser会先进行一遍检查，例如需要确保所有的名字引用都是能够找到对应对象的。这个阶段的检查是静态的，**无论某段代码是否会被运行到，其中的错误都需要汇报出来**。这里可能需要注意的一点是，对于tac中的除法和取模，当两个操作数都是常数，且右操作数为0时，这个阶段会汇报一个错误。

tacvm会检测tac代码的运行错误，这里直接列出tacvm中表示所有的错误种类的代码：

```rust
#[derive(Debug)]
pub enum Error {
  // program calls _Halt explicitly
  Halt,
  // base % 4 != 0 or off % 4 != 0 or allocate size % 4 != 0
  UnalignedMem,
  // base == 0
  NullPointer,
  // base is in an invalid object
  MemOutOfRange,
  // base is in a valid obj, base + off is not or in another obj
  ObjOutOfRange,
  // access to string id is invalid(typically a string id is not a valid mem address)
  StrOutOfRange,
  // instruction fetch out of range
  IFOutOfRange,
  // call a register which is not a valid function id
  CallOutOfRange,
  // call stack exceeds a given level(specified in RunConfig)
  StackOverflow,
  // instructions exceeds a given number(specified in RunConfig)
  TLE,
  // the number of function arguments > function's stack size
  TooMuchArg,
  // div 0 or mod 0
  Div0,
  // fails in reading or writing things
  IO,
}
```

相信注释写的比较清晰了，这里补充几点：

- 虽然从tac格式来看访存的偏移量可以是任意值，但是只要不是4的倍数模拟器都会报错；正常情况下框架不会生成偏移量不是4的倍数的访存，所以如果实际运行时触发了`UnalignedMem`，一定是因为基地址不是4的倍数
- `ObjOutOfRange`的逻辑是：基地址在一个合法对象中但基地址+偏移量不在同一个对象中则报错，虽然这并不一定是一次非法的访存，但是decaf的语言模型中不可能出现这样的访存，所以可以确定是生成的tac不符合decaf语义，所以需要报错
  - 然而，上面的"不可能"其实太绝对了，还真有一种情况：访问一个长度为0的数组的长度。现在，**tacvm对这种合法的访存会报错**。尽管如此，因为还没有想到什么比较优美的修复方式，我们不打算修复这个问题。测例会避开这样的情形
  - 访问一个长度为0的数组的长度还可能触发`MemOutOfRange`，原理是类似的，我们同样不打算修复
  - 其它框架使用的tac模拟器有也有完全类似的问题
- 所有函数都必须显式地返回，即使是decaf返回void的函数，否则执行到函数末尾时会触发`IFOutOfRange`
  - 虽然在decaf语法层面上允许返回void的函数不显式写出`return`，但是ir层面的`return`是不能省略的，**任何正常的ir或者汇编代码应该都是这样要求的**
- 目前decaf的tac翻译策略中还没有检测空指针的错误，这个错误目前在tac层面可以由tacvm检测，但是在汇编阶段则没有任何保护了
  - 也许未来我们会考虑在到tac的翻译中加入对于对象，字符串和数组的空指针检测。显然，如果没有高效的编译优化，这样的检测会产生很大的运行开销。目前我们的编译优化还很初级，还没有达到这个程度
  - 空指针检测也不一定需要依靠tac代码或者汇编代码来体现，另一种可行的方案是利用os提供的虚拟内存和信号机制，[SIGSEGV as control flow - How the JVM optimizes your null checks](https://jcdav.is/2015/10/06/SIGSEGV-as-control-flow/)对此有一定的描述
  - 现在大家直接忽略这个问题就好了，可以**假装**我们已经合理地处理了空指针

当检测到一个运行时错误并且开启了`stacktrace`选项之后，tacvm会输出发生错误时刻的栈帧信息，例如：

```
VM halted with error: Div0
stacktrace: 
  - function `main`, line 561, code `call f`, [_T0 = 0]
  - function `f`, line 556, code `_T6 = (_T6 / _T6)`, [_T0 = 1, _T1 = -1610612728("string"), _T2 = -572662307(uninitialized), _T3 = 233, _T4 = 120(ptr), _T5 = -1342177257(func<main>), _T6 = 0]
```

其中唯一需要解释的是后面方括号内的内容。它列出了本函数内所有虚拟寄存器的值，同时还对每个值可能的含义做了一个**猜测**，也就是说它可以用作这些用途，但不能保证代码本身的意图就是把它用作这个用途。

为了让这个猜测更加接近真实，tacvm中设定字符串的地址和函数的地址(其实叫做编号更好)都是从一个特定的非零值开始增长的，这样就比较不容易和小整数相同。当值为`0xDDDDDDDD`的时候猜测它为`uninitialized`，这是每个虚拟寄存器的初值。不过你的代码中显然不应该依赖这些性质。

顺便说一句，访问字符串，访问函数，和其他的访存，你可以理解成三者并不共用同一片地址空间。例如对于字符串地址的访存几乎一定会失败(不管是否按4对齐了)。

这些设定都是为了帮助大家更好的调试，对tacvm的运行没有任何影响，如果你一次实现正确的话，那么他们其实都毫无意义。

## 运行参数

tacvm可以作为一个库链接到程序中(我们的测试程序就是这样用的)，也可以作为一个独立的程序。为了避免跨平台的问题，框架中并没有提供编译好的版本，大家如果想作为一个独立的程序运行tacvm，自己去github上clone一份然后编译即可。

目前tacvm提供以下的运行参数：

```
--inst_count # 指定info_output是否输出执行的指令条数
--stacktrace # 指定info_output中是否在tac发生运行时错误时输出stacktrace
--info_output # tacvm的运行信息输出路径
--inst_limit # tacvm至多执行多少条指令
--stack_limit # tacvm至多允许多少层调用栈
--vm_input # tacvm的输入路径(如ReadLine()等函数用到)
--vm_output # tacvm的输出路径(如Print()等函数用到)
<file> # tacvm的输入tac文件
```

框架的测试程序中`inst_limit`和`stack_limit`设置的都不是很大(分别是100000和1000)，不过也都够用了。