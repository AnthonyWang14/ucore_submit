# lab1 实验报告

2013011356 董豪宇

非编程题, 使用`.../labcodes_answer/lab1_result`中的代码进行实验

编程题使用`.../labcodes/lab1`中的代码进行实验

## 练习1

### 第一题

_操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)?_

在这里使用中的代码进行实验.

通过执行命令`make "V="`, 可以发现`ucore.img`由两部分构成, 分别是bootblock(也就是bootloader)和kernel, 
另外, 在整个过程中还需要`.../bin/sign`的协助. 
这两者的生成方式都遵循类似的过程: 先通过`gcc`将依赖的文件编译成`.o`文件.

`gcc`用到了这些参数:

* `-I`: 指定寻找头文件的路径

* `-fno-builtin`: 表示不使用C语言的内建函数

* `-Wall`: 表示打开所有警告

*  `-ggdb`: 表示编译时为gdb生成调试信息

* `-m32`: 表示将程序编译为32位程序

* `-gstab`: 产生不需要gdb扩展支持的调试信息

* `-nostdinc`: 表示不搜寻系统中的头文件, 除非用`-I`显式指定

* `-fno-stack-protector`: 不对栈进行保护(主要指溢出保护)

* `-Os`: 一个优化选项, 使用了`-O2`的所有优化项目, 但是不增加代码的尺寸. 

* `-c`: 指只编译不连接

* `-o`: 目标文件

之后, 再通过链接工具`ld`将需要的`.o`文件链接起来生成bootblock和kernel, 
不同之处在于bootblock被生成之后, 还需要经过`.../bin/sign`(`.../bin/sign`由`.../tools/sign.c`编译而成)的加工, 使之符合硬盘主引导扇区的特征. 

`ld`用到了这些参数: 

* `-m`: 个人理解应该是指定平台, 根据文档中的说法, 应该是模拟相应平台的连接

* `-N`: 令代码段可读可写, 且令数据段不用进行页对齐

* `-e`: 显式的指定入口符号

* `-Ttext`:指定代码段的起始位置, 类似的还有`-Tbss`, `-Tdata`

* `-o`: 指定输出文件

* `-nostdlib`: 只在命令行中显示指定的文件夹中搜寻链接库

得到bootblock和kernel文件之后, 通过`dd`工具将两者写到ucore.img当中,
需要注意的是, bootblock需要写到第一个块里, kernel写在之后, 这里一个block的大小是512B.

`dd`用到的参数有: 

* `if`: 表示文件输入流(默认为stdin)

* `of`: 表示文件输出流(默认为stdout)

* `count=num`: 表示写入num个block

* `seek=num`: 表示跳过num个block(在输出端)再写入

* `conv=notrunc`: 指在输出过程当中不截断输出文件. 

### 第二题

_一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？_

从`.../tools/sign.c`的源代码中可以看出, 要符合规范, 只需要具备两个特征:

1. 长度正好为512字节

2. 最后两个字节一定为`55AA`(16进制)

## 练习2

1. 从CPU加电后执行的第一条指令开始, 单步跟踪BIOS的执行. 
2. 在初始化位置0x7c00设置实地址断点,测试断点正常. 
3. 从0x7c00开始跟踪代码运行, 将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较. 
4. 自己找一个bootloader或内核中的代码位置, 设置断点并进行测试. 

使用指令: 

```
gnome-terminal -e "qemu-system-i386 -S -s -d in_asm -D bin/q.log -monitor stdio -hda bin/ucore.img -serial null"
```

在新的终端中打开qemu, 其中`-s`就是`-gdb tcp::1234`的缩写, 表示启动gdbserver并在端口1234上进行监听. 

之后, 通过`gdb -q`以安静模式进入gdb, 依次执行以下命令: 

```
file bin/kernel
target remote :1234
b *0x7c00
c
si
....
```

即可在在刚进入BIOS的位置下断点, 并通过`si`进行汇编代码级别的单步执行, 结果请见lab1_1.png

从结果上来看, 测试断点正常, 以下是通过gdb得到的代码: 

```
cld    
xor    %eax,%eax
mov    %eax,%ds
mov    %eax,%es
mov    %eax,%ss
in     $0x64,%al
test   $0x2,%al
jne    0x7c0a
mov    $0xd1,%al
out    %al,$0x64
in     $0x64,%al
test   $0x2,%al
jne    0x7c14
mov    $0xdf,%al
out    %al,$0x60
lgdtl  (%esi)
mov    %cr0,%eax
or     $0x1,%ax
mov    %eax,%cr0
ljmp   $0xb866,$0x87c32
mov    $0x10,%ax
mov    %eax,%ds
mov    %eax,%es
mov    %eax,%fs
mov    %eax,%gs
mov    %eax,%ss
mov    $0x0,%ebp
mov    $0x7c00,%esp
call   0x7d00
# 之后跳到bootmain中
```

最后一句汇编指令陷入指向自身的死循环, 与bootasm.S和bootblock.asm进行比较, 反汇编出的代码, 没有了函数结构和标签, 其余均相同. 

之后在0x7c02处下了断点, 重做了测试, 与之前的结果大致相同, 结果就不在此贴出. 

## 练习3

_为何开启A20，以及如何开启A20_

A20事实上是向下兼容的产物, 在intel早期的CPU中有20位地址线, 提供1MB的地址空间, 然而CPU却只有16位, 为了充分理由20位总线, 采用了将段寄存器的值向左移四位再与偏移量相加的方法, 但这样寻址的范围就超过了1MB(大约为1088KB), 因此对超出的部分进行了"回卷", 之后的CPU总线超过了20位, 意味着可以访问1M以上的地址空间了, 即不会发生"回卷", 这就造成了向下不兼容, 因此才有了A20用于控制是否"回卷". 

如何开启A20, 涉及到对8042键盘控制器的操作, 在`bootasm.S`中有以下一段代码进行控制(各句代码的意思在代码中以注释的形式给出): 

```
seta20.1:
    inb $0x64, %al                                  # 读状态寄存器
    testb $0x2, %al								    # 判断input buffer中是否有数据
    jnz seta20.1								    # 如果有就继续检查

    movb $0xd1, %al                                 # 0xd1 -> port 0x64 (这时 input buffer 已空)
    outb %al, $0x64                                 # 向0x64写入0xd1, 表示要向output port写入数据

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2								    # 等待 input buffer 为空

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

_如何初始化GDT表_

在bootasm.S当中, 有关初始化GDT表的代码如下: 

```
lgdt gdtdesc
......
gdt:
    SEG_NULLASM                                                  # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)                        # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                              # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                    				 # sizeof(gdt) - 1
    .long gdt                                     				 # address gdt
```

指令`lgdt gdtdesc`用于初始化GDT表, 并将段表基址存在一个特殊的寄存器GDTR中, `gdtdesc`是段表的描述, 包括段表地址和大小, 段表部分在`gdt:`后, 可以看见, ucore中规定了三个段, 一个空段, 一个数据段, 一个代码段. 

_如何使能和进入保护模式_

进入保护模式, 是通过将一个特殊的寄存器CR0的最低位置为1, 在bootasm.S中的代码如下: 

```
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0
```

## 练习4

_bootloader如何读取硬盘扇区的_

在bootmain.c中, 与实际读取硬盘扇区有关的代码如下: 

```
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* do nothing */;
}

/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) { // dst是目的内存的地址, secno是扇区号
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

因此, 读取扇区的大致过程是, 等待磁盘空闲, 将要读出的扇区个数和扇区号(最高位bit抹零)以及读命令写到相应的地址, 再等待磁盘空闲, 然后从特定磁盘地址将一个扇区的内容读到相应的内存中. 

_bootloader是如何加载ELF格式的OS_

在bootmain.c中, 与加载OS相关的代码如下: 

```
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

大概过程是, 从1号扇区开始(磁盘从零号扇区开始)读连续的8个扇区, 将内容读到ELFHDR中, 通过检查ELFHDR中的一个magic number是否符合预期来判断正确读入了ELF. 

之后将代码段和数据段读进相应地址, 之后跳到`ELFHDR->e_entry`, bootloader就完成了自己的工作. 

## 练习5

这里的`ebp`中存着`ebp`的地址, `ebp+4`存着caller调用时的`eip`, 之后的是函数的参数(如果有), 最后一行是调用栈中的最底层的函数调用. 

编程部分与答案的区别在于, 答案中没有将系统调用填充为陷入门, 也没有针对系统调用更改DPL. 

## 练习6

_中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？_

一个表项占8个字节(2*32bit), 第1, 2个字节是offset的低16位, 第7, 8字节是offset的高16位, 第3, 4个字节是段选择子, 只要知道段选择子和偏移量, 就能通过段表得到中断处理代码的入口. 

_编程题_

请见代码, 与答案对照后, 觉得和答案的实现没有太大区别. 

## challenge 1

### 有关于内核态和用户态的相互切换. 

系统进入异常时, 如果是内核态到内核态, 那么硬件不会压入ss和eip, 如果是用户态到内核态, 那么不仅会压入ss和eip, 而且会建立一个新的栈用于处理异常. 

### 解题思路

#### 内核态到用户态

若直接调用相应的`syscall`, 则操作系统正常进入异常处理流程, 而如果要到用户态, 关键在于执行`iret`的时候, 能让CPU误以为当前的栈是由用户态到内核态转换的时候生成的, 因此, 在异常处理流程当中, 需要对四个段寄存器和eflags进行赋值(这部分容易想到), 然而两种栈的"尺寸"不同, 因此在执行系统调用之前, 需要将栈空出2个32bit的空间, 用于存放"人造的"`ss`和`esp`, 另外, `esp`的值根据计算, 应该是`tf + 2 * sizeof(uint32_t) + sizeof(struct trapframe)`(就是刚进入`lab1_switch_to_user`函数时的`esp`值).

#### 用户态到内核态

用户态到内核态时, 会在栈中多压入`ss`和`esp`, 要返回内核态, 就需要欺骗CPU, 让其误以为是从内核态进入内核态, 首先需要将段寄存器和`eflags`进行值的修改, 这样就会像返回内核态一样返回, 但由于CPU误以为返回到内核态, 因此在iret的时候, esp仍然指向CPU认为的"内核栈"上, 因此在系统调用结束后, 需要将`esp`进行恢复, 恢复的方法就是在`lab1_switch_to_kernel`系统调用结束之后, 将`ebp`赋值给`esp`, 这样在函数返回时, `esp`的值自然就被修正. 

#### 与答案不同的地方

最大的不同在于, 答案中似乎使用了一块新的栈空间, 但我的方法是通过提前操作栈来"欺骗"CPU.  





