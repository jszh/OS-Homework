# lec 3 SPOC Discussion
https://github.com/chyyuu/os_course_exercises/blob/2018spring/all/02-1-spoc-discussion.md

## 第三讲 启动、中断、异常和系统调用-思考题

## 3.1 BIOS
- BIOS从磁盘读入的第一个扇区是是什么内容？为什么没有直接读入操作系统内核映像？

首先读入的事引导扇区，包含加载程序。没有直接读入操作系统内核映像是因为 BIOS 需要通过加载程序识别磁盘文件系统，才能读取内核镜像，将操作系统加载到内存。

- 比较UEFI和BIOS的区别。
  + UEFI：在所有平台上提供一致的操作系统启动服务，配置灵活，安全性强。
  + BIOS：是固化在计算机主板上的程序，在磁盘管理、安全性等方面局限较大。


## 3.2 系统启动流程

- 分区引导扇区的结束标志是什么？

0x55AA

- 在UEFI中的可信启动有什么作用？

在引导记录中加入签名，在读入引导记录时进行可信验证，只会加载通过验证的代码，减少安全风险。

## 3.3 中断、异常和系统调用比较
- 什么是中断、异常和系统调用？
  + 中断：对外部意外的响应
  + 异常：指令执行意外的响应
  + 系统调用：系统调用指令的响应
 
-  中断、异常和系统调用的处理流程有什么异同？

源头、响应方式、处理机制不同。

- 以ucore lab8的answer为例，uCore的系统调用有哪些？大致的功能分类有哪些？
  + 进程管理：exec, exit, fork, getpid, kill, sleep, wait, yield
  + 文件操作：open, close, read, write, seek, fsync, fstat, getcwd, getdirentry, dup
  + 内存管理：pgdir
  + 外设输出：putc

## 3.4 linux系统调用分析
- 通过分析[lab1_ex0](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab1/lab1-ex0.md)了解Linux应用的系统调用编写和含义。(仅实践，不用回答)
- 通过调试[lab1_ex1](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab1/lab1-ex1.md)了解Linux应用的系统调用执行过程。(仅实践，不用回答)


## 3.5 ucore系统调用分析 （扩展练习，可选）
-  基于实验八的代码分析ucore的系统调用实现，说明指定系统调用的参数和返回值的传递方式和存放位置信息，以及内核中的系统调用功能实现函数。
- 以ucore lab8的answer为例，分析ucore 应用的系统调用编写和含义。

```c
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

这段代码是用户态的syscall函数，其中num参数为系统调用号，该函数将参数准备好后，通过 `SYSCALL` 汇编指令进行系统调用，进入内核态，返回值放在 `eax` 寄存器，传入参数通过 `eax` ~ `esi` 依次传递进去。在内核态中，首先进入 `trap()` 函数，然后调用 ` trap_dispatch()` 进入中断分发，当系统得知该中断为系统调用后，OS调用如下的 `syscall` 函数

```c
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

该函数得到系统调用号 `num = tf->tf_regs.reg_eax;`，通过计算快速跳转到相应的 `sys_` 开头的函数，最终在内核态中，完成系统调用所需要的功能。

- 以ucore lab8的answer为例，尝试修改并运行ucore OS kernel代码，使其具有类似Linux应用工具`strace`的功能，即能够显示出应用程序发出的系统调用，从而可以分析ucore应用的系统调用执行过程。

利用 `trap.c` 的 `trap_in_kernel()` 函数判断是否是用户态的系统调用，调用 `syscall()` 时传入此参数

```c
    case T_SYSCALL:
        syscall(trap_in_kernel(tf));
        break;
```
更改 `syscall()` 的函数原型为

```c
    void syscall(bool);
```

之后在 `syscall(bool)` 中加入输出即可
```c
    int num = tf->tf_regs.reg_eax;
    if (num >= 0 && num < NUM_SYSCALLS) {
        if (syscalls[num] != NULL) {
            arg[0] = tf->tf_regs.reg_edx;
            arg[1] = tf->tf_regs.reg_ecx;
            arg[2] = tf->tf_regs.reg_ebx;
            arg[3] = tf->tf_regs.reg_edi;
            arg[4] = tf->tf_regs.reg_esi;

	    if (!in_kernel) {
	    	cprintf("SYSCALL: %d\n", num);
	    }
            tf->tf_regs.reg_eax = syscalls[num](arg);
            return ;
        }
    }
```
 
## 3.6 请分析函数调用和系统调用的区别
- 系统调用与函数调用的区别是什么？

指令不同、特权级不同、堆栈切换。

- 通过分析`int`、`iret`、`call`和`ret`的指令准确功能和调用代码，比较函数调用与系统调用的堆栈操作有什么不同？

SS:SP压栈
