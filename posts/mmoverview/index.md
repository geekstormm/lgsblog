# Linux内存子系统




# 系统物理内存初始化

## 1.0 关于内存的一些问题

-   内核通过什么手段来识别可用内存硬件范围的
-   内核管理物理内存用了哪些技术手段
-   为什么free -m显示的总内存总比dmicode输出的要少，少了的内存去哪里了
-   内核是如何知道某个内存地址范围属于那个NUMA的呢

&gt;   NUMA (Non-Uniform Memory Access，非统一内存访问) 是一种内存设计架构，特点是： - 处理器可以访问自己的本地内存比访问非本地内存(远程内存)快 - 每个处理器都有自己的本地内存 - 处理器和内存被组织成&#34;节点&#34;(Node)

## 1.1 从固件开始

将内存放到硬件层面，本质上就是带有一根根触角的硬件。操作系统需要识别到这些内存才能进行使用。那么操作系统如何知道内存等硬件信息的呢？答案是固件，也就是我们常说的UEFI/BIOS，固件是操作系统和硬件的中间层，在硬件上电后，在操作系统启动之前，实际上最先启动的是固件程序，固件会进行硬件自检，初始化硬件程序，电源功能管理等功能，同时操作系统需要用到的内存信息也是通过固件来获取的。

在固件启动前，硬件会做上电，处理器复位，内存控制器初始化，芯片组初始化等操作。

&gt;   **UEFI（Unified Extensible Firmware Interface）: BIOS的继任者，更现代化**
&gt;
&gt;   **BIOS（Basic Input/Output System）**

固件的职责从上电到启动依次包括：

-   POST（Power-on Self-test），上电，自检，如CPU，内存，外设等。
-   硬件初始化
-   设备枚举以及初始化
-   ACPI表（**Advanced Configuration and Power Interface，高级配置与电源接口**）创建（根据设备资源生成ACPI表）
-   引导设备准备（通常是硬盘、固态硬盘或网络启动设备），UEFI架构中包括找到Bootloader（比如GRUB）
-   向Bootloader转交控制权

## 1.2 物理内存安装检测

在操作系统启动后，会去探测可用的内存范围，具体操作是，前面固件的ACPI表提供了内存探测的物理分布规范，内核设置中断号为15H，并设置操作码为E820 H。然后就会执行detect_memory操作。

detect_memory会将可用内存信息保存到`boot_params.e820_table` 对象中，内核后续会将`boot_params.e820_table` 中的信息存到全局变量`e820_table`中（通过`e820_memory_setup`函数）。

内核启动过程中的输出信息可以通过dmesg命令来查看。

## 1.3 memblock内存分配器

内存分配器分为两种，一种是刚启动时的初期分配器，这个内存管理粒度大，只是为了满足系统启动时对内存页的简单管理。另一种是系统启动后的伙伴系统（Buddy System）。

上面说到调用完`e820_memory_setup`函数，可用的内存信息就已经存到了全局对象`e820_table`中。随后调用`e820_memblock_setup`创建memblock内存分配器。

memblock的逻辑很简单，就是按照检测到的地址范围是usable还是reserved，然后分别用一个memblock_region数组存起来。具体的过程就是遍历`e820_table` ，然后类型存储到对应的数组中。

memblock创建完成后会有一个`memblock_dump_all`的函数来一次打印所有信息，前提是打开了debug参数。

```C&#43;&#43;
void __init_memblock memblock_dump_all(void)
{
    if (memblock_debug)
        __memblock_dump_all();
}

static void __init_memblock __memblock_dump_all(void)
{
    pr_info(&#34;MEMBLOCK configuration:\n&#34;);
    pr_info(&#34; memory size = %pa reserved size = %pa\n&#34;,
        &amp;memblock.memory.total_size,
        &amp;memblock.reserved.total_size);

    memblock_dump(&amp;memblock.memory);
    memblock_dump(&amp;memblock.reserved);
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
    memblock_dump(&amp;physmem);
#endif
}
```

## 1.4 向memblock申请内存

在伙伴系统创建之前，所有的内存都是通过memblock来分配的，这里面有两个重要的场景，一个是crash kernel内存申请，一个是页管理初始化。

### Crash Kernel 内存申请

首先要清楚，crash kernel不是program kernel。crash kernel的设计是为了kernel在崩溃时可以记录崩溃现场，方便后续排查分析。crash kernel基于kdump机制，在内核崩溃时，kdump会使用kexec来启动第二个内核，这样就可以保留第一个内核内存以及其运行状态。

### 页管理初始化

Linux的伙伴系统是按页来管理内存的，页的大小是4kb，前面我们知道memblock管理的内存粒度很大，那么内核是如何将memblock的内存转化成页来管理的呢。

在kernel中有一个页管理模型，具体初始化函数是`paging_init`，在这个函数中为所有的page都申请了一个struct page对象，通过这个对象对页进行管理。

现代页管理模型都是SPARSEMEM模型，也就是通过二维数据对页进行管理，SPARESEMEM模型大致组成如下：

```C&#43;&#43;
// 第一维数组
struct mem_section *mem_section[NR_SECTION_ROOTS];

// 第二维数组
struct mem_section {
    unsigned long section_mem_map;
    unsigned long *pageblock_flags;
};
```

具体来说，mem_section数组用于索引内存根目录，大小为NR_SECTION_ROOTS，每个元素指向一组section。实际的section数组每个section管理一段连续的物理内存，通常每个section大小为128MB，使用section_mem_map指向实际的页面。

而每一个struct page的大小一般为64字节，而每一个页是4KB，64/4096大致为1.6%，那么16GB的内存就会有大约256M的内存需要用来管理页。

## 1.5 NUMA

在继续内存管理之前，需要先了解一下NUMA。（Non-Uniform Memory Access，非统一内存访问）是一种内存设计架构，将处理器和内存组织成多个节点（Node）。每个Node包含CPU，本地内存，I/O设备。在内核的实现中，每一个Node用`pglist_data`结构体表示。

```C&#43;&#43;
//Linux6.8.8 mmzone.h
typedef struct pglist_data {
    // 该节点包含的所有zone
    struct zone node_zones[MAX_NR_ZONES];
    
    // 节点的zonelist，用于内存分配失败时的备用列表
    struct zonelist node_zonelists[MAX_ZONELISTS];
    
    // 节点ID
    int node_id;
    
    // 节点中zone的数量
    int nr_zones;
    
    // 节点的页面映射
    struct page *node_mem_map;
    
    // 节点的起始页帧号
    unsigned long node_start_pfn;
    
    // 实际存在的页面数
    unsigned long node_present_pages;
    
    // 节点跨越的总页面数（包括空洞）
    unsigned long node_spanned_pages;
    
    // 节点状态标志
    unsigned long node_flags;
} pg_data_t;
```

这样做的原因是，随着计算机的发展，单个机器的CPU和内存越来越多，而CPU是有缓存一致性（Cache Coherency）的限制。在传统的SMP（对称多处理器）架构中，所有CPU共享同一个内存总线，当CPU数量增加时会导致总线竞争，缓存一致性开销和CPU扩展性限制。NUMA的出现使得这三个问题都得以解决。

这种架构的好处是CPU访问同一个Node的时候延迟低，带宽高，也就是说本地内存访问快，远程内存访问慢，一般而言，本地内存访问在100ns左右，跨Node访问则在300ns左右。

那么内核在启动的时候如何知道NUMA信息呢，答案是ACPI表，ACPI表中存在两个表，分别是SRAT（System Resource Affinity Table）和SLIT（System Locality Information Table），这两个表描述了NUMA拓扑。在操作系统启动的时候会去读这两个表，然后把NUMA信息存到numa_meminfo这个全局的结构体中。

```C&#43;&#43;
// Linux6.8.8 numa.h
struct numa_meminfo {
    int         nr_blks;
    struct numa_memblk  blk[NR_NODE_MEMBLKS];
};
```

到这里为止内核创建好了memblock分配器，也通过ACPI表知道了NUMA信息。接下来就是把NUMA信息和memblock关联起来。简单来说就是为每一个Node申请一个前面说明的pglist_data内核对象，并将memblock region与Node关联起来。

# 伙伴系统

前面说过内核启动后会通过ACPI表中的NUMA信息为每个Node创建一个内核对象。但是内核不可能这么简单的来管理每个Node，毕竟每个Node的内存是相当大的。因此内核为了更好的管理Node，会在每个Node下创建Zone，Zone的类型分为三种：

-   ZONE_DMA：支持ISA设备DMA访问
-   ZONE_DMA32：在64位系统中支持32位设备DMA访问
-   ZONE_NORMAL：其余的全部都是NORMAL

在每个zone下都有很多个page，也就是页，Linux下page大小一般是4KB。

可以通过`cat /proc/zoneinfo` 来查看机器具体的zone划分。

那么zone那么多的page是如何进行管理的呢，就是本节的标题，伙伴系统（Buddy System）。首先来看zone中内核的实现：

```C&#43;&#43;
// Linux 6.8.8 include/linux/mmzone.h
struct zone {
    /* Read-mostly fields */
    char *name;
    unsigned long zone_start_pfn;
    atomic_long_t managed_pages;
    unsigned long spanned_pages;
    unsigned long present_pages;

    /* free areas of different sizes */
    struct free_area free_area[MAX_ORDER];
    
    /* zone flags, see below */
    unsigned long flags;

    /* Write-intensive fields used from the page allocator */
    spinlock_t lock;
    // ... 更多字段 ...
};
```

注意到zone中有一个结构体数据free_area，这个就是伙伴系统的核心数据结构。到这里我们就可以大致知道内核的内存架构层次了：

```C&#43;&#43;
Node (NUMA节点)                            ｜       struct pglist_data
    ↓                                     ｜
Zone (内存区域：DMA、NORMAL等)               ｜       struct zone
    ↓                                     ｜
Buddy System (伙伴系统，包含free_area等)     ｜        struct free_area
    ↓                                     ｜
Page (物理页面)                             ｜
```

Node是NUMA中的概念，每个Node都有多个zone，伙伴系统在zone的层面工作，每个zone都有自己的伙伴系统，也就是free_area数组。Node是NUMA下最大的内存单元，伙伴系统是每个zone中实现的内存管理机制，Node主要用于NUMA系统中的内存单元管理，具体的页面管理由伙伴系统负责。

free_area实际是一个数组，这个数组有`MAX_PAGE_ORDER &#43; 1`个元素，也就是11个元素，元素依次管理是4KB、8KB、16KB的连续内存块。

实际分配页面时的工作流：

```C&#43;&#43;
// 分配内存时的层次调用
alloc_pages()
    → get_page_from_freelist()  // 在zonelist中查找合适的zone
        → __rmqueue()           // 在zone的伙伴系统中分配页面
```

实际`alloc_pages` 执行的操作是就是上学时操作系统教材上说的那一套策略了，比如相邻的合并，没有会向上级申请并裂变。现在的Linux Kernel在细节上做了很多优化，比如延迟合并，保持热页面，迁移类型隔离等等。

最后memblock会将内存完全交接给伙伴系统。

# 进程如何使用内存

## 3.0 虚拟内存背景

虚拟内存这个概念最早出现于论文One-Level Storage System，是上世纪70年代的论文，比Linux早很多。当时的背景下内存容量十分有限并且非常昂贵。程序稍微一大就会受到内存限制，以至于不能一次性在内存中完整运行。当时对于内存的需求远超硬件能提供的容量。传统的做法是程序员手动实现overlay（覆页），或者操作系统粗糙的进行内存与磁盘的调度，这种方式工作量大并且效率低下。在这样的背景下如何在有限的内存下运行大程序成为了热门话题。

虚拟内存的出现让机器可以运行远远大于内存的程序，因为程序通过虚拟内存会认为自己拥有从0开始的一大片地址，其实背后都是虚拟内存在调度，一般来说程序只会在物理内存中保留一些常用的页。

正是这种设计不仅使得程序设计变得简单（程序员不再需要具体为内存不足或者地址冲突等问题费神），对于系统的吞吐量以及后面的内存保护以及进程隔离等机制奠定了基础。

## 3.1 虚拟内存与物理页

在Linux Kernel中，虚拟内存的管理是以进程为单位的，每个进程都有一个虚拟地址空间。具体到实现上，每个进程的`task_struct`都有一个`mm_struct`类型的核心对象mm，它表示的就是进程的虚拟地址空间。

```C&#43;&#43;
// Linux6.8.8 include/linux/sched.h
struct task_struct {
// ...
    struct mm_struct       *mm;
    struct mm_struct        *active_mm;
// ...
};
```

在Linux6.1的版本之前，`mm_struct`用`vm_area_struct`（核心是红黑树）来管理VMA（Virtual Memory Area）。

vm_start 和 vm_end表示虚拟地址范围的开始和结束。进程中会有很多个vm_area_struct对象，每个vm_area_struct对象都会有自己的vm_start和vm_end，加起来就是对整个虚拟地址空间的占用情况。而因为进程中的vm_area_struct太多，在内存访问的时候进程需要查vm_area_struct和虚拟地址的对应关系，就需要一种高效的数据架构来进行管理以便进行遍历和查询，这就引入了红黑树&#43;双链表。红黑树可以快速查找特定地址的VMA，引入双链表的目的是加速遍历以及找到相邻的VMA（合并时很重要）。

```C&#43;&#43;
struct mm_struct{
// Linux 6.1以前
    struct rb_root mm_rb;        // 红黑树根
    struct vm_area_struct *mmap; // 按地址排序的双向链表头
}

struct vm_area_struct {
    struct mm_struct *vm_mm;     // 指向所属mm_struct
    struct rb_node vm_rb;        // 红黑树节点
    struct list_head vm_next;    // 链表节点
    // ...
};
```

在现在的版本中，使用了`maple_tree`来管理VMA（核心是线段树）。主要的原因是服务器上的核现在越来越多，线程页越来越多，多线程之间的锁竞争越来越严重，而红黑树的每次操作都需要加锁，因为涉及到树的平衡，并且还需要同步双向链表，基于这两个原因，就必须加锁。而`maple_tree`基于RCU（read-copy-update）实现了线程安全的无锁数据结构，大大降低了锁的开销。

并且maple_tree因为基于线段树，因此能提供范围查询，高效的相邻节点访问以及双向遍历。等于用一个数据结构完成了两个组合数据结构的功能。

```C&#43;&#43;
struct mm_struct {
    struct maple_tree mm_mt;        // 使用maple tree存储VMA
    // ...
};

// ma_root指针存储和管理vma
struct maple_tree {
    union {
        spinlock_t  ma_lock;
        lockdep_map_p   ma_external_lock;
    };
    unsigned int    ma_flags;
    void __rcu      *ma_root;
};
```

### 3.1.1 缺页中断

用户进程申请内存的时候，申请的只是一个vm_area_struct，也就是说只是一段地址范围。不是实际分配物理内存，具体的分配发生在实际访问的时候。进程运行时，需要访问变量，如果物理页还未分配就会触发缺页中断。

在 x86_64 Linux 内核中，触发缺页异常 (#PF) 后，最终会调用到 do_page_fault()，而 do_page_fault() 里又会根据具体情况（如发生在用户态还是内核态，是否合法地址等）调用到 do_user_addr_fault() 来进一步处理用户态的缺页异常。在Linux6.8.8中的具体处理如下，不同于6.1以前的红黑树，在这里查找也是用的线段树。

```C&#43;&#43;
// Linux 6.8.8 arch/x86/mm/fault.c
static inline
void do_user_addr_fault(struct pt_regs *regs,unsigned long error_code, unsigned long address) {
    // ...
    vma = lock_vma_under_rcu(mm, address);
    fault = handle_mm_fault(vma, address, flags | FAULT_FLAG_VMA_LOCK, regs);
    // ...
}
```

在找到vma后，内核会调用handle_mm_fault，随后通过__handle_mm_fault来完成真正的物理内存申请。

```C&#43;&#43;
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
               unsigned int flags, struct pt_regs *regs) {
    ret = __handle_mm_fault(vma, address, flags);
}
```

在__handle_mm_fault中会先去检测四级页表，如果不存在就申请创建，页表完整后再去调用

```
-&gt;handle_pte_fault
-&gt; do_pte_missing(vmf);
```

 `-&gt;do_anonymous_page(vmf);`

`do_anonymous_page(vmf)`的内存实际来源于伙伴系统，到此为止完成真正的内存分配。

## 虚拟内存使用方式

到这里可以从可执行文件开始到进程运行时捋一下：在进程加载的过程中，解析完ELF文件后（加载完成后实际上就已经初始化了），将为进程创建一个新的地址空间，以及一个4KB大小的栈；随后将可执行文件以及所依赖的库通过elf_map映射到虚拟内存中；最后还会进行堆区的初始化。

展开来说，所有的行为都集中在`load_elf_binary()`中，`load_elf_binary()` 读取 ELF 头及 Program Header Table (PHT)，针对每个 PT_LOAD 段，调用内核的 `mmap()`（底层是 `do_mmap()`) 来映射到指定虚拟地址：

-   **创建 VMA**：`do_mmap()` 会分配一个新的 `vm_area_struct`，设置 `vm_start`, `vm_end`, `vm_page_prot`（读、写、执行权限），以及对应的 `vm_file` 等信息；
-   **将这个 VMA** 挂到当前进程的 `mm-&gt;mmap` 链表或红黑树里，并更新 `mm-&gt;map_count`, `total_vm` 等统计；
-   **目的**：将 ELF 中的 .text 段（代码，只读可执行），.rodata 段（只读数据），.data 段（可写数据），.bss（零初始化段）等都按需映射到相应的虚拟地址。
-   如果是动态链接的 ELF，则还会额外处理 PT_INTERP 段，去加载动态链接器本身（它也是一个 ELF），然后再为它映射对应的区域。
-   当各段加载好以后，内核还要为进程分配并设置**用户态栈**：调用 `setup_arg_pages()` 等函数，为 argv[], envp[] 以及 auxv[] 分配空间，写入到用户态栈中；
-   同时记录下新进程栈的顶部地址，比如 mm-&gt;start_stack；

### execve的行为

execve的原型为：

```C&#43;&#43;
int execve(const char *filename, char *const argv[], char *const envp[]);
```

在用户执行execve的时候，会触发如下的路径：

1.  **用户态**

通过软中断（或 syscall 指令等）进入内核态，调用内核提供的 __do_sys_execve()。

1.  **内核态**

-   __do_sys_execve()

这是 syscall handler，读取用户传入的参数（文件名、argv、envp），然后调用：

-   do_execve()

进行一些初步检查后，最终调用：

-   do_execveat_common()

这里是更通用的实现，处理路径解析、flags 等，最终打开要执行的文件，形成一个 file * 结构，再调用：

-   do_execve_file()

给“待执行文件”创建并初始化 linux_binprm 结构（简称 bprm，存放将要被加载执行的程序相关的信息，如命令行参数、环境变量、打开的文件指针等），随后调用：

-   exec_binprm()

这是加载用户程序的核心入口函数，内部会调用注册在系统中的各类二进制格式处理器（例如 ELF, script, a.out 等）。在 Linux 中最常见的就是 ELF 格式处理器，也就是 load_elf_binary()。

其调用逻辑类似：

```C&#43;&#43;
int exec_binprm(struct linux_binprm *bprm) {
    return search_binary_handler(bprm);
}
```

-   **load_elf_binary()**
    -   **检查 ELF Header**
    -   **初始化新进程的内存描述，ELF 解析通过后，会先对原有进程进行“替换式”加载，也就是说，需要废弃旧内存布局，准备建立新内存布局。**
    -   **解析 Program Header Table，ELF 文件中的 Program Header Table 指示可执行文件中各段（Segment）的加载方式（可执行、可写、可读等），其结构为 struct elf_phdr。load_elf_binary() 会逐个解析 p_type == PT_LOAD 的段，然后进行相应的 mmap() 操作，将这些段映射到目标进程的虚拟地址空间中。**
    -   **设置进程入口点**
    -   **处理解释器段（动态链接器）**
    -   **建立用户栈**
    -   **设置进程各项元数据**
    -   **完成加载 &amp; 返回**

可以简化为：

```C&#43;&#43;
// 用户态调用
execve(filename, argv, envp);

// 进入内核态（SYSCALL）
__do_sys_execve()
  -&gt; do_execve(filename, argv, envp)
    -&gt; do_execveat_common()
      -&gt; do_execve_file(file, argv, envp, flags)
        -&gt; exec_binprm(bprm)
          -&gt; search_binary_handler(bprm)
            -&gt; load_elf_binary(bprm)    // 如果文件是 ELF
              [解析 ELF Header]
              [解析 Program Header Table]
              [mmap(PT_LOAD) 段] （创建VMA并挂到mm-&gt;mmap上）
              [如果有 PT_INTERP =&gt; load_elf_interp()]
              [setup_arg_pages() 构建用户栈，一般初始是4KB]
              [start_thread() 设置入口点]
            -&gt; ...

//准备用户态寄存器并返回            
// 设置 RIP/EIP = e_entry
// 栈指针 = 新的用户栈地址
// 返回到用户态开始执行
```

## 进程栈内存的使用

我们这里聊到的栈实际都是用户栈，与之对应的概念的是内核栈，后面有空再补。上面已经说了进程在创建完成后，虚拟内存大致的布局。就是初始的4KB，并且在最后执行`start_thread(regs, e_entry, stack_ptr)`的时候会把并把 **栈指针**（RSP/ESP）设置到我们刚刚分配好的用户栈顶。

&gt;   **内核栈（kernel stack）**：
&gt;
&gt;   每个进程/线程在内核态都有一块固定大小的栈（如 x86_64 典型 16KB，两页）。
&gt;
&gt;   分配时机：在 fork() / clone() 的 copy_process() 中由 alloc_thread_info_node() 等完成。
&gt;
&gt;   用途：当进程陷入内核态（系统调用、中断、异常）时使用，用于保存内核态的函数调用帧、局部变量等。
&gt;
&gt;   大小固定，不会像用户态栈那样自动扩张。
&gt;
&gt;   **用户态栈（user stack）**：
&gt;
&gt;   是我们上文描述的那块由进程自己使用的栈，从高地址向低地址增长，默认可扩展至一定限制（由内核 VMA &#43; ulimit -s 控制）。
&gt;
&gt;   分配时机：在 execve() 的 setup_arg_pages() 时初始建立；运行时通过 expand_stack() 等逻辑动态增大。
&gt;
&gt;   大小可变，受进程的内存限制和操作系统策略管控。

在 x86/x86_64 等多数架构上，**用户态栈地址**从高地址往低地址方向增长。也就是说，在 C 语言里调用函数、压入局部变量时，栈指针会不断减小。当栈帧变大或深度增加时，就需要更多的内存空间。

内核里有一个约定：**用户栈对应的虚拟内存区域具有** VM_GROWSDOWN **标志**。这意味着如果进程访问了比当前栈底（vma-&gt;vm_start）更低的地址，而且仍在“可增长区间”内，内核会尝试“向下扩展”这块栈 VMA，从而容纳新的栈数据。

### 动态扩展

当用户态执行某些操作，导致栈指针（RSP/ESP）越过了当前 VMA 的边界，就会触发一个 **缺页异常**（page fault）。内核的缺页异常处理大致会经过以下函数（以 x86_64 为例，路径和名称在不同内核版本里略有差异）：

1.  do_page_fault() (位于 arch/x86/mm/fault.c 等)

-   入口点，处理所有用户/内核缺页异常；
-   自用户态还是内核态？）；

1.  handle_mm_fault() (位于 mm/memory.c 等)

-   件映射？匿名映射？栈？）去做相应的处理；
-   _GROWSDOWN 标志的栈 VMA 之下，内核会调用 expand_stack(vma, address) 或类似流程。

1.  xpand_downwards() (位于 mm/mmap.c)

-   这是栈自动扩展的核心函数之一；
-   检查要扩展的地址 address 是否“合理”，比如不能无限制扩到进程地址空间之外、或者超出 ulimit -s 等限制；
-   **vma-&gt;vm_start** 到更低的地址，腾出更多虚拟地址空间供栈使用；
-      随后故障页就可以通过正常的缺页处理分配物理页，完成映射。

这样一来，进程的栈空间就会 “无缝” 地向下扩展——对用户程序来说，这一切自动发生，只要没超限，就可以继续使用。

## 线程栈内存的使用

首先要明确一个事情，在 Linux 内核中，“线程”与“进程”本质上都是 task_struct，只是在一些资源是否共享（如 mm_struct）方面有不同的 clone 标志位。二者在内核视角下非常相似，但在用户空间（尤其是 C/POSIX 线程库）会有额外的栈管理逻辑。

创建线程一般都是通过调用`pthread_create()`，这个函数底层是`clone() `系统调用（或更底层的 `clone3()`）来创建一个**新任务**（新“线程”）。它与“普通进程”最大的区别是：**新线程与父线程共享** `mm_struct`（地址空间、代码段、数据段等），而不是像新进程那样复制/独立地址空间。

&gt;   sys_clone() / sys_clone3()是 syscall 的入口点；负责解析用户传入的寄存器地址、栈地址、标志位（如 CLONE_VM, CLONE_FS, CLONE_FILES, CLONE_SIGHAND）等。
&gt;
&gt;   随后调用copy_process()分配新的 task_struct 和内核栈（通过 alloc_thread_info()），判断 clone_flags 里有没有 CLONE_VM：
&gt;
&gt;   -   如果 **有** CLONE_VM，则**新任务与父任务共享地址空间**（mm_struct），这就变成了**线程**；
&gt;   -   如果 **没有** CLONE_VM，会 copy 一份 mm_struct，就像传统的进程 fork()；
&gt;
&gt;   对于线程，**只保留**一个 mm_struct 引用计数 &#43;1（即不创建新的 mm）。设置好寄存器上下文，包括指令指针、栈指针（从用户态传来的那个“thread stack top”）。

但是，一个新的线程也需要自己的 **“用户态栈”**。这块栈往往是由用户层的线程库（例如 glibc 的 pthread 实现）使用 mmap() 或者从进程堆里分配一段内存来当作**线程栈**。

1.  **pthread 库的做法（简要）**

-   当你调用 pthread_create(&amp;tid, &amp;attr, start_routine, arg) 时，pthread 库内部会：
-   如果用户没有自行指定 pthread_attr_setstack()，则库会用 mmap() 分配一块内存作为“新线程栈”；
-   设置好这块内存区域的保护属性（PROT_READ|PROT_WRITE），并在用户层记录其起始地址、大小；
-   调用 clone() 系统调用时，将“新线程入口函数（start_routine）”以及“用户态栈顶地址”一起传给内核，让内核知道“进入用户态后，该线程从哪块栈开始执行”。

1.  **内核对用户栈并不“主动分配”**

-   与进程的“主线程栈”在 execve() 里通过 setup_arg_pages() 分配不同，**新线程**的用户栈是由用户层自己准备的（pthread 库做的那一堆 mmap() 或内存管理），再把栈顶指针告诉内核。
-   在内核看来，这些线程共用同一个 mm_struct，即同一个虚拟地址空间，只是每个线程有**不同的寄存器上下文**（包括不同的用户态栈指针）、以及**不同的内核栈**。

1.  **是否支持** VM_GROWSDOWN

-   对于“主线程”的栈，Linux 常常把它标记为带 VM_GROWSDOWN，以支持自动扩展；
-   对于“额外创建的线程栈”，取决于 pthread 库是否设置了相应的标志：通常 pthread 是**固定大小**的栈，不一定使用 VM_GROWSDOWN（或者只在一定范围内可扩展），因为该栈往往由库分配的一片匿名映射区域（mmap），不一定也不总是带自动扩展标志。
-   如果库或用户手动使用 MAP_GROWSDOWN 标志，也能生成类似“向下扩展”的 VMA，但这是**实现细节**，大多数常见的 pthread 实现里都是定长栈。

**对比：**

-   **线程栈内存的使用**：主要由用户空间的线程库来负责分配(固定大小或自定义)；内核在 clone() → copy_process() 时只需要记住“返回用户态时要用哪块栈”。
-   **对比进程**：进程在 execve() 时，由内核 setup_arg_pages() 创造“主线程栈”并可自动扩展；线程则没有单独 execve() 的过程，只是共享地址空间、共享可执行文件映射，各自使用用户层分配好的栈。

## 进程堆的内存管理

待补充

---

> 作者: [LGS](https://github.com/geekstormm)  
> URL: https://geekstormm.github.io/posts/mmoverview/  

