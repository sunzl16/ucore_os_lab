# Lab1 Report

## [练习1]

**[练习1.1]操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)**

```makefile
整个bin目录结构如下、

bin/
 ├── bootblock
 ├── kernel
 ├── sign
 └── ucore.img

#---#
> bin/ucore.img
其中生成ucore.img的相关Makefile代码为

UCOREIMG	:= $(call totarget,ucore.img)
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000	#创建512×10000 byte空间到/bin/ucore.img，初始全部为零
	$(V)dd if=$(bootblock) of=$@ conv=notrunc	#将512byte的bootblock拷到ucore.img开头处
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc	#跳过开头的1块512byte，将kernel拷到ucore.img
$(call create_target,ucore.img)

可以看出需要生成bootblock、kernel之后，才能生成ucore.img

#---|---#
> bin/bootblock
其中生成bootblock的相关Makefile代码为

bootblock = $(call totarget,bootblock)
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
$(call create_target,bootblock)

可以看出需要生成boot目录下的bootasm.o、bootmain.o和sign之后，才能生成bootblock

#---|---|---#
> obj/boot/bootasm.o，obj/boot/bootmain.o
其中生成bootasm.o、bootmain.o的相关Makefile代码为

bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

实际代码由foreach函数批量生成

#---|---|---|---#
生成bootasm.o需要bootasm.S
实际命令为
gcc -Iboot/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  \
	-fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S \
	-o obj/boot/bootasm.o
	
参数含义为：
	-fno-builtin：不接受没有 __builtin_ 前缀的函数作为内建函数
	-Wall：打开一些警告选项
	-ggdb：生成为gdb使用的调试信息，可供qemu+gdb来调试bootloader or ucore
	-m32：生成32为机器上的代码
	-gstabs：生成stabs格式的调试信息,不包含gdb扩展,方便ucore的monitor显示函数调用栈信息
    -nostdinc：查找头文件时，不搜索标准系统库所在路径，只搜索显示使用-I选项提供的目录
    -fno-stack-protector：不生成用于保护缓冲区、防止溢出的代码
    -I<dir>：表示将指定文件夹dir添加到头文件路径搜索列表中
    -Os：减少编译后代码的大小
    -c：编译或者汇编源代码时，不做链接
    -o file：指定输出进行入file文件，无论输出是可执行程序、对象程序、汇编程序还是预处理的C程序，都可以执行此项操作
	
#---|---|---|---#
生成bootmain.o需要bootmain.c
实际命令为
gcc -Iboot/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  \
	-fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c \
	-o obj/boot/bootmain.o

#---|---|---#
> bin/sign

其中生成sign的相关Makefile代码为
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

实际命令为
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

参数含义为
	-g：按照操作系统本身的格式，生成额外的只有gdb可以使用的调试信息
	-O2：开启各类优化来编译程序

#---|---#
有了bootasm.o与bootmain。o之后生成bootblock.o

实际命令为
ld -m  elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o \
	obj/boot/bootmain.o -o obj/bootblock.o

参数含义为
	-m <emulation>：模拟制定模拟器的链接器，这里为模拟为i386上的链接器
	-nostdlib：只使用命令中显示出现的库文件夹，不使用标准库
	-N  设置代码段和数据段均可读写
	-e <entry>：指定入口作为可执行文件的开始
	-Ttext <org>：制定代码段开始的绝对位置为org

#---|---#
> bin/kernel
其中生成kernel的相关Makefile代码为

kernel = $(call totarget,kernel)
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
$(call create_target,kernel)

可以看出需要生成kernel.ld、readline.o、stdio.o、init.o、kdebug.o、kmonitor.o 、panic.o、clock.o、console.o、intr.o、picirq.o、trap.o、trapentry.o、vectors.o、 pmm.o 、printfmt.o、string.o之后，才能生成kernel

#---|---|---#
> obj/kern/*/*.o
其中生成这些.o文件的相关Makefile代码为
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

其中以init.o为例，实际代码为
gcc -Ikern/init/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc \
	-fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ \
	-Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
	
#---|---#
当一系列.o文件都生成完毕后，开始生成kernel

实际命令为
ld -m  elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
	obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o \
    obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o \
    obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o \
    obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o \
    obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o \
    obj/libs/printfmt.o
    
参数含义为
	-T <scriptfile>：使用制定脚本文件作为链接器的脚本

#---#
有了bootblock与kernel之后，生成ucore.img

实际命令为：
dd if=/dev/zero of=bin/ucore.img count=10000
	#生成一个有10000个块的文件，每个块默认512字节，用0填充
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
	#把bootblock中的内容写到第一个块
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
	#从第二个块开始写kernel中的内容
```

**[练习1.2]一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？**

tools/sign.c是构建硬盘主引导扇区的，此扇区大小必须为512bit，最后两位必须是0x55AA。



## [练习2]

**[练习2.1]从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行**

在Makefile文件中，修改debug命令中调用qemu的参数（第一行），结果如下，就可以实现qemu和gdb结合起来调试bootloader，并把他执行的汇编指令记录和保存至q.log中。

```makefile
debug: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

同时修改tools/gdbinit文件为，

```makefile
file bin/kernel
set architecture i8086
target remote :1234
```

此时在lab1目录下，执行“make debug”命令，即可进入调试模式，在gdb调试界面中输入“si”命令，或者在qemu调试界面下输入“x /i $pc”，就可以对BIOS进行单步跟踪。

```
(qemu)	x /i $pc
0xfffffff0:   ljmp   $0xf000,$0xe05b
```

**[练习2.2]在初始化位置0x7c00设置实地址断点,测试断点正常。**

上一步的基础上，在tools/gdbinit文件后面添加如下代码，以实现在初始化位置设置断点与调试：

```makefile
b *0x7c00
continue
x /2i $pc
```

经测试，进入gdb后，断点正常。

```
Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:      cli
   0x7c01:      cld
```

**[练习2.3]从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。**

通过观察q.log的输出，反汇编代码和原代码基本相同。

**[练习2.4]自己找一个bootloader或内核中的代码位置，设置断点并进行测试。**

选择内核中的print_stackframe函数位置，进入gdb后，在次函数位置设置断点“b print_stackframe”，然后执行“continue”指令，断点停在了0x100a26的指令处，运行结果如下：

```
(gdb) b print_stackframe
Breakpoint 2 at 0x100a26: file kern/debug/kdebug.c, line 292.
(gdb) continue
Continuing.

Breakpoint 2, print_stackframe () at kern/debug/kdebug.c:292
(gdb) x /10i $pc
=> 0x100a26 <print_stackframe>: push   %ebp
   0x100a27 <print_stackframe+1>:       mov    %esp,%ebp
   0x100a29 <print_stackframe+3>:       sub    $0x28,%esp
   0x100a2c <print_stackframe+6>:       mov    %ebp,%eax
   0x100a2e <print_stackframe+8>:       mov    %eax,-0x20(%ebp)
   0x100a31 <print_stackframe+11>:      mov    -0x20(%ebp),%eax
   0x100a34 <print_stackframe+14>:      mov    %eax,-0xc(%ebp)
   0x100a37 <print_stackframe+17>:      call   0x100a15 <read_eip>
   0x100a3c <print_stackframe+22>:      mov    %eax,-0x10(%ebp)
   0x100a3f <print_stackframe+25>:      movl   $0x0,-0x14(%ebp)
(gdb) 
```



## [练习3]

**BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。**

Intel早期的8086 CPU提供了20根地址线，可寻址空间范围为1M的内存空间。但8086的数据处理位宽为16位，无法直接寻址1M内存空间，为此8086提供了段基址+段内偏移的地址转换机制，具体实现为短号左移四位与偏移量相加，因此最大可以访问$0xffff<<4+0xffff=0x10ffefh>0x100000=1M$，也就是说超过了段机制的寻址方式超过了20位地址线的物理寻址能力，因此当寻址空间超过1M时，会发生“回卷”而不产生异常。

而从这之后，80286地址线数量达到24根，80386地址线数量达到32根，均远超1M的寻址空间，此时为了保持向下兼容，使得寻址超过1M仍然产生“回卷”，IBM决定在PC AT计算机系统上加个硬件逻辑，来模仿以上的回绕特征，于是出现了A20 Gate，通过做与操作，来控制高位能否被访问，一开始为0表示屏蔽，直到系统软件通过IO操作打开它。

bootloader从实模式切换到保护模式的大致过程为：初始化寄存器；开启A20；初始化GDT表；使能并进入保护模式。

结合bootasm.S代码，分析如下：

+ 禁止中断，字符串操作增加

  ```assembly
  cli                                             # Disable interrupts
  cld                                             # String operations increment
  ```

+ 初始化数据段寄存器DS、ES、SS，置为零

  ```assembly
  xorw %ax, %ax                                   # Segment number zero
  movw %ax, %ds                                   # -> Data Segment
  movw %ax, %es                                   # -> Extra Segment
  movw %ax, %ss                                   # -> Stack Segment
  ```

+ 使能A20，具体操作为：利用busy-waiting的方法，不断检查8042输入缓存区是否为空，确定空闲之后，将0xd1放入0x64端口，表示写数据进入8042的P2端口。然后利用bush-waiting的方式，等待8042输入缓存区空闲，然后将0xdf写入0x60接口，表示置P2的A20位为1，即打开地址线A20。

  ```assembly
  seta20.1:
      inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
      testb $0x2, %al
      jnz seta20.1
      movb $0xd1, %al                                 # 0xd1 -> port 0x64
      outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
      
  seta20.2:
      inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
      testb $0x2, %al
      jnz seta20.2
  
      movb $0xdf, %al                                 # 0xdf -> port 0x60
      outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
  
  ```

+ 加载GDT表

  ```assembly
  	lgdt gdtdesc
  ```

+ 使能CR0寄存器的PE位，跳转到32位代码段，切换到32位模式，完成实模式到保护模式的转换

  ```assembly
      movl %cr0, %eax
      orl $CR0_PE_ON, %eax
      movl %eax, %cr0
      
      ljmp $PROT_MODE_CSEG, $protcseg
  ```

+ 设置32位模式下的段寄存器，建立保护模式下的堆栈及其寄存器，并调用bootmain函数

  ```assembly
  .code32                                             # Assemble for 32-bit mode
  protcseg:
      # Set up the protected-mode data segment registers
      movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
      movw %ax, %ds                                   # -> DS: Data Segment
      movw %ax, %es                                   # -> ES: Extra Segment
      movw %ax, %fs                                   # -> FS
      movw %ax, %gs                                   # -> GS
      movw %ax, %ss                                   # -> SS: Stack Segment
  
      # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
      movl $0x0, %ebp
      movl $start, %esp
      call bootmain
  ```



## [练习4]

**分析bootloader加载ELF格式的OS的过程。**

**[练习4.1]bootloader如何读取硬盘扇区的？**

ucore实验中，读取扇区可以分成四个步骤：等待磁盘准备好；发出读取扇区命令；等待磁盘准备好；读取磁盘数据到指定内存。

分析bootmain.c代码如下：

```c
static void
readsect(void *dst, uint32_t secno) {
    waitdisk(); //用while循环，检查0x1f7不忙的时候，可以开始准备读取扇区

    outb(0x1F2, 1); //设置要读取扇区数量为1
    outb(0x1F3, secno & 0xFF); //写入希望读取的扇区号低0-7位
    outb(0x1F4, (secno >> 8) & 0xFF); //写入希望读取的扇区号低8-15位
    outb(0x1F5, (secno >> 16) & 0xFF); //写入希望读取的扇区号低16-23位
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0); //写入希望读取的扇区号最高8位
    outb(0x1F7, 0x20);                      // 吸入0x20，开始读取扇区
    // 再次等待0x1f7空闲状态
    waitdisk();
    // 读取一个扇区的数据到指定内存中
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

**[练习4.2]bootloader是如何加载ELF格式的OS？**

```c
void
bootmain(void) {
    // 读取磁盘第一页的信息
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    // 以magic值判断是否ELF文件合法
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;	// 不合法则调到bad代码段执行相应动作
    }

    struct proghdr *ph, *eph; //申明存储偏移和入口数目

    // 加载每一个程序段
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff); // 获得ph表的偏移
    eph = ph + ELFHDR->e_phnum; // 获得ph表的入口数目
    for (; ph < eph; ph ++) { // 遍历所有ph表项
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset); //读取其中变量数据扇区到指定内存单元中去
    }

    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))(); // 跳转到ELF头存储的入口虚拟地址

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```



## [练习5]

**实现函数调用堆栈跟踪函数**

此练习按照kern/debug/kdebug.c中的print_stackframe函数提示，从(1)~(3)的步骤一块块的完成即可。主要在于仔细实验指导书的“函数堆栈”部分的知识。搞清楚ebp和eip的关系。同时注意边界条件ebp==0,。

**实验代码和注释**

```c
void print_stackframe(void) {
	uint32_t ebp = read_ebp();	// 获取当前ebp
	uint32_t eip = read_eip();	// 获取当前eip

	int i, j;	// 设置循环变量i，j
	for(i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++) {	//注意边界条件
		cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);	//按格式输出信息
		uint32_t *arg = (uint32_t *)ebp + 2;	// 获取参数地址
		for(j = 0; ebp != 0 && j < 4; j ++) {
			cprintf("0x%08x ", arg[j]);
		}
		cprintf("\n");
		print_debuginfo(eip-1);			// 打印当前函数名和行数
		eip = ((uint32_t *)ebp)[1];		// 弹出堆帧，eip指向压栈时的eip
		ebp = ((uint32_t *)ebp)[0];		// 弹出堆帧，ebp指向压栈函数的ebp
	}
}

```

**实验结果**

```
ebp:0x00007b38 eip:0x00100a3c args:0x00010094 0x00010094 0x00007b68 0x0010007f 
    kern/debug/kdebug.c:306: print_stackframe+21
ebp:0x00007b48 eip:0x00100d3e args:0x00000000 0x00000000 0x00000000 0x00007bb8 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b68 eip:0x0010007f args:0x00000000 0x00007b90 0xffff0000 0x00007b94 
    kern/init/init.c:48: grade_backtrace2+19
ebp:0x00007b88 eip:0x001000a1 args:0x00000000 0xffff0000 0x00007bb4 0x00000029 
    kern/init/init.c:53: grade_backtrace1+27
ebp:0x00007ba8 eip:0x001000be args:0x00000000 0x00100000 0xffff0000 0x00100043 
    kern/init/init.c:58: grade_backtrace0+19
ebp:0x00007bc8 eip:0x001000df args:0x00000000 0x00000000 0x00000000 0x00103440 
    kern/init/init.c:63: grade_backtrace+26
ebp:0x00007be8 eip:0x00100050 args:0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+79
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d6d --
```

和实验指导书基本一致，最后一行的参数含义如下：

+ ebp：基值指针
+ eip：当前PC位置的下一条指令地址
+ args：函数调用传入的参数
+ \<unknow\>: -- 0x00007d6d -- 表示进行函数调用开始的程序地址，没有c语言的函数名称



## [练习6]

**完善中断初始化和处理**

**[练习6.1]中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**

每一个表项占8个字节（64位），其中0、1字节表示地址偏移的低16位，第6、7字节表示地址偏移的高16位。第2、3字节表示段选择子，段选择子与段内偏移共同组成中断处理代码的入口。

**[练习6.2]请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。**

这里按照kern/trap/trap.c中idt_init函数的指导过程，就能比较好的完成程序设计。需要注意的是对中断、异常的处理进行分类：中断号0到31的是trap gate，中断号在32到255的时interrupt gate，另外T_SYSCALL和T_SWITCH_TOK的DPL为USER而不是KERNEL。

**代码实现如下：**

```c
void idt_init(void) {
	extern const uintptr_t __vectors[];
	int i;
	for(i = 0; i < 256; i++) {
		if(i < 32) {	// 低于三十二的为自陷门
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_KERNEL);
		}
		else {		// 大于等于三十二的为中断门
			SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
		}
	}
    // 设置T_SYSCALL，T_SWITCH_TOK的DPL为USER特权级
	SETGATE(idt[T_SYSCALL], 0, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	lidt(&idt_pd);	// 通知CPU加载IDT表
}
```

**[练习6.3]请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。**

“Too Simple”

**代码实现如下：**

```c
    case IRQ_OFFSET + IRQ_TIMER:
		ticks ++;	// 时钟计数加一
		if(ticks % TICK_NUM == 0) {	// 每TICK_NUM次，打印一次信息
			print_ticks();	// 打印信息
		}
        break;
```





## [扩展练习]

**[扩展练习1]**

由于CS、DS、SS寄存器会保存当前处于内核态还是用户态的信息，根据课上所讲，在中断服务例程中，直接将这些寄存器的值改掉就可以了。而对于kern/init/int.c中的lab1_switch_to_user与lab1_switch_to_kernel函数，使用内敛汇编去调用中断，需要注意调用之前和之后保存esp和ebp的值。防止破坏栈结构。具体代码如下

```asm
static void lab1_switch_to_user(void) {
	asm volatile (
		"sub $0x8, %%esp \n"
		"int %0 \n"
		"movl %%ebp, %%esp"
		:
		: "i"(T_SWITCH_TOU)
	);
}

static void lab1_switch_to_kernel(void) {
	asm volatile (
		"sub $0x8, %%esp \n"
		"int %0 \n"
		"movl %%ebp, %%esp"
		:
		: "i"(T_SWITCH_TOK)
	);
}
```

服务例程代码如下：

```c
    case T_SWITCH_TOU:
    	if(tf->tf_cs != USER_CS) {
    		tf->tf_cs = USER_CS;
    		tf->tf_ds = tf->tf_ss = tf->tf_es = USER_DS;
    		tf->tf_eflags |= FL_IOPL_MASK;
    	}
    	break;
    case T_SWITCH_TOK:
    	if(tf->tf_cs != KERNEL_CS) {
    		tf->tf_cs = KERNEL_CS;
    		tf->tf_ds = tf->tf_ss = tf->tf_es = KERNEL_DS;
    	}
        break;
```

**[扩展练习2]**

修改中断服务例程对于键盘中断的处理即可，分别处理输入是0和3的情况，代码如下：

```asm
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        if(c == '0') {
        	if(tf->tf_cs != KERNEL_CS) {
            	cprintf("+++ Typing number 0: SUCCESSFULLY! now switch to kernel mode +++\n");
        		tf->tf_cs = KERNEL_CS;
        		tf->tf_ds = tf->tf_ss = tf->tf_es = KERNEL_DS;
        	}
        	else {
        		cprintf("+++ Typing number 0: SORRY! you are being kernel mode now +++\n");
        	}
        }
        else if(c == '3') {
        	if(tf->tf_cs != USER_CS) {
        		cprintf("+++ Typing number 3: SUCCESSFULLY! now switch to user mode +++\n");
        		tf->tf_cs = USER_CS;
        		tf->tf_ds = tf->tf_ss = tf->tf_es = USER_DS;
        		tf->tf_eflags |= FL_IOPL_MASK;
        	}
        	else {
        		cprintf("+++ Typing number 3: SORRY! you are being user mode now +++\n");
        	}
        }
        break;
```



## 本人实现与参考答案的区别

+ 练习1：参考答案给的结构非常好，但最开始自己没有意识到为什么要这样组织。所以后来在每一块之前添加“#---#”、"#---|---"#、"#---|---|---#"的方式来表达结构中所在的层级。但整体而言还是很有逻辑性的。
+ 练习2：此题基本一致，都是修改了gdbinit文件作为配置
+ 练习3：我对于A20Gate的解答较为详细，关于代码的解读大同小异
+ 练习4：我是按照实验指导书的两个问题进行解答，参考答案按照代码结构解读
+ 练习5：由于Lab1中的注释给的足够详细，代码实现区别不大。
+ 练习6：主要在于init_gdt中的实现不同，参考答案没有区分trap gate和interrupt gate，给出了T_SWITCH_TOK的DPL设置为USER，这个我最早没有考虑到。我是按照指导书区分了两种门以及T_SYSCALL的DPL为USER。



## 与OS原理课相关知识点

+ 练习1：主引导扇区格式
+ 练习2：BIOS、x86实模式与保护模式
+ 练习3：bootloader启动过程，GDT表的建立
+ 练习4：ELF文件格式及其加载方式，bootloader读取硬盘的过程
+ 练习5：函数调用的堆栈使用
+ 练习6：中断与异常、中断处理向量、中断描述符表
+ 未提到的知识点：进入BIOS后的硬件自检POST、系统设备启动顺序、段机制与页机制的处理。