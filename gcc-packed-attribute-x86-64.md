# 在 x86-64 平台上，關於 GCC 編譯器中 `packed` 屬性的說明

https://gcc.gnu.org/onlinedocs/gcc-15.2.0/gcc/

**packed**

The `packed` attribute specifies that a structure member should have the smallest
possible alignment—one bit for a bit-field and one byte otherwise, unless a larger
value is specified with the `aligned` attribute. The attribute does not apply to
non-member objects.
For example in the structure below, the member array x is packed so that it
immediately follows a with no intervening padding:

```c
struct foo
{
    char a;
    int x[2] __attribute__ ((packed));
};
```

*Note:* The 4.1, 4.2 and 4.3 series of GCC ignore the `packed` attribute on bitfields of type `char`. This has been fixed in GCC 4.4 but the change can lead to differences in the structure layout. See the documentation of `-Wpacked-bitfield-compat` for more information.

使用 GCC 編譯器撰寫 C 語言的時候，使用 `__attribute__((packed))` 會告訴編譯器移除所有為了對齊 (alignment) 而加入的「填充位元組」(padding)，盡可能地壓縮結構的大小。

下面程式碼示範了 `packed` 屬性的作用，本文使用的開發環境為 MSYS2 UCRT64，編譯器為 GCC 15.1.0 (x86_64-w64-mingw32)，預設 C 標準為 C23：

```c=
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

struct pxx {
    uint32_t b;
    uint16_t f;
    uint8_t a;
    uint8_t c;
    uint8_t d;
    uint8_t e;
} __attribute__((packed));

struct sxx {
    uint8_t a;
    uint32_t b;
    uint8_t c;
    uint8_t d;
    uint8_t e;
    uint16_t f;
};

int main(int argc, char *argv[])
{
    struct pxx p;
    memset(&p, 0, sizeof(p));

    p.a = 0x11;
    p.b = 0x22334455;
    p.c = 0x66;
    p.d = 0x77;
    p.e = 0x88;
    p.f = 0x99AA;

    printf("sizeof(p):%lld\n", sizeof(p));
    printf("&p:%p\n", &p);
    printf("p.b: %p, %8x, %lld\n", &p.b, p.b, sizeof(p.b));
    printf("p.f: %p, %8x, %lld\n", &p.f, p.f, sizeof(p.f));
    printf("p.a: %p, %8x, %lld\n", &p.a, p.a, sizeof(p.a));
    printf("p.c: %p, %8x, %lld\n", &p.c, p.c, sizeof(p.c));
    printf("p.d: %p, %8x, %lld\n", &p.d, p.d, sizeof(p.d));
    printf("p.e: %p, %8x, %lld\n", &p.e, p.e, sizeof(p.e));

    uint32_t x = p.b;
    printf("\n");
    printf("x  : %x\n", x);
    printf("p.b: %x\n", p.b);
    printf("\n");

    struct sxx s;
    memset(&s, 0, sizeof(s));

    s.a = 0x11;
    s.b = 0x22334455;
    s.c = 0x66;
    s.d = 0x77;
    s.e = 0x88;
    s.f = 0x99AA;

    printf("sizeof(s):%lld\n", sizeof(s));
    printf("&s:%p\n", &s);
    printf("s.a: %p, %8x, %lld\n", &s.a, s.a, sizeof(p.a));
    printf("s.b: %p, %8x, %lld\n", &s.b, s.b, sizeof(p.b));
    printf("s.c: %p, %8x, %lld\n", &s.c, s.c, sizeof(p.c));
    printf("s.d: %p, %8x, %lld\n", &s.d, s.d, sizeof(p.d));
    printf("s.e: %p, %8x, %lld\n", &s.e, s.e, sizeof(p.e));
    printf("s.f: %p, %8x, %lld\n", &s.f, s.f, sizeof(p.f));

    return 0;
}
```

執行結果為：

```
# ./bin/c-test.exe
sizeof(p):10
&p:000000E34CFFFAA2
p.b: 000000E34CFFFAA2, 22334455, 4
p.f: 000000E34CFFFAA6,     99aa, 2
p.a: 000000E34CFFFAA8,       11, 1
p.c: 000000E34CFFFAA9,       66, 1
p.d: 000000E34CFFFAAA,       77, 1
p.e: 000000E34CFFFAAB,       88, 1

x  : 22334455
p.b: 22334455

sizeof(s):16
&s:000000E34CFFFA90
s.a: 000000E34CFFFA90,       11, 1
s.b: 000000E34CFFFA94, 22334455, 4
s.c: 000000E34CFFFA98,       66, 1
s.d: 000000E34CFFFA99,       77, 1
s.e: 000000E34CFFFA9A,       88, 1
s.f: 000000E34CFFFA9C,     99aa, 2
```

## `p` 的位置在哪？

```
subq	$64, %rsp
```

當 `main` 函數開始執行時，`subq $64, %rsp` 指令會為所有的區域變數 (包括 `p`, `s`, `x` 等) 在 stack 上保留 64 bytes 的空間。編譯器的工作就是決定這 64 bytes 空間中，哪個位元組要放哪個變數。

```
leaq -14(%rbp), %rax
```

在這個例子中，編譯器決定把 `p` 變數（我們知道它有 10 bytes）放在相對於 `%rbp` (frame pointer) 位移 -14 的地方。

```c
memset(&p, 0, sizeof(p));
```

```
# main.c:27:     memset(&p, 0, sizeof(p));
.loc 1 27 5
leaq	-14(%rbp), %rax	 #, tmp123
movl	$10, %r8d	 #,
movl	$0, %edx	 #,
movq	%rax, %rcx	 # tmp123,
call	memset	 #
```

## `b` 的地址會是 4-byte align 嗎？

`b` 的地址是 `struct pxx` 變數本身的地址，因為 `b` 是第一個成員，offset 為 0。

`__attribute__((packed))` 會產生兩個主要影響：
1. `packed` 結構的總大小不一定會是 4 的倍數。
2. 結構本身的對齊要求 (alignment requirement) 被降低。

我們可以透過宣告存放 `packed` 結構的陣列來說明：

```c
struct pxx my_array[2];
```

假設 `my_array[0]` 的地址，也就是 `my_array[0].b` 的地址，恰好是 4-byte aligned，例如 0x1000。則 `my_array[1]` 的地址將會是 0x1000 + `sizeof(struct pxx)`，也就是 0x1000 + 10 = 0x100A。0x100A 不是 4 的倍數，因此 `my_array[1].b` 沒有 4-byte align。

`packed` 屬性不僅移除了 padding，它還將整個 `struct pxx` 的對齊要求降至 1 byte。

這意味著，即使我們只宣告一個單一變數：

```c
void my_function() {
    uint8_t some_char;
    struct pxx my_var; // 編譯器沒有義務將 my_var 放在 4-byte 邊界上
}
```

編譯器完全可能將 `my_var` 緊跟在 `some_char` 後面存放，例如 `some_char` 在 0x0FFF，`my_var` 就可能從 0x1000 開始（這種情況是 aligned）。但也可能 `some_char` 在 0x1000，而 `my_var` 從 0x1001 開始（這種情況 `my_var.b` 就是 unaligned）。

因為 `packed` 屬性的緣故，`b` 的地址不保證一定是 4-byte aligned。在陣列中幾乎可以肯定會遇到 unaligned 的情況。

## 程式碼 `main.s` 的說明

```=
	.file	"main.c"
 # GNU C23 (Rev5, Built by MSYS2 project) version 15.1.0 (x86_64-w64-mingw32)
 #	compiled by GNU C version 15.1.0, GMP version 6.3.0, MPFR version 4.2.2, MPC version 1.3.1, isl version isl-0.27-GMP

 # GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
 # options passed: -mtune=generic -march=nocona -g0 -O0
	.text
	.section .rdata,"dr"
.LC0:
	.ascii "sizeof(p):%lld\12\0"
.LC1:
	.ascii "&p:%p\12\0"
.LC2:
	.ascii "p.b: %p, %8x, %lld\12\0"
.LC3:
	.ascii "p.f: %p, %8x, %lld\12\0"
.LC4:
	.ascii "p.a: %p, %8x, %lld\12\0"
.LC5:
	.ascii "p.c: %p, %8x, %lld\12\0"
.LC6:
	.ascii "p.d: %p, %8x, %lld\12\0"
.LC7:
	.ascii "p.e: %p, %8x, %lld\12\0"
.LC8:
	.ascii "x  : %x\12\0"
.LC9:
	.ascii "p.b: %x\12\0"
.LC10:
	.ascii "sizeof(s):%lld\12\0"
.LC11:
	.ascii "&s:%p\12\0"
.LC12:
	.ascii "s.a: %p, %8x, %lld\12\0"
.LC13:
	.ascii "s.b: %p, %8x, %lld\12\0"
.LC14:
	.ascii "s.c: %p, %8x, %lld\12\0"
.LC15:
	.ascii "s.d: %p, %8x, %lld\12\0"
.LC16:
	.ascii "s.e: %p, %8x, %lld\12\0"
.LC17:
	.ascii "s.f: %p, %8x, %lld\12\0"
	.text
	.globl	main
	.def	main;	.scl	2;	.type	32;	.endef
	.seh_proc	main
main:
	pushq	%rbp	 #
	.seh_pushreg	%rbp
	movq	%rsp, %rbp	 #,
	.seh_setframe	%rbp, 0
	subq	$64, %rsp	 #,
	.seh_stackalloc	64
	.seh_endprologue
	movl	%ecx, 16(%rbp)	 # argc, argc
	movq	%rdx, 24(%rbp)	 # argv, argv
 # main.c:25: {
	call	__main	 #
 # main.c:27:     memset(&p, 0, sizeof(p));
	leaq	-14(%rbp), %rax	 #, tmp123
	movl	$10, %r8d	 #,
	movl	$0, %edx	 #,
	movq	%rax, %rcx	 # tmp123,
	call	memset	 #
 # main.c:29:     p.a = 0x11;
	movb	$17, -8(%rbp)	 #, p.a
 # main.c:30:     p.b = 0x22334455;
	movl	$573785173, -14(%rbp)	 #, p.b
 # main.c:31:     p.c = 0x66;
	movb	$102, -7(%rbp)	 #, p.c
 # main.c:32:     p.d = 0x77;
	movb	$119, -6(%rbp)	 #, p.d
 # main.c:33:     p.e = 0x88;
	movb	$-120, -5(%rbp)	 #, p.e
 # main.c:34:     p.f = 0x99AA;
	movw	$-26198, -10(%rbp)	 #, p.f
 # main.c:36:     printf("sizeof(p):%lld\n", sizeof(p));
	leaq	.LC0(%rip), %rax	 #, tmp124
	movl	$10, %edx	 #,
	movq	%rax, %rcx	 # tmp124,
	call	printf	 #
 # main.c:37:     printf("&p:%p\n", &p);
	leaq	-14(%rbp), %rax	 #, tmp125
	leaq	.LC1(%rip), %rcx	 #, tmp126
	movq	%rax, %rdx	 # tmp125,
	call	printf	 #
 # main.c:38:     printf("p.b: %p, %8x, %lld\n", &p.b, p.b, sizeof(p.b));
	movl	-14(%rbp), %edx	 # p.b, _1
	leaq	-14(%rbp), %rax	 #, tmp127
	leaq	.LC2(%rip), %rcx	 #, tmp128
	movl	$4, %r9d	 #,
	movl	%edx, %r8d	 # _1,
	movq	%rax, %rdx	 # tmp127,
	call	printf	 #
 # main.c:39:     printf("p.f: %p, %8x, %lld\n", &p.f, p.f, sizeof(p.f));
	movzwl	-10(%rbp), %eax	 # p.f, _2
 # main.c:39:     printf("p.f: %p, %8x, %lld\n", &p.f, p.f, sizeof(p.f));
	movzwl	%ax, %ecx	 # _2, _3
	leaq	-14(%rbp), %rax	 #, tmp129
	leaq	4(%rax), %rdx	 #, tmp130
	leaq	.LC3(%rip), %rax	 #, tmp131
	movl	$2, %r9d	 #,
	movl	%ecx, %r8d	 # _3,
	movq	%rax, %rcx	 # tmp131,
	call	printf	 #
 # main.c:40:     printf("p.a: %p, %8x, %lld\n", &p.a, p.a, sizeof(p.a));
	movzbl	-8(%rbp), %eax	 # p.a, _4
 # main.c:40:     printf("p.a: %p, %8x, %lld\n", &p.a, p.a, sizeof(p.a));
	movzbl	%al, %ecx	 # _4, _5
	leaq	-14(%rbp), %rax	 #, tmp132
	leaq	6(%rax), %rdx	 #, tmp133
	leaq	.LC4(%rip), %rax	 #, tmp134
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _5,
	movq	%rax, %rcx	 # tmp134,
	call	printf	 #
 # main.c:41:     printf("p.c: %p, %8x, %lld\n", &p.c, p.c, sizeof(p.c));
	movzbl	-7(%rbp), %eax	 # p.c, _6
 # main.c:41:     printf("p.c: %p, %8x, %lld\n", &p.c, p.c, sizeof(p.c));
	movzbl	%al, %ecx	 # _6, _7
	leaq	-14(%rbp), %rax	 #, tmp135
	leaq	7(%rax), %rdx	 #, tmp136
	leaq	.LC5(%rip), %rax	 #, tmp137
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _7,
	movq	%rax, %rcx	 # tmp137,
	call	printf	 #
 # main.c:42:     printf("p.d: %p, %8x, %lld\n", &p.d, p.d, sizeof(p.d));
	movzbl	-6(%rbp), %eax	 # p.d, _8
 # main.c:42:     printf("p.d: %p, %8x, %lld\n", &p.d, p.d, sizeof(p.d));
	movzbl	%al, %ecx	 # _8, _9
	leaq	-14(%rbp), %rax	 #, tmp138
	leaq	8(%rax), %rdx	 #, tmp139
	leaq	.LC6(%rip), %rax	 #, tmp140
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _9,
	movq	%rax, %rcx	 # tmp140,
	call	printf	 #
 # main.c:43:     printf("p.e: %p, %8x, %lld\n", &p.e, p.e, sizeof(p.e));
	movzbl	-5(%rbp), %eax	 # p.e, _10
 # main.c:43:     printf("p.e: %p, %8x, %lld\n", &p.e, p.e, sizeof(p.e));
	movzbl	%al, %ecx	 # _10, _11
	leaq	-14(%rbp), %rax	 #, tmp141
	leaq	9(%rax), %rdx	 #, tmp142
	leaq	.LC7(%rip), %rax	 #, tmp143
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _11,
	movq	%rax, %rcx	 # tmp143,
	call	printf	 #
 # main.c:45:     uint32_t x = p.b;
	movl	-14(%rbp), %eax	 # p.b, tmp144
	movl	%eax, -4(%rbp)	 # tmp144, x
 # main.c:46:     printf("\n");
	movl	$10, %ecx	 #,
	call	putchar	 #
 # main.c:47:     printf("x  : %x\n", x);
	movl	-4(%rbp), %eax	 # x, tmp145
	leaq	.LC8(%rip), %rcx	 #, tmp146
	movl	%eax, %edx	 # tmp145,
	call	printf	 #
 # main.c:48:     printf("p.b: %x\n", p.b);
	movl	-14(%rbp), %eax	 # p.b, _12
	leaq	.LC9(%rip), %rcx	 #, tmp147
	movl	%eax, %edx	 # _12,
	call	printf	 #
 # main.c:49:     printf("\n");
	movl	$10, %ecx	 #,
	call	putchar	 #
 # main.c:52:     memset(&s, 0, sizeof(s));
	leaq	-32(%rbp), %rax	 #, tmp148
	movl	$16, %r8d	 #,
	movl	$0, %edx	 #,
	movq	%rax, %rcx	 # tmp148,
	call	memset	 #
 # main.c:54:     s.a = 0x11;
	movb	$17, -32(%rbp)	 #, s.a
 # main.c:55:     s.b = 0x22334455;
	movl	$573785173, -28(%rbp)	 #, s.b
 # main.c:56:     s.c = 0x66;
	movb	$102, -24(%rbp)	 #, s.c
 # main.c:57:     s.d = 0x77;
	movb	$119, -23(%rbp)	 #, s.d
 # main.c:58:     s.e = 0x88;
	movb	$-120, -22(%rbp)	 #, s.e
 # main.c:59:     s.f = 0x99AA;
	movw	$-26198, -20(%rbp)	 #, s.f
 # main.c:61:     printf("sizeof(s):%lld\n", sizeof(s));
	leaq	.LC10(%rip), %rax	 #, tmp149
	movl	$16, %edx	 #,
	movq	%rax, %rcx	 # tmp149,
	call	printf	 #
 # main.c:62:     printf("&s:%p\n", &s);
	leaq	-32(%rbp), %rax	 #, tmp150
	leaq	.LC11(%rip), %rcx	 #, tmp151
	movq	%rax, %rdx	 # tmp150,
	call	printf	 #
 # main.c:63:     printf("s.a: %p, %8x, %lld\n", &s.a, s.a, sizeof(p.a));
	movzbl	-32(%rbp), %eax	 # s.a, _13
 # main.c:63:     printf("s.a: %p, %8x, %lld\n", &s.a, s.a, sizeof(p.a));
	movzbl	%al, %edx	 # _13, _14
	leaq	-32(%rbp), %rax	 #, tmp152
	leaq	.LC12(%rip), %rcx	 #, tmp153
	movl	$1, %r9d	 #,
	movl	%edx, %r8d	 # _14,
	movq	%rax, %rdx	 # tmp152,
	call	printf	 #
 # main.c:64:     printf("s.b: %p, %8x, %lld\n", &s.b, s.b, sizeof(p.b));
	movl	-28(%rbp), %ecx	 # s.b, _15
	leaq	-32(%rbp), %rax	 #, tmp154
	leaq	4(%rax), %rdx	 #, tmp155
	leaq	.LC13(%rip), %rax	 #, tmp156
	movl	$4, %r9d	 #,
	movl	%ecx, %r8d	 # _15,
	movq	%rax, %rcx	 # tmp156,
	call	printf	 #
 # main.c:65:     printf("s.c: %p, %8x, %lld\n", &s.c, s.c, sizeof(p.c));
	movzbl	-24(%rbp), %eax	 # s.c, _16
 # main.c:65:     printf("s.c: %p, %8x, %lld\n", &s.c, s.c, sizeof(p.c));
	movzbl	%al, %ecx	 # _16, _17
	leaq	-32(%rbp), %rax	 #, tmp157
	leaq	8(%rax), %rdx	 #, tmp158
	leaq	.LC14(%rip), %rax	 #, tmp159
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _17,
	movq	%rax, %rcx	 # tmp159,
	call	printf	 #
 # main.c:66:     printf("s.d: %p, %8x, %lld\n", &s.d, s.d, sizeof(p.d));
	movzbl	-23(%rbp), %eax	 # s.d, _18
 # main.c:66:     printf("s.d: %p, %8x, %lld\n", &s.d, s.d, sizeof(p.d));
	movzbl	%al, %ecx	 # _18, _19
	leaq	-32(%rbp), %rax	 #, tmp160
	leaq	9(%rax), %rdx	 #, tmp161
	leaq	.LC15(%rip), %rax	 #, tmp162
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _19,
	movq	%rax, %rcx	 # tmp162,
	call	printf	 #
 # main.c:67:     printf("s.e: %p, %8x, %lld\n", &s.e, s.e, sizeof(p.e));
	movzbl	-22(%rbp), %eax	 # s.e, _20
 # main.c:67:     printf("s.e: %p, %8x, %lld\n", &s.e, s.e, sizeof(p.e));
	movzbl	%al, %ecx	 # _20, _21
	leaq	-32(%rbp), %rax	 #, tmp163
	leaq	10(%rax), %rdx	 #, tmp164
	leaq	.LC16(%rip), %rax	 #, tmp165
	movl	$1, %r9d	 #,
	movl	%ecx, %r8d	 # _21,
	movq	%rax, %rcx	 # tmp165,
	call	printf	 #
 # main.c:68:     printf("s.f: %p, %8x, %lld\n", &s.f, s.f, sizeof(p.f));
	movzwl	-20(%rbp), %eax	 # s.f, _22
 # main.c:68:     printf("s.f: %p, %8x, %lld\n", &s.f, s.f, sizeof(p.f));
	movzwl	%ax, %ecx	 # _22, _23
	leaq	-32(%rbp), %rax	 #, tmp166
	leaq	12(%rax), %rdx	 #, tmp167
	leaq	.LC17(%rip), %rax	 #, tmp168
	movl	$2, %r9d	 #,
	movl	%ecx, %r8d	 # _23,
	movq	%rax, %rcx	 # tmp168,
	call	printf	 #
 # main.c:70:     return 0;
	movl	$0, %eax	 #, _60
 # main.c:71: }
	addq	$64, %rsp	 #,
	popq	%rbp	 #
	ret
	.seh_endproc
	.def	__main;	.scl	2;	.type	32;	.endef
	.ident	"GCC: (Rev5, Built by MSYS2 project) 15.1.0"
	.def	memset;	.scl	2;	.type	32;	.endef
	.def	printf;	.scl	2;	.type	32;	.endef
	.def	putchar;	.scl	2;	.type	32;	.endef
```

### 1. 檔案標頭與編譯器選項

```
	.file	"main.c"
 # GNU C23 (Rev5, Built by MSYS2 project) version 15.1.0 (x86_64-w64-mingw32)
 #	compiled by GNU C version 15.1.0, GMP version 6.3.0, MPFR version 4.2.2, MPC version 1.3.1, isl version isl-0.27-GMP

 # GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
 # options passed: -mtune=generic -march=nocona -g0 -O0
```

* `.file` 是 metadata，說明了原始檔案是 `main.c`。
* 註解的部分則顯示了編譯器版本以及傳入的編譯器選項。

### 2. 唯讀資料區段 `.rdata`

```
	.text
	.section .rdata,"dr"
.LC0:
	.ascii "sizeof(p):%lld\12\0"
.LC1:
	.ascii "&p:%p\12\0"
.LC2:
	.ascii "p.b: %p, %8x, %lld\12\0"
.LC3:
	.ascii "p.f: %p, %8x, %lld\12\0"
.LC4:
	.ascii "p.a: %p, %8x, %lld\12\0"
.LC5:
	.ascii "p.c: %p, %8x, %lld\12\0"
.LC6:
	.ascii "p.d: %p, %8x, %lld\12\0"
.LC7:
	.ascii "p.e: %p, %8x, %lld\12\0"
.LC8:
	.ascii "x  : %x\12\0"
.LC9:
	.ascii "p.b: %x\12\0"
.LC10:
	.ascii "sizeof(s):%lld\12\0"
.LC11:
	.ascii "&s:%p\12\0"
.LC12:
	.ascii "s.a: %p, %8x, %lld\12\0"
.LC13:
	.ascii "s.b: %p, %8x, %lld\12\0"
.LC14:
	.ascii "s.c: %p, %8x, %lld\12\0"
.LC15:
	.ascii "s.d: %p, %8x, %lld\12\0"
.LC16:
	.ascii "s.e: %p, %8x, %lld\12\0"
.LC17:
	.ascii "s.f: %p, %8x, %lld\12\0"
```

* `.rdata` 存放 read-only data，在這個程式中主要都是放 `printf(...)` 所需要的格式化字串。
* `.LC0`, `.LC1` 等是編譯器產生的**標籤**，用來在程式碼(組合語言)中引用這些字串。
* `\12` 是 `\n` (換行符) 的八進位表示法，`\0` 是 C 語言字串的結尾符。

### 3. 函數前導 (Prologue) 與堆疊 (Stack) 設定

```
	.text
	.globl	main
	.def	main;	.scl	2;	.type	32;	.endef
	.seh_proc	main
main:
	pushq	%rbp	         # 備份 caller 的 frame pointer (%rbp)。
	.seh_pushreg	%rbp
	movq	%rsp, %rbp	 # 將目前的 stack pointer (%rsp) 備份到 frame pointer (%rbp) 中。
	.seh_setframe	%rbp, 0
	subq	$64, %rsp	 # 將 stack pointer 減去 64 bytes，為所有的區域變數預留空間。
	.seh_stackalloc	64
	.seh_endprologue
	movl	%ecx, 16(%rbp)	 # argc, 儲存傳入的第1個參數 `argc`。
	movq	%rdx, 24(%rbp)	 # argv, 儲存傳入的第2個參數 `argv`。
 # main.c:25: {
	call	__main	         # 呼叫 C 語言執行環境(runtime)的初始化。
```

* main: 是函數的起點。
* `pushq %rbp` 和 `movq %rsp, %rbp` 是 x86-64 架構中建立 stack frame 的標準作法。`%rbp` 現在是 main 函數的 frame pointer，所有的區域變數都將透過 `%rbp` 減去位移量來存取。
* `subq $64, %rsp` 這行指令在 stack 上分配了 64 bytes 的空間。main 函數中的所有區域變數 (包括 `p`, `s`, `x`) 都會被放在這 64 bytes 的空間內。

### 4. packed 結構 'p' 的操作

```
# main.c:27:     memset(&p, 0, sizeof(p));
	leaq	-14(%rbp), %rax	     # 將 `p` 的位址 (%rbp - 14) 載入到 %rax 中
	movl	$10, %r8d	         # 設定第 3 個參數，size = 10，到 %r8d
	movl	$0, %edx	         # 設定第 2 個參數，value = 0，到 %edx
	movq	%rax, %rcx	         # 設定第 1 個參數，dest = p 的位址，到 %rcx
	call	memset	                 # 呼叫 memset(%rcx, %edx, %r8d)
 # main.c:29:     p.a = 0x11;
	movb	$17, -8(%rbp)	         # 寫入 1 byte (17 = 0x11) 到 (%rbp - 8)
 # main.c:30:     p.b = 0x22334455;
	movl	$573785173, -14(%rbp)	 # 寫入 4 bytes (long) 到 (%rbp - 14)
 # main.c:31:     p.c = 0x66;
	movb	$102, -7(%rbp)	         # 寫入 1 byte 到 (%rbp - 7)
 # main.c:32:     p.d = 0x77;
	movb	$119, -6(%rbp)	         # 寫入 1 byte 到 (%rbp - 6)
 # main.c:33:     p.e = 0x88;
	movb	$-120, -5(%rbp)	         # 寫入 1 byte 到 (%rbp - 5)
 # main.c:34:     p.f = 0x99AA;
	movw	$-26198, -10(%rbp)	 # 寫入 2 bytes (word) 到 (%rbp - 10)
```

結構 `p` 的memory layout 整理如下：
* `p.b` (4 bytes) 被放在 `-14(%rbp)`。
* `p.f` (2 bytes) 被放在 `-10(%rbp)` ( = -14 + 4 )。
* `p.a` (1 byte) 被放在 `-8(%rbp)` ( = -10 + 2 )。
* `p.c` (1 byte) 被放在 `-7(%rbp)` ( = -8 + 1 )。
* `p.d` (1 byte) 被放在 `-6(%rbp)` ( = -7 + 1 )。
* `p.e` (1 byte) 被放在 `-5(%rbp)` ( = -6 + 1 )。

---

* `leaq` (Load Effective Address) 指令的用途是「計算有效地址」。它不會真的去讀取記憶體，只是做數學計算。
* 從 `movl $10, %r8d` 這行指令可以知道 ，`sizeof(p)` 是 10 bytes。
* `p` 的起始位址是 `-14(%rbp)`。`%rbp` 通常是 16-byte aligned，所以 `%rbp` - 14 不會是 4-byte aligned。
* `movl	$573785173, -14(%rbp)` 這行指令證實了編譯器正在對一個非 4-byte aligned 的位址執行 32-bit (4-byte) 寫入。

### 5. 一般結構 's' 的操作

```
 # main.c:52:     memset(&s, 0, sizeof(s));
	leaq	-32(%rbp), %rax	 # 將 `s` 的位址 (%rbp - 32) 載入到 %rax 中
	movl	$16, %r8d	         # 設定第 3 個參數，size = 16，到 %r8d
	movl	$0, %edx	         # 設定第 2 個參數，value = 0，到 %edx
	movq	%rax, %rcx	         # 設定第 1 個參數，dest = s 的位址，到 %rcx
	call	memset	                 # 呼叫 memset(%rcx, %edx, %r8d)
 # main.c:54:     s.a = 0x11;
	movb	$17, -32(%rbp)	         # 寫入 1 byte 到 (rbp-32)
 # main.c:55:     s.b = 0x22334455;
	movl	$573785173, -28(%rbp)	 # 寫入 4 bytes 到 (rbp-28)
 # main.c:56:     s.c = 0x66;
	movb	$102, -24(%rbp)	         # 寫入 1 byte 到 (rbp-24)
 # main.c:57:     s.d = 0x77;
	movb	$119, -23(%rbp)	         #, s.d
 # main.c:58:     s.e = 0x88;
	movb	$-120, -22(%rbp)	 #, s.e
 # main.c:59:     s.f = 0x99AA;
	movw	$-26198, -20(%rbp)	 #, s.f
```

結構 `s` 的memory layout 整理如下：
* `s` 的起始位址被放在 `-32(%rbp)`。`%rbp` - 32 保證是至少 4-byte aligned。
* `s.a` (1 bytes) 被放在 `-32(%rbp)`。
* `s.b` (4 bytes) 被放在 `-28(%rbp)` ( = -32 + 4 )。
* `s.c` (1 byte) 被放在 `-24(%rbp)` ( = -28 + 4 )。
* `s.d` (1 byte) 被放在 `-23(%rbp)` ( = -24 + 1 )。
* `s.e` (1 byte) 被放在 `-22(%rbp)` ( = -23 + 1 )。
* `s.f` (2 byte) 被放在 `-20(%rbp)` ( = -22 + 2 )。

---

* 從 `movl $16, %r8d` 這行指令可以知道 ，`sizeof(s)` 是 16 bytes。
* 為了讓 `s.b` (4 bytes) 能對齊，編譯器在 `s.a` 和 `s.b` 之間自動插入了 3 bytes 的 padding。
* `movl	$573785173, -28(%rbp)` 這行指令存取的位址 `%rbp` - 28 保證是 4-byte aligned，因此 CPU 執行效能最高。

### 6. 函數結尾 (Epilogue)

```
 # main.c:70:     return 0;
	movl	$0, %eax	 # 將傳回值 0 放入 %eax 暫存器。
 # main.c:71: }
	addq	$64, %rsp	 # 將 stack pointer 加上 64 bytes，釋放區域變數空間。
	popq	%rbp	         # 還原 caller 的 frame pointer。
	ret	                 # 返回至 caller 呼叫處。
	.seh_endproc
```

* 在 `movl $0, %eax` 這行指令中，`%eax` 是 x86-64 架構上預設的函數傳回值暫存器。
* `addq $64, %rsp` 和 `popq %rbp`：這兩行是函數前導 (Prologue) 的逆操作，它們會銷毀目前的 stack frame，並將 stack pointer 與 frame pointer 恢復到呼叫 `main` 函數之前的狀態。
* `ret`：從 `main` 函數返回。

## 使用 `packed` 的時機

`packed` 屬性是一個強大的工具，但它犧牲了效能來換取空間。

### 效能取捨 (Performance Trade-off)
1. CPU 存取速度：現代 CPU (如 x86-64) 被設計為在「對齊」的記憶體位址上讀取資料效率最高。例如，讀取一個 4-byte uint32_t 時，如果其位址是 4 的倍數 (如 0x...A94)，CPU 通常可以在一個時脈週期內完成。
2. Unaligned 存取懲罰：當 CPU 被要求從一個「未對齊」的位址 (如 0x...AA2) 讀取 4 bytes 資料時，它無法一次完成。CPU 必須執行額外的工作，例如：
    * 讀取兩次記憶體 (例如，一次讀取 0x...AA0 到 0x...AA3，另一次讀取 0x...AA4 到 0x...AA7)。
    * 透過內部位移和遮罩 (shifting and masking) 來組合出所需的 4 bytes 資料 (0x...AA2 到 0x...AA5)。
這個過程會比對齊存取慢上許多。

在本文的例子中，`movl $573785173, -14(%rbp)` 就是一個 unaligned 存取，它的效能會低於 `movl $573785173, -28(%rbp)`。

### 架構差異

值得注意的是，雖然 x86-64 (CISC) 允許 unaligned 存取 (但有效能懲罰)，許多 RISC 架構 (如 ARM) 預設情況下不允許 unaligned 存取。在這些平台上，嘗試讀取 unaligned 位址可能會觸發 Exception 並使程式崩潰。

### 合理的使用時機

基於上述的種種考量，我們不應該為了節省一點點記憶體空間而濫用 `packed` 屬性，它僅須在適當的時機使用，常見情境包括：
* **硬體暫存器映射**：當 C 結構需要直接映射 (map) 到一個硬體的暫存器佈局 (layout) 時，而該佈局不符合 C 語言的對齊規則。
* **封包定義**：定義通訊協定 (protocol) 的封包結構時，若需要確保沒有任何 padding 以符合協定規範的確切位元組佈局時，會利用 `packed` 屬性來優化/簡化程式碼設計。
* **檔案格式**：讀取或寫入特定且緊湊的二進位檔案格式 (例如，某些圖片檔、音訊檔的檔頭)。
* **系統互通性**：當需要與其他系統 (或使用不同編譯器、不同 pack 規則的程式) 交換資料時，`packed` 屬性可以確保雙方對記憶體佈局有相同的認知。

除非我們正在處理上述情況，否則應該優先使用 C 語言的預設對齊規則，因為這會是編譯器為我們的平台所做的最佳效能優化。

## 附錄 A: GCC Options

若想要閱讀帶有 C 註解的組合語言，但又不想產生額外除錯資訊的需求，編譯時我們可以使用下列 GCC 選項：

```
-g0 -O0 -fverbose-asm
```

詳細解釋見官方說明：

**-glevel**
**-ggdblevel**
**-gvmslevel**

Request debugging information and also use level to specify how much information.
The default level is 2.
* Level 0 produces no debug information at all. Thus, `-g0` negates `-g`.
* Level 1 produces minimal information, enough for making backtraces in parts
of the program that you don’t plan to debug. This includes descriptions of
functions and external variables, and line number tables, but no information
about local variables.
* Level 3 includes extra information, such as all the macro definitions present in
the program. Some debuggers support macro expansion when you use `-g3`.
If you use multiple `-g` options, with or without level numbers, the last such
option is the one that is effective.

`-gdwarf` does not accept a concatenated debug level, to avoid confusion with
`-gdwarf-level`. Instead use an additional `-glevel` option to change the debug
level for DWARF.

**-fverbose-asm**

Put extra commentary information in the generated assembly code to make it
more readable. This option is generally only of use to those who actually need
to read the generated assembly code (perhaps while debugging the compiler
itself).
`-fno-verbose-asm`, the default, causes the extra information to be omitted and
is useful when comparing two assembler files.

The added comments include:
* information on the compiler version and command-line options,
* the source code lines associated with the assembly instructions, in the form
FILENAME:LINENUMBER:CONTENT OF LINE,
* hints on which high-level expressions correspond to the various assembly
instruction operands.

For example, given this C source file:

```c
int test (int n)
{
    int i;
    int total = 0;
    for (i = 0; i < n; i++)
    total += i * i;
    return total;
}
```

compiling to (x86 64) assembly via -S and emitting the result direct to stdout
via -o -

```
gcc -S test.c -fverbose-asm -Os -o -
```

gives output similar to this:

```
.file "test.c"
# GNU C11 (GCC) version 7.0.0 20160809 (experimental) (x86_64-pc-linux-gnu)
[...snip...]
# options passed:
[...snip...]
.text
.globl test
.type test, @function
test:
.LFB0:
.cfi_startproc
# test.c:4: int total = 0;
xorl %eax, %eax # <retval>
# test.c:6: for (i = 0; i < n; i++)
xorl %edx, %edx # i
.L2:
# test.c:6: for (i = 0; i < n; i++)
cmpl %edi, %edx # n, i
jge .L5 #,
# test.c:7: total += i * i;
movl %edx, %ecx # i, tmp92
imull %edx, %ecx # i, tmp92
# test.c:6: for (i = 0; i < n; i++)
incl %edx # i
# test.c:7: total += i * i;
addl %ecx, %eax # tmp92, <retval>
jmp .L2 #
.L5:
# test.c:10: }
ret
.cfi_endproc
.LFE0:
.size test, .-test
.ident "GCC: (GNU) 7.0.0 20160809 (experimental)"
.section .note.GNU-stack,"",@progbits
```

The comments are intended for humans rather than machines and hence the
precise format of the comments is subject to change.

## References

Using the GNU Compiler Collection For gcc version 15.2.0
https://gcc.gnu.org/onlinedocs/gcc-15.2.0/gcc/

###### tags: `GCC` `packed` `C` `MSYS2` `mingw-w64`
