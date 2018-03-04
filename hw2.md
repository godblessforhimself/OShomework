

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？
	- 硬件上提供系统调用、实现基本汇编指令的支持。提供中断、读写外设的特权指令
- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？
	- 实模式只能访问地址在1M以下的内存称为常规内存，我们把地址在1M 以上的内存称为扩展内存。在保护模式下，全部32条地址线有效，可寻址高达4G字节的物理地址空间;实模式可以直接访问硬件地址，没有内存保护和多道程序的概念。而保护模式中，只能访问每个进程自己的虚拟地址空间，并且是多道程序运行模式。
- 理解list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```
- 数字表示位域的长度，16表示这个变量占16位。
	
- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？

- 小尾端。运行结果是0x20003 = 131075。intr是32位的，对应结构体的低32位，gd_off_15_0和gd_ss。SETGATE把低16位改成了3，把高16位改成了2，所以是0x20003。


### 课堂实践练习

#### 练习一

请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。


```


.globl start

start:

.code16                                             # Assemble for 16-bit mode

    cli                                             # 关闭中断

    cld                                             # 令DF=0

    xorw %ax, %ax                                   # ax寄存器自己异或自己，即清0

    movw %ax, %ds                                   # 使ds寄存器清0

    movw %ax, %es                                   # es寄存器清0

    movw %ax, %ss                                   # ss寄存器清0

seta20.1:

    inb $0x64, %al                                  # 读内存0x64的值到al寄存器

    testb $0x2, %al									# al寄存器的值与0x2比较

    jnz seta20.1									# 若比较寄存器不为0，则跳回开头
	movb $0xd1, %al                                 # 令al=0xd1

    outb %al, $0x64                                 # 写0xd1到0x64内存地址

	该段循环等待。直到0x64的值等于2，才继续执行。把0xd1写回0x64内存地址，向下执行。

seta20.2:

    inb $0x64, %al                                  # 读0x64地址值到al寄存器

    testb $0x2, %al									# 比较2与al寄存器的值

    jnz seta20.2									# 若不相等则回头



    movb $0xdf, %al                                 # 将0xdf赋值给al寄存器

    outb %al, $0x60                                 # 将0xdf写到0x60地址。

	该段循环等待，直到mem[0x64]=2，才继续执行. mem[0x60]=0xdf，向下执行。


    lgdt gdtdesc									#加载全局描述符表地址

	movl %cr0, %eax									# 将cr0赋值给eax
	
	orl $CR0_PE_ON, %eax							# 将常量或eax

    movl %eax, %cr0									# 将eax赋值给cr0

	这段功能把cr0和CR0_PE_ON作或写给cr0。不知道为什么要用eax


    ljmp $PROT_MODE_CSEG, $protcseg					#跳转到gdt[PROT_MODE_CSEG]+protcseg地址



.code32                                             # Assemble for 32-bit mode

protcseg:

    movw $PROT_MODE_DSEG, %ax                       # 将$PROT_MODE_DSEG赋值给ax寄存器

    movw %ax, %ds                                   # 将$PROT_MODE_DSEG赋值给ds寄存器

    movw %ax, %es                                   # 将$PROT_MODE_DSEG赋值给es寄存器

    movw %ax, %fs                                   # 将$PROT_MODE_DSEG赋值给fs寄存器

    movw %ax, %gs                                   # 将$PROT_MODE_DSEG赋值给gs寄存器

    movw %ax, %ss                                   # 将$PROT_MODE_DSEG赋值给ss寄存器

    movl $0x0, %ebp									#将ebp置为0

    movl $start, %esp								#将start赋值给esp

    call bootmain									#调用bootmain函数

spin:

    jmp spin										#原地旋转

	以上代码主要做了：关中断，清空一系列寄存器的值，监控内存地址0x64的值，加载全局描述符表，设置寄存器cr0等的值，最后调用bootmain，以及一个出错的循环结尾。
```
#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore中宏定义的用途，并举例描述其含义。

 > 利用宏进行复杂数据结构中的数据访问；
```
#define le2vma(le, member)                  \

    to_struct((le), struct vma_struct, member)
```
 > 进行虚实地址转换，同时有检错功能。
```	
	#define PADDR(kva) ({                                                   \

            uintptr_t __m_kva = (uintptr_t)(kva);                       \

            if (__m_kva < KERNBASE) {                                   \

                panic("PADDR called with invalid kva %08lx", __m_kva);  \

            }                                                           \

            __m_kva - KERNBASE;                                         \

        })
```

 
 > 利用宏进行数据类型转换；如 to_struct, 
```	#define to_struct(ptr, type, member)                               \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```
 > 常用功能的代码片段优化；如  ROUNDDOWN, SetPageDirty
```
#define ROUNDDOWN(a, n) ({                                          \
            size_t __a = (size_t)(a);                               \
            (typeof(a))(__a - __a % (n));                           \
        })
```