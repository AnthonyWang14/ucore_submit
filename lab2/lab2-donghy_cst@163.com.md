# 操作系统lab2实验报告

## 练习1

_实现first-fit连续物理内存分配算法_

### 设计实现过程

ucore中的default_pmm.c中已经实现了简单的连续物理内存分配算法(下称default算法), 但还不是first-fit, 因此只要对其中的`default_alloc_pages`和`default_free_pages`进行修改, 使空闲块链表总是保持物理地址顺序即可. 

#### 分配的设计思路

对于连续物理内存的分配, 修改`default_alloc_pages`, 对于一次分配之后剩下的物理空间, 原来的default算法, 是将其直接插入到Page链表的最前端, 如果要实现first-fit, 将这部分物理空间插回到被申请的Page之后, 再将被申请的Page从链表中删掉即可. 

#### 可行的优化

在找到被分配的页面之后, 不要立即将其从链表中删去, 计算出剩余的物理空间后, 将其插入到被分配的页面的后面, 再将被分配的页面删除. 

#### 与答案的不同之处

感觉答案似乎不是很对, 之前对`page.flags`中`PROPERTY bit`的约定是说, 如果被置为1, 则表示这个页面空闲, 且是一段连续空闲空间的**首页**, 但在答案中似乎误解了这个约定, 另外, 答案中还滥用了`RESERVE bit`, 这部分是设置不可被交换的保存页, 不应该在分配时使用. 

#### 释放的分配思路

在`default_free_pages`中, default算法中已经实现了置位和引用相关的工作, 但将释放的空间插入到了空闲链表的头部, 因此只要对其进行修改即可. 暴力的算法是遍历链表, 合并物理空闲块, 找到合适的插入位置. 

#### 可行的优化

释放物理空间的时候, 需要在page_list中进行遍历, 查看是否存在需要相邻页面合并的情况, 但一次释放最多对应着两次合并, 分别是前面一个页面和后面的一个页面, 因此只需遍历到比释放页面物理地址更高的页面, 作合并判断后, 就可以停止遍历, 并且记下相应位置, 根据这个位置进行插入(就不用从头遍历).

#### 与答案不同的地方

这里答案的问题, 感觉还是和分配的时候相同, 就不再赘述. 

## 练习2

_实现寻找虚拟地址对应的页表项_

#### 设计实现过程

完成这个练习的任务, 只需要填写`pmm.c`中的`get_pte`即可, 在设计之前, 需要知道`mmu.h`当中已经定义了`PDX`, `PTX`, 两个宏运算, 可以用于取出物理地址中的高10位和中间10位, 以及`KADDR`宏, 可以将物理地址转换成内核中的虚拟地址. 

要获得某个线性地址`la`对应的`pte`, 只需要通过`(pte_t *)(KADDR(ROUNDDOWN((uintptr_t)pde, PGSIZE))) + PTX(la)`即可(ROUNDOWN(..., PGSIZE)是说将某数按照PGSIZE大小向下取整). 

但要注意`get_pte`的语义还要求, 如果其参数`create`为1, 则需要在请求页表项对应的页面不存在的时候, 为页表项申请一个页面, 申请之后需要对相应的页面进行引用次数, 空间初始化, 填写页表项三方面的工作. 

```
set_page_ref(p, 1);
uintptr_t pg_pad = page2pa(p);
memset(KADDR(pg_pad), 0, PGSIZE);
pgdir[PDX(la)] = pg_pad | PTE_P | PTE_U | PTE_W;
```

需要说明的是, pg_pad是申请页面的首地址, 它一定是按页对齐的. 

#### 与答案的区别

思路基本一致, 但答案的写法稍微简洁一些. 

_请描述页目录项（Pag Director Entry）和页表项（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处_

页目录项是指向储存页表的页面的, 所以本质上与页表项相同, 结构也应该相同. 每个页表项的高20位, 就是该页表项指向的物理页面的首地址的高20位(当然物理页面首地址的低12位全为零), 而每个页表项的低12为, 则是一些功能位, 可以通过在`mmu.h`中的一组宏定义发现. 

```
#define PTE_P           0x001                   // Present 对应物理页面是否存在
#define PTE_W           0x002                   // Writeable 对应物理页面是否可写
#define PTE_U           0x004                   // User 对应物理页面用户态是否可以访问
#define PTE_PWT         0x008                   // Write-Through 对应物理页面在写入时是否写透(即向更低级储存设备写入)
#define PTE_PCD         0x010                   // Cache-Disable 对应物理页面是否能被放入高速缓存
#define PTE_A           0x020                   // Accessed 对应物理页面是否被访问
#define PTE_D           0x040                   // Dirty 对应物理页面是否被写入
#define PTE_PS          0x080                   // Page Size 对应物理页面的页面大小
#define PTE_MBZ         0x180                   // Bits must be zero 必须为零的部分
#define PTE_AVAIL       0xE00                   // Available for software use 用户可自定义的部分
```

PTE_P, PTE_W, PTE_U在ucore中已经用得很多, 就不再赘述, PTE_A和PTE_D可以用于虚拟内存管理和页面的置换算法, PTE_PWT和PTE_PCD可以用于影响cache的行为. 

_如果ucore执行过程中访问内存, 出现了页访问异常, 请问硬件要做哪些事情_

出现了页访问异常, 硬件会将eflags, cs寄存器的值, eip的值, 错误码压入栈中, 如果涉及到了用户态到内核态的切换, 那么还会在压入前面几个值之前, 更换一个栈, 并且压入ss的值和esp的值. 

另外, 会将引起异常的访问地址存在cr2寄存器当中. 

以上结论来自ucore的`trap.c`文件中. 

## 练习3

_释放某虚地址所在的页并取消对应二级页表项的映射_

#### 设计思路

这部分只需修改`pmm.c`中的`page_remove_pte`即可, 使用`pte2page`就可以从页表项得到对应的物理页面所对应的Page结构体. 得到该结构体之后, 需要对页面引用进行更改, 如果引用降到零, 那么就需要释放该页, 另外由于映射关系的改变, 因此还需要重填tlb表. 

#### 与答案的区别

基本相同, 答案稍简介, 因为我没有注意到`page_ref_dec`的返回值就是变化后的引用数值. 

_数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系? 如果有, 其对应关系是啥?_

page结构体是物理页面的管理者, 因此和系统中的物理页面是一一对应的, 因此其排列顺序也是按照其对应的物理页面地址进行的, 所以可以使用`pte2page`函数从页表项找到相应的物理页面的Page结构体, `pte2page`实现如下: 

```
#define PTE_ADDR(pte)   ((uintptr_t)(pte) & ~0xFFF)

#define PPN(la) (((uintptr_t)(la)) >> PTXSHIFT)

pa2page(uintptr_t pa) {
    if (PPN(pa) >= npage) {
        panic("pa2page called with invalid pa");
    }
    return &pages[PPN(pa)];
}

static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}
``` 

可以看出page结构体和对应的物理页面的首地址是一一对应的, 换句话说, 第X个物理页面, 正好由pages[i]管理, 因此从pte到对应的page结构体.  

_如果希望虚拟地址与物理地址相等, 则需要如何修改lab2, 完成此事? _

更改KERNBASE, 从0xC0000000改到0x00000000, 另外, 段表应该做相应的修改, 链接的时候, 虚拟地址也应该有0xC0100000改到0x00100000. 

----- 

## 本实验中重要的知识点

模块化的编程思想, C中没有类的概念, 但可以通过函数指针和结构体来模仿类与实例的关系, 例如这次实验中的`pmm_manager`和`default_pmm_manager`. 
