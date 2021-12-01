### 实验1

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

### 实验2

1. `split_page`的`order`参数说明似乎写错了，应该是目标块的order。
2. 手册D5.3.1节可以找到页表中table项和block项的定义，D5.3.2节有L3 page项的定义，D5.3.3节有attribute部分的定义。

#### 问题1

bootloader应该是被读入由arch决定的固定位置，kernel编译时在`scripts/linker-aarch64.lds.in`里指定了各section的位置，bootloader按照elf里的信息加载。另外`mm_init`会在运行时具体设置`free_mem_start`及以上的地址空间，这个值是根据ld script里指定的`img_end`在运行时得到的，与内核镜像的大小有关。

#### 问题2

性能：上下文切换的时候内核页表一般不用改，相对于单一的页表寄存器基本没有性能损失。

安全：为进程/内核地址空间提供了更好的隔离（？）

#### 问题3

1. 物理地址
2. 虚拟地址，任何访存都要过mmu

#### 问题4

1. 4G内存共1M个页，每页需要8字节L3 entry，加上L0-L2的几个字节，大约是8MB。

   多级页表可以降低内存开销：因为如果一段内存没有被映射，页表中也需要存在一个空洞。如果只有一级页表，意味着每4KB就要有一个8字节的页表空洞；如果采用多级页表，可以直接在高级页表中留下这个空洞，大大节省了空间。

2. 对于连续的内存映射，AArch64可以用一个块而非逐页映射，不用完全走完四级地址翻译。

#### 问题5

​	使用哪个寄存器放内核页表应该只是内核设计者决定的，并没有硬件限制。这就意味着用户态进程也可以访问那个放内核页表的寄存器。所以在页表项里做访问控制是有必要的。

#### 问题6

1. 连续的内存映射可以用块，地址翻译层级更少。
2. 当前特权级如果低于页表项中指定的特权级，MMU抛出异常终止非法的内存访问。

### 实验3
1. 内联汇编不能忘记指定clobbered registers，否则编译器不知道用户弄脏了哪些寄存器。

#### 练习2

首先使用`ramdisk_read_file`读入root process对应的二进制文件，然后使用`process_create`创建process结构体，该函数有一些capability相关的操作，并会使用`vmspace_init`创建进程地址空间。

`process_create`返回后，开始执行`thread_create_main`创建进程中的第一个线程。该函数创建用户栈并在vmspace中进行虚拟地址映射，创建thread结构体，使用`prepare_env`准备用户栈上的初始状态（main的参数），使用`load_binary`读入elf文件到内存指定位置并设置pc。最后调用`thread_init`将stack，pc等信息装入线程内核栈。

#### 练习4

异常向量表中的`sync_el0_64`是系统调用的入口，它首先检查ESR_EL1的EC部分是否为ESR_EL1_EC_SVC_64（代表系统调用），如果是的话控制流来到`el0_syscall`，它首先进行一番先压栈在弹栈的操作（lab4中可以解释，这是在调用c函数`lock_kernel`），然后查syscall_table跳到对应的系统调用函数处。

#### 练习7

`START`是通过直接恢复上下文来调用的，返回地址是0。执行完`START`后PC变成了非法地址0从而触发Instruction Abort。

### 实验4

#### 练习1

通过mpidr_el1寄存器低8位的值判断是否为主CPU。

AP全部在wait_until_smp_enabled处自旋，直到BSP依次通过修改secondary_boot_flag唤醒它们为止。

#### 练习3

每个CPU有自己的kernel_stack和secondary_boot_flag和cpu_status，这是它们获取大内核锁前唯三使用的全局变量。所以应该不会出现data race。

#### 练习6

`lock_kernel`时需要保护寄存器是因为c函数夹在汇编中间，而且后面的汇编逻辑还依赖寄存器的值，所以理论上要保护caller saved registers。

`exception_return`时不保护是因为后面就从内存上恢复寄存器了，当前的值已经不重要。

#### 练习8

idle thread逻辑上是一个自旋线程，执行的并非内核代码，但却是运行在内核态。根据ESR寄存器的值判断异常来源的原理是异常是否来自用户态，在这里是一个特殊逻辑。忽视这个特殊逻辑不拿锁会导致unlock比lock多，这样本该拿到锁的lock调用会自旋，导致永远阻塞。

### 实验5

1. lab2里有祖传bug，映射内核的时候页表权限位设置不对，导致lab5中读入文件的时候calloc失败。
2. user/lib/liblauncher.c里的`fs_read()`展示了如何在shell里用ipc去调文件系统的接口。
3. 利用ipc调文件系统接口时要注意：文件系统进程并没有共享缓冲区的页表映射，要在server进程重新map_pmo。