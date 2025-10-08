<h1 align="center">Ucore Lab 1 Report</h1>

# 一、理解内核启动中的程序入口操作

# 二、使用GDB验证启动流程

## 调试过程

启动qemu和gdb，此时qemu模拟的riscv64 CPU停在复位地址0x1000处，即此时pc的值为0x1000

![image-20251008213452384](report.assets/image-20251008213452384.png)

在kern_entry处断点，输出信息显示在0x8020000处断点，该地址正是entry.S中定义的内核入口kern_entry在链接脚本中被指定的加载位置。

![image-20251008214737168](report.assets/image-20251008214737168.png)

继续执行，程序停在第一条指令前。

```asm
kern_entry:
    la sp, bootstacktop
    tail kern_init
```

在entry.S中，控制权被交给init.c之前，栈先一步被开辟。使用gdb si单步执行并使用gdb i r获取执行前后寄存器值如下：

| reg  | pre        | end        |
| ---- | ---------- | ---------- |
| pc   | 0x80200000 | 0x80200004 |
| sp   | 0x8001bd80 | 0x80203000 |

栈顶指针SP被SP成功设置为entry.S中标签bootstacktop的地址0x80203000。

控制权交到init.c后，引入了两个地址变量：edata, end。这两个变量在链接脚本中被定义，两个地址之间是未初始变量，这部分空间需要被填充为0。

```c
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);

    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);
   while (1)
        ;
}
```

继续使用gdb si，观察memset()函数执行前后edata与end间的内存情况：

<img src="report.assets/image-20251008224017243.png" width="80%">

由于本次实验代码中没有.bss代码段，edata = end，所以此处仅作演示。

本次实验实现的系统内核实现了通过opensbi实现了字符打印，关键函数为sbi_call()

```c
uint64_t sbi_call(uint64_t sbi_type, uint64_t arg0, uint64_t arg1, uint64_t arg2) {
    uint64_t ret_val;
    __asm__ volatile (
        "mv x17, %[sbi_type]\n"
        "mv x10, %[arg0]\n"
        "mv x11, %[arg1]\n"
        "mv x12, %[arg2]\n"
        "ecall\n"
        "mv %[ret_val], x10"
        : [ret_val] "=r" (ret_val)
        : [sbi_type] "r" (sbi_type), [arg0] "r" (arg0), [arg1] "r" (arg1), [arg2] "r" (arg2)
        : "memory"
    );
    return ret_val;
}
```

调试至该位置：

![image-20251008230516804](report.assets/image-20251008230516804.png)

使用gdb i r 获取寄存器值：

<img src="report.assets/image-20251008231838909.png" width="70%">

记录函数执行前后寄存器的值：

| reg                 | pre        | end  |
| ------------------- | ---------- | ---- |
| x17 / a7 (sbi_type) | 0x80200000 | 0x1  |
| x10 / a0 (arg0)     | 0x28       | 0x28 |
| x11 / a1 (arg1)     | 0x80202f94 | 0x0  |
| x12 / a2 (arg2)     | 0x802004c8 | 0x0  |

根据功能号可知触发了SBI_CONSOLE_PUTCH，参数arg0为要打印的字符“(“，实现了向屏幕打印字符的功能。

最终调试完成，代码运行正常。

<img src="report.assets/image-20251008233019452.png" width="70%">





## 观察结果

## Q&A

- RISC-V 硬件加电后最初执行的几条指令位于什么地址？它们主要完成了哪些功能？

| 位置               | 指令                           | 功能             |
| ------------------ | ------------------------------ | ---------------- |
| 0x0000000080200004 | la sp, bootstacktop            | 开辟栈           |
| 0x000000008020000e | memset(edata, 0, end - edata); | 未初始数据块置0  |
| 0x000000008020002a | cprintf("%s\n\n", message);    | 向屏幕打印字符串 |

