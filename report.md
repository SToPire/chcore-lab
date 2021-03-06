#### 练习1

1. AArch64有31个通用寄存器（X0-X30），x86-64是8个。可以用W0-W30指定这些通用寄存器中的低32位部分。

2. AArch64有专门的stack pointer寄存器（4个，每个特权级一个），x86-64使用了通用寄存器中的esp。
3. AArch64有4个特权级别（0-3），数字越大特权级越高。有3个link register在特权级切换时存储返回地址，3个saved program status register存储程序状态。
4. AArch64的基址+偏移寻址模式中，加上一个感叹号表示寄存器写回，基址寄存器会被更新为加上偏移量后的值。[here](https://stackoverflow.com/questions/39780289/what-does-the-exclamation-mark-mean-in-the-end-of-an-a64-instruction)

#### 练习2&3

入口地址在0x80000处，这是定义在start.S中的`_start`函数的起始地址。该函数读了MPIDR_EL1寄存器中的低8位，以此判断一个cpu core是否为primary，并让非primary的core陷入死循环。

#### 练习4

`.text` `.rodata` `.bss`节的LMA和VMA不一致。

对于一般的用户态程序而言，LMA和VMA一般是一致的，反正内核会负责映射LMA对应的物理地址并完成加载，之后执行，用户程序不需要感知物理地址。但是对内核而言，它的任务之一就是构建页表和虚拟地址空间，所以LMA是加载到的物理地址，VMA是虚拟地址。内核会通过页表在这两者间建立映射，并启动mmu。

#### 练习6

在head.S文件中的`start_kernel`函数中完成了对内核栈的初始化，内核栈位于`kernel_stack`所在的内存区域，这是一个未初始化的全局变量，因此位于.bss节。该变量在文件中不占空间，而是在加载时被读入内存并预留指定大小的空间，这个就成了内存栈的位置。

#### 练习7&8

根据A64的[Calling Convention](https://en.wikipedia.org/wiki/Calling_convention)，r19-r28寄存器是callee-saved，如果callee需要使用它们，需要预先在栈上保存。

r29是frame pointer(相当于ebp)，r30是返回地址，它们需要被在栈上保存。

因此一次递归调用会压入3个8字节的值，但是栈指针其实每次会减少32，猜测是为了对齐到16字节？

此外，前8个参数会使用r0-r7寄存器传递，超过8个参数使用栈传递。

| 位置                      | 值                          |
| ------------------------- | --------------------------- |
| ...                       | more arguments              |
| old sp+8                  | argument 10                 |
| old sp                    | argument 9                  |
| old sp-8                  | rubbish(for alignment?)     |
| old sp-16                 | callee-saved register       |
| ...                       | n-1 callee -saved registers |
| old sp-(16+8*n)           | r30(return address)         |
| old sp-(24+8*n) = new r29 | r29(frame pointer)          |
| ...                       | stack grow downwards        |

#### 练习9

我认为从栈上取前五个参数是不合理的，参数都是用寄存器传递，栈上的值只是callee-saved register，其值只是“恰好”等于参数值而已。

