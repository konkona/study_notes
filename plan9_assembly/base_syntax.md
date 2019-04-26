# 基本指令
## 栈调整
intel 或 AT&T 汇编提供了 push 和 pop 指令族，plan9 中没有 push 和 pop，栈的调整是通过对硬件 SP 寄存器进行运算来实现的，例如:

```
SUBQ $0x18, SP // 对 SP 做减法，为函数分配函数栈帧
...               // 省略无用代码
ADDQ $0x18, SP // 对 SP 做加法，清除函数栈帧
```

## 数据搬运
常数在 plan9 汇编用 $num 表示，可以为负数，默认情况下为十进制。可以用 $0x123 的形式来表示十六进制数。

```
MOVB $1, DI      // 1 byte
MOVW $0x10, BX   // 2 bytes
MOVD $1, DX      // 4 bytes
MOVQ $-10, AX     // 8 bytes
```
可以看到，搬运的长度是由 MOV 的后缀决定的。方向从做到右。

## 常见计算指令

```
ADDQ  AX, BX   // BX += AX
SUBQ  AX, BX   // BX -= AX
IMULQ AX, BX   // BX *= AX
```

## 条件跳转/无条件跳转
```
// 无条件跳转
JMP addr   // 跳转到地址，地址可为代码中的地址，不过实际上手写不会出现这种东西
JMP label  // 跳转到标签，可以跳转到同一函数内的标签位置
JMP 2(PC)  // 以当前指令为基础，向前/后跳转 x 行
JMP -2(PC) // 同上

// 有条件跳转
JNZ target // 如果 zero flag 被 set 过，则跳转
```

## 伪寄存器
```
FP: Frame pointer: arguments and locals.
PC: Program counter: jumps and branches.
SB: Static base pointer: global symbols.
SP: Stack pointer: top of stack.
```

+ FP: 使用形如 symbol+offset(FP) 的方式，引用函数的输入参数。例如 arg0+0(FP)，arg1+8(FP)，使用 FP 不加 symbol 时，无法通过编译，在汇编层面来讲，symbol 并没有什么用，加 symbol 主要是为了提升代码可读性。另外，官方文档虽然将伪寄存器 FP 称之为 frame pointer，实际上它根本不是 frame pointer，按照传统的 x86 的习惯来讲，frame pointer 是指向整个 stack frame 底部的 BP 寄存器。假如当前的 callee 函数是 add，在 add 的代码中引用 FP，该 FP 指向的位置不在 callee 的 stack frame 之内，而是在 caller 的 stack frame 上。具体可参见之后的 栈结构 一章。
+ PC: 实际上就是在体系结构的知识中常见的 pc 寄存器，在 x86 平台下对应 ip 寄存器，amd64 上则是 rip。除了个别跳转之外，手写 plan9 代码与 PC 寄存器打交道的情况较少。
+ SB: 全局静态基指针，一般用来声明函数或全局变量，在之后的函数知识和示例部分会看到具体用法。
+ SP: plan9 的这个 SP 寄存器指向当前栈帧的局部变量的开始位置，使用形如 symbol+offset(SP) 的方式，引用函数的局部变量。offset 的合法取值是 [-framesize, 0)，注意是个左闭右开的区间。假如局部变量都是 8 字节，那么第一个局部变量就可以用 localvar0-8(SP) 来表示。这也是一个词不表意的寄存器。与硬件寄存器 SP 是两个不同的东西，在栈帧 size 为 0 的情况下，伪寄存器 SP 和硬件寄存器 SP 指向同一位置。手写汇编代码时，如果是 symbol+offset(SP) 形式，则表示伪寄存器 SP。如果是 offset(SP) 则表示硬件寄存器 SP。务必注意。对于编译输出(go tool compile -S / go tool objdump)的代码来讲，目前所有的 SP 都是硬件寄存器 SP，无论是否带 symbol。