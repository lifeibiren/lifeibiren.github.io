+++
title = "TLS 内存分配方式探究"
date = "2024-08-20T01:13:00+08:00"
description = "TLS 内存分配方式探究"
tags = [
    "libc",
    "linux",
]
categories = "blog"
+++

事情的起因是，需要将一堆代码编译生成动态链接库（以前是直接生成 executable），在加载的时候报错。
翻查资料发现是跟 TLS 变量的大小和编译代码时使用的 TLS Model 有关系，在 initial-exec 模式下总大小超过 2048 字节（这个说法不精确，具体细节请参考[libc-tls.c](https://github.com/lattera/glibc/blob/895ef79e04a953cac1493863bcae29ad85657ee1/csu/libc-tls.c#L57)）就会产生这个错误。

<!--more-->

简化之后的问题模型如下：

1.动态库so
```
$ cat > test_tls.cpp <<< EOF
thread_local char buf[2048];

int foo() {
    return buf[0];
}
EOF

$ g++ -shared -fpic -ftls-model=initial-exec -o libtest_tls.so
```
2.加载动态库的程序
```
$ cat > loadso.cpp <<< EOF
#include <dlfcn.h>
#include <stdio.h>

int main() {
    if (!dlopen("libtest_tls.so", RTLD_NOW | RTLD_GLOBAL)) {
        printf("%s\n", dlerror());
    }
    return 0;
}
EOF

$ c++ loadso.cpp -o loadso
$ LD_LIBRARY_PATH=./ ./a.out
./libtest_tls.so: cannot allocate memory in static TLS block
```

在 gcc 的手册中有这样的描述：

```
-ftls-model=model
           Alter the thread-local storage model to be used.  The model argument should be one of
           global-dynamic, local-dynamic, initial-exec or local-exec.  Note that the  choice  is
           subject  to optimization: the compiler may use a more efficient model for symbols not
           visible outside of the translation unit, or if -fpic is  not  given  on  the  command
           line.

           The default without -fpic is initial-exec; with -fpic the default is global-dynamic.
```

详细的 TLS Model 及寻址方式参见文档 [ELF Handling For Thread-Local Storage](https://www.akkadia.org/drepper/tls.pdf "ELF Handling For Thread-Local Storage")。

下面简单总结一下这四种 TLS Model：

# global-dynamic（ELF 文档中称为 General Dynamic）
最通用的模式，在 .got 中分配一个 tls_index，将其地址传给函数 __tls_get_addr，返回结果即为 tls 变量的实际地址。
```
$c++ test_tls.cpp -shared -fpic -ftls-model=global-dynamic -S
...
	.text
	.globl	_Z3foov
	.type	_Z3foov, @function
_Z3foov:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	data16	leaq	buf@tlsgd(%rip), %rdi
	.value	0x6666
	rex64
	call	__tls_get_addr@PLT
	movzbl	(%rax), %eax
	movsbl	%al, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
...
```

# local-dynamic
当 tls 变量与访问它的代码在同一个 module 的时候，可以使用 local-dynamic 的方式进行优化，特别是当需要访问的 tls 变量数量 > 1 时，可以有效减少地址计算的开销。
C++ 代码修改为
```
thread_local char buf[2048];
thread_local char buf2[2048];

int foo() {
    return buf[0] + buf2[0];
}
```

```
$c++ test_tls.cpp -shared -fpic -ftls-model=global-dynamic -S
...
_Z3foov:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	pushq	%rbx
	subq	$8, %rsp
	.cfi_offset 3, -24
	leaq	buf@tlsld(%rip), %rdi
	call	__tls_get_addr@PLT
	addq	$buf@dtpoff, %rax
	movzbl	(%rax), %eax
	movsbl	%al, %ebx
	leaq	buf@tlsld(%rip), %rdi
	call	__tls_get_addr@PLT
	addq	$buf2@dtpoff, %rax
	movzbl	(%rax), %eax
	movsbl	%al, %eax
	addl	%ebx, %eax
	movq	-8(%rbp), %rbx
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

看起来还是调用了两次，但是两次调用参数是一样的，开启优化之后就清晰了

```
$c++ test_tls.cpp -shared -fpic -ftls-model=local-dynamic -S -O2
_Z3foov:
.LFB0:
	.cfi_startproc
	subq	$8, %rsp
	.cfi_def_cfa_offset 16
	leaq	buf@tlsld(%rip), %rdi
	call	__tls_get_addr@PLT
	movsbl	buf@dtpoff(%rax), %edx
	movsbl	buf2@dtpoff(%rax), %eax
	addq	$8, %rsp
	.cfi_def_cfa_offset 8
	addl	%edx, %eax
	ret
	.cfi_endproc
...

```

# initial-exec
从 TCB 中分配内存，对应的 GOT 中由 dynamic linker 计算 offset。在 x86-64 架构中，段寄存器 %fs 保存了 TCB 的偏移地址。由于 TCB 的大小是固定的，因此使用这种方式分配的 TLS 变量不宜过大。

```
c++ test_tls.cpp -shared -fpic -ftls-model=initial-exec -S
_Z3foov:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movq	buf@gottpoff(%rip), %rax
	movzbl	%fs:(%rax), %eax
	movsbl	%al, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

# local-exec
这种方式是 local-dynamic 的进一步优化，只能用于可执行程序访问自身的变量。所有 TLS 变量在 TCB 中的 offset 在 link-time 被确定（生成立即数）。**这种方式编译代码，即使声明 -fpic，也无法链接成动态库。**

```
c++ test_tls.cpp -shared -fpic -ftls-model=local-exec -S
_Z3foov:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movzbl	%fs:buf@tpoff, %eax
	movsbl	%al, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```