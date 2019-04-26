# plan9汇编笔记

# 从一段函数调用代码说起，创建一个main.go文件
```
package main

//go:noinline
func add(a, b int32) (int32, bool) { return a + b, true }

func main() { add(10, 32) }

```

使用命令 go tool compile -S main.go 生成汇编文件。

```
"".add STEXT nosplit size=20 args=0x10 locals=0x0
        0x0000 00000 (main.go:10)       TEXT    "".add(SB), NOSPLIT, $0-16
        0x0000 00000 (main.go:10)       FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:10)       FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:10)       FUNCDATA        $3, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:10)       PCDATA  $2, $0
        0x0000 00000 (main.go:10)       PCDATA  $0, $0
        0x0000 00000 (main.go:10)       MOVL    "".b+12(SP), AX
        0x0004 00004 (main.go:10)       MOVL    "".a+8(SP), CX
        0x0008 00008 (main.go:10)       ADDL    CX, AX
        0x000a 00010 (main.go:10)       MOVL    AX, "".~r2+16(SP)
        0x000e 00014 (main.go:10)       MOVB    $1, "".~r3+20(SP)
        0x0013 00019 (main.go:10)       RET
        0x0000 8b 44 24 0c 8b 4c 24 08 01 c8 89 44 24 10 c6 44  .D$..L$....D$..D
        0x0010 24 14 01 c3                                      $...

```
接下来一行一行的来分析 

```
 0x0000 00000 (main.go:10)       TEXT    "".add(SB), NOSPLIT, $0-16
```
+ 0x0000 : 当前指令相对于当前函数的偏移量 

+ TEXT "".add ：TEXT 指令声明了 "".add 是 .text 段(程序代码在运行期会放在内存的
.text 段中)的一部分，并表明跟在这个声明后的是函数的函数体。在链接期，"" 这个空字符会被替换为当前的包名: 也就是说，"".add 在链接到二进制文件后会变成 main.add。

+ (SB): SB 是一个虚拟寄存器，保存了静态基地址(static-base)
指针，即我们程序地址空间的开始地址。

+ NOSPLIT: 向编译器表明不应该插入 stack-split 的用来检查栈需要扩张的前导指令

+ $0-16: $0 代表即将分配的栈帧大小；而 $16 指定了调用方传入的参数大小。

```
0x0000 00000 (main.go:10)       MOVL    "".b+12(SP), AX
0x0004 00004 (main.go:10)       MOVL    "".a+8(SP), CX
```
(SP): SP stack pointer,表示栈顶
 
 Go的调用规约要求每一个参数都通过栈来传递，这部分空间由 caller 在其栈帧(stack
 frame)上提供。调用其它过程之前，caller
 就需要按照参数和返回变量的大小来对应地增长(返回后收缩)栈。

Go 编译器不会生成任何 PUSH/POP 族的指令: 栈的增长和收缩是通过在栈指针寄存器 SP 上分别执行减法和加法指令来实现的。

"".b+12(SP) 和 "".a+8(SP) 分别指向栈的低 12 字节和低 8 字节位置(记住:
栈是向低位地址方向增长的！)。
从汇编层面看到a离栈顶更近，说明参数是从右向左入栈的。那么为什么a的地址是8(sp)而不是0(sp)呢？

因为调用方通过使用CALL伪指令，把其返回地址保存在了0(sp)处。

```
0x0008 00008 (main.go:10)       ADDL    CX, AX
```
ADDL 进行实际的加法操作，L 这里代表 Long，4 字节的值，其将保存在 AX 和 CX 寄存器中的值进行相加，然后再保存进 AX 寄存器中。

```
0x000a 00010 (main.go:10)       MOVL    AX, "".~r2+16(SP)
0x000e 00014 (main.go:10)       MOVB    $1, "".~r3+20(SP)
```        

将add操作的结果从寄存器AX取出，移动到"".~r2+16(SP)位置，"".~r2并没有语义。 常量bool的返回是通过$1,
"".~r3+20(SP)移动到栈所对应位置，因为内存对齐的缘故，所以bool值也占用4个字节。

```
0x0013 00019 (main.go:10)       RET
```
最后的 RET 伪指令告诉 Go 汇编器插入一些指令，这些指令是对应的目标平台中的调用规约所要求的，从子过程中返回时所需要的指令。
一般情况下这样的指令会使在 0(SP) 寄存器中保存的函数返回地址被 pop 出栈，并跳回到该地址。

![stack frame](../doc/images/stack_frame.png)
上面是main.add 即将执行 RET 指令时的栈的情况.