# 操作系统lab5实验报告

## 练习1

_加载应用程序并执行_

这部分只需要填充`proc.c`中的`load_icode`函数即可, 填充任务是完成对进程中断帧的填充, 包括CS, DS, ES, 以及栈顶和程序入口, 并修改eflags允许中断, 由于这是解析elf文件而开启进程(即内核外部的应用程序), 因此这里的段寄存器和栈顶用的都是用户态下的栈顶和段寄存器, 这一段比较简单, 因此与答案没有太大区别. 

_加载应用程序到开始执行的过程_

第一个用户态的线程, 是由`init_main`线程(即ucore中的第二个线程)创建的, 通过`kernel_thread`为`user_main`分配了一个新线程所需要的资源, 包括中断帧的简建立, 进程入口的设置, 以及栈空间分配, 页表拷贝, id申请, 将新创建的`proc_struct`加入链表当中, 之后就唤起进程, 等待被调度. 

`user_main`被调度之后, 就会执行`KERNEL_EXECVE(exit);`, 这是一个宏, 效果是指定一个叫做exit的应用程序进行执行, 最终会调用函数`kernel_execve(name, binary, (size_t)(size));`, 其中`name`就是该应用进程的名字, `binary`是应用程序的开始地址, `size`是应用程序的大小. 

`kernel_execve`会通过系统调用, 最终调用函数`do_execve`进行实际的执行, 在这部分中做得事情包括cr3寄存器的重新赋值, 内存管理引用计数更改, 名字设置, 以及调用函数`load_icode`(从elf文件中将获得信息, 具体功能见下. 

`load_icode`的功能, 包括创建新的内存管理, 进行内存映射, 为应用程序申请页面, 并拷贝代码段和初始与未初始数据段, 为进程创建可供使用的用户态堆栈, 最后修改中断帧, 在其中设置代码段入口. 

上述工作全部结束之后, 进程等待被调度即可, 至于被调度之后如何开始执行, 具体过程依赖于`proc`中的中断帧和上下文, 以及异常返回机制. 进程被选择调度之后, 会通过`proc_run`实际唤醒, 在其中会执行汇编函数`switch_to`, 在函数的最后, 会把被调度程序中`context.eip`压入到栈中然后返回, 这时, 程序就会跳到汇编函数`forkrets`, 这个函数会将`proc`中保存的`trapframe`取出作为参数, 然后跳到__trapret进行返回, 在返回过程当中, 自然就跳到了进程的代码入口(会取trapframe中的eip)

## 练习2

_父进程复制自己的内存空间给子进程_

练习2的编码部分, 只需要对`pmm.c`中的`copy_range`函数进行填充即可, 填充的内容只是把源进程中的页面中的内容拷贝到目标进程对应的页面中. 与答案的思路没有任何区别, 只是写法略有不同. 

```
memcpy(page2kva(npage), page2kva(page), PGSIZE);
ret = page_insert(to, npage, start, perm);
```

_设计实现”Copy on Write 机制“, 给出概要设计_

总体的思路是, 在copy_range函数当中, 不进行从npage到page的拷贝, 而是取消写权限后直接插入到目标进程中的页表里, 

```
ret = page_insert(to, page, start, perm & ~PTE_W);
```

因此, 只要进行读操作, 就没有任何问题, 当对其进行写操作时, 就会发生缺页异常. 

在`trap.c`, `do_pgfault`当中, 判断引发缺页的页表项(*pte)是否PTE_P为1, 如果不为1, 表示这时访问到了一个延迟拷贝的页, 这时做以下的操作

1. 重新申请一个页面

2. 将*pte指向的页面的内容拷贝到新申请的页面. 

3. 删除*pte, 插入新的页面. 

## 练习3

_理解进程执行 fork/exec/wait/exit 的实现, 以及系统调用的实现_

`exec`在lab5中没有被实现, 另外三个函数在`ulib.c`中实现, 他们分别调用了`sys_fork`, `sys_wait`, `sys_exit`三个函数(在`syscall.c`中实现, 其中还包括了函数`sys_exec`), 这几个函数分别调用了`do_fork`, `do_execve`, `do_wait`, `do_exit`四个函数调用用户态下的`syscall`, 这个函数通过内联汇编进行系统调用, 之后将控制权交给操作系统, 实际进行fork, exec, wait, exit的工作. 对于exec, 在本试验中还是采用宏的方式调用函数`kernel_execve`触发系统调用, 转到内核态下的`sys_exec`.

有关与系统调用, ucore中有三处比较关键, 第一处在`syscall.c`

```
static int (*syscalls[])(uint32_t arg[]) = {
    [SYS_exit]              sys_exit,
    [SYS_fork]              sys_fork,
    [SYS_wait]              sys_wait,
    [SYS_exec]              sys_exec,
    [SYS_yield]             sys_yield,
    [SYS_kill]              sys_kill,
    [SYS_getpid]            sys_getpid,
    [SYS_putc]              sys_putc,
    [SYS_pgdir]             sys_pgdir,
};

#define NUM_SYSCALLS        ((sizeof(syscalls)) / (sizeof(syscalls[0])))

void
syscall(void) {
    struct trapframe *tf = current->tf;
    uint32_t arg[5];
    int num = tf->tf_regs.reg_eax;
    if (num >= 0 && num < NUM_SYSCALLS) {
        if (syscalls[num] != NULL) {
            arg[0] = tf->tf_regs.reg_edx;
            arg[1] = tf->tf_regs.reg_ecx;
            arg[2] = tf->tf_regs.reg_ebx;
            arg[3] = tf->tf_regs.reg_edi;
            arg[4] = tf->tf_regs.reg_esi;
            tf->tf_regs.reg_eax = syscalls[num](arg);
            return ;
        }
    }
    print_trapframe(tf);
    panic("undefined syscall %d, pid = %d, name = %s.\n",
            num, current->pid, current->name);
}
```

这部分是实际的系统调用处理过程, 可以看到, 系统在调用`syscall`, 会将寄存器中的值(准确说是进入异常前的寄存器)取出, 选择`syscalls`(一个函数指针数组)中的一个函数指针, 传入参数, 然后调用. 

第二部分在`trap.c`中

```
trap_dispatch(struct trapframe *tf) {
    // .....
    case T_SYSCALL:
        syscall();
        break;
    case IRQ_OFFSET + IRQ_TIMER:
    // .....
}
```

可以发现, 对于系统调用异常, 操作系统会调用`syscall`这个函数. 

第三部分在`syscall.c`中, 这里作为一个进行系统调用的例子. 

```
static inline int
syscall(int num, ...) {
    va_list ap;
    va_start(ap, num);
    uint32_t a[MAX_ARGS];
    int i, ret;
    for (i = 0; i < MAX_ARGS; i ++) {
        a[i] = va_arg(ap, uint32_t);
    }
    va_end(ap);

    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL),
          "a" (num),
          "d" (a[0]),
          "c" (a[1]),
          "b" (a[2]),
          "D" (a[3]),
          "S" (a[4])
        : "cc", "memory");
    return ret;
}
```

因此ucore中系统调用的整个过程是: 内联汇编给出syscall命令 --> 进入异常, 操作系统根据中断号调用`syscall`函数 --> `syscall`根据寄存器中的值选择相应的异常处理例程. 

_fork/exec/wait/exit在实现中是如何影响进程的执行状态的?_

fork将进程的状态转为"可执行"状态, 并为进程分配资源, exec将进程装入真正要执行的内容, wait让进程检查自己的(特定或所有)孩子有没有处于"僵尸"状态, 如果有, 那么释放子进程资源, 如果没有, 那么就转入"睡眠状态", 并且将等待状态设置为"等待孩子", exit令进程进入"僵尸"状态, 并且唤醒父进程. 

_请给出ucore中一个用户态进程的执行状态生命周期图_

 在`proc.c`中给出了用户进程的执行状态生命周期图, 就直接贴在这里了
 
```
process state changing:

  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  +
```
 
 
## 重要的知识点

系统调用的实现方式, 在ucore中的实现方法, 其实是和原理课中所说的是一致的. 原理课中说, 系统调用的处理例程一般不合其他异常处理例程放在一起, 而是通过一个总的系统调用处理例程去调用特定的系统调用处理例程. 