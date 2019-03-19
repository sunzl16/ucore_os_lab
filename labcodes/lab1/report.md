# Lab1 Report

## [练习1]

[练习1.1]操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

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

[练习1.2]一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

tools/sign.c是构建硬盘主引导扇区的，此扇区大小必须为512bit，最后两位必须是0x55AA。



## [练习2]

