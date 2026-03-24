# 关于 x86 与 ARM 的状态标志

## x86架构下的标志位

在 Intel 软件开发者指南的第1卷中3.4.3小节（EFLAGS Register）中描述了关于x86处理器中的6种状态标志。这些标志都位于x86的 **`EFLAGS`** 寄存器中，其官方描述如下：

- CF（比特0） 进位标志—— 如果一个算术操作对该计算结果的最高有效位产生了一个进位或是借位，则该标志位被置1；否则对该标志位清零。这个标志用于指明对一个 **无符号整数** 算术操作的溢出条件。它也被用于多精度算术操作中。
- PF（比特2） 奇偶标志——如果计算结果的最低有效字节（即低8位）包含了偶数个比特1，那么该标志位被置1；否则对该标志位清零。
- AF（比特4） 辅助进位标志——如果一个算术操作对该计算结果的比特3产生了一个进位或是借位，那么该标志位被置1；否则该标志位被清零。（注意，这个标志也针对于无符号整数的计算。此外，在8086架构下或是在虚拟8086模式下，对16位寄存器的操作结果可能会是针对第7比特是否有进位或是借位进行判定，但在现代化的 x86_64 模式下，无论寄存器是8位、16位、32位还是64位，都是针对比特3进行判定，即 **半字节**，**nybble** 的无符号进位或借位。）此标志被用于“二进制编码的十进制数”（BCD码）的算术计算（比如 **`AAA`**、**`AAS`** 等指令）。
- ZF（比特6） 零标志位——如果计算结果为0，那么该标志位被置1；否则该标志位被清零。
- OF（比特11） 溢出标志——如果计算得到的整数结果对于一个正整数而言过大，或是对于一个负整数而言过小（即该负数的绝对值过大），使得该整数无法落于目的操作数的表示范围，那么该标志位被置1；否则该标志位被清零。此标志指明了对 **带符号整数**（二进制补码）的算术操作的溢出条件。

本博文想着重介绍的是x86与ARM两大处理器架构会有不一样行为的两个标志位—— CF 和 OF，对应于 ARM 中的 C 和 V 两个标志。

由于现代主流处理器架构都是采用二进制补码形式来做各种算术逻辑操作，因此 CPU 的一个加法或减法操作对于程序员来说，既可以将它视作为一个 **带符号** 的算术计算，也可以将它视作为一个 **无符号** 的算术计算，而是否为带符号的取决于我们后续逻辑对该算术计算的结果做何处理。比如我们C语言中写这么段代码：

```c
int a = -1, b = 1;
a -= b;
```
以及
```c
unsigned a = -1U, b = 1;
a -= b;
```

这两段代码结束后，a的值都是 `0xffff'fffe`，但对于带符号数而言，其大小为-2，而对于无符号数而言，其大小为 2<sup>32</sup> - 2。因此如果后续要拿这个 a 与0比较大小的话，带符号数与无符号数的结果肯定是不一样的。

因此这里就引出了，我一个处理器中如何判别一个无符号数的溢出和带符号数的溢出呢？在x86处理器中就引入了 **`CF`** 标志（对应于ARM架构的 **`C`** 标志）来指示当前计算结果是否为 **无符号整数** 的溢出；而引入 **`OF`** 标志（对应于ARM架构的 **`V`** 标志）来指示当前计算结果是否为 **带符号整数** 的溢出。

对于32位无符号整数，其范围是 `[0, 0xffff'ffff]`，即 0 到 2<sup>32</sup> - 1。如果一个无符号整数的计算结果不在此范围内，则判定为溢出，CF 标志被置1。

对于带符号整数，其范围是 `[0x8000'0000, 0x7fff'ffff]`，即 -2<sup>31</sup> 到 2<sup>31</sup> - 1。如果一个带符号整数的计算结果不在此范围内，则判定为溢出，OF 标志被置1。

因此，CPU 会在执行一条算术操作之后同时以无符号数和带符号数这两种视角进行判断，从而分别给出计算结果对 CF 标志以及 OF 标志的设置条件。

下面给出x86处理器上主要针对 CF 与 OF 两个标志的设置条件样例。该项目基于 Win11 下 Visual Studio 2022 版，当然，各位使用 Visual Studio 2017 也没问题。

下面是main.c源文件的内容：

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>

extern void FlagsTestSet(uint8_t resultFlags[]);

static void OutputFlagBits(uint8_t flags)
{
    enum
    {
        CF_BIT = 1 << 0,
        PF_BIT = 1 << 2,
        OF_BIT = 1 << 3,
        AF_BIT = 1 << 4,
        ZF_BIT = 1 << 6,
        SF_BIT = 1 << 7
    };

    printf("CF = %d, PF = %d, OF = %d, AF = %d, ZF = %d, SF = %d\n",
        (flags & CF_BIT) != 0, (flags & PF_BIT) != 0, (flags & OF_BIT) != 0,
        (flags & AF_BIT) != 0, (flags & ZF_BIT) != 0, (flags & SF_BIT) != 0);
}

int main(void)
{
    uint8_t flags[16] = { 0 };
    FlagsTestSet(flags);

    printf("0x12345678 + 0x6789abcd: ");    // CF = 0, PF = 0, OF = 0, AF = 1, ZF = 0, SF = 0
    OutputFlagBits(flags[0]);

    printf("0x76543210 + 0x12345678: ");    // CF = 0, PF = 1, OF = 1, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[1]);

    printf("0x76543210 + 0x9abcdef0: ");    // CF = 1, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[2]);

    printf("0x87654321 + 0x12345678: ");    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[3]);

    printf("0x98765432 + 0x87654321: ");    // CF = 1, PF = 1, OF = 1, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[4]);

    printf("0x6789abcd - 0x12345678 = %d: ", 0x6789abcdU - 0x12345678U);    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[5]);

    printf("0x12345678 - 0x76543210 = %d: ", 0x12345678U - 0x76543210U);    // CF = 1, PF = 0, OF = 0, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[6]);

    printf("0x76543210 - 0x9abcdef0 = %d: ", 0x76543210U - 0x9abcdef0U);    // CF = 1, PF = 0, OF = 1, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[7]);

    printf("0x87654321 - 0x01234567 = %d: ", 0x87654321U - 0x01234567U);    // CF = 0, PF = 0, OF = 0, AF = 1, ZF = 0, SF = 1
    OutputFlagBits(flags[8]);

    printf("0xff000000 - 0xfe000000 = %d: ", 0xff000000U - 0xfe000000U);    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[9]);
}
```

下面为 test.asm 汇编文件的代码

```nasm
.code

LOAD_FLAGS_AND_STORE    macro disp

    lahf
    seto    al
    shl     al, 3
    or      ah, al
    mov     byte ptr [rcx + disp], ah

endm

; void FlagsTestSet(uint8_t resultFlags[])
FlagsTestSet    proc public

    mov     eax, 12345678H
    mov     edx, 6789abcdH
    add     eax, edx
    LOAD_FLAGS_AND_STORE 0

    mov     eax, 76543210H
    mov     edx, 12345678H
    add     eax, edx
    LOAD_FLAGS_AND_STORE 1

    mov     eax, 76543210H
    mov     edx, 9abcdef0H
    add     eax, edx
    LOAD_FLAGS_AND_STORE 2

    mov     eax, 87654321H
    mov     edx, 12345678H
    add     eax, edx
    LOAD_FLAGS_AND_STORE 3

    mov     eax, 98765432H
    mov     edx, 87654321H
    add     eax, edx
    LOAD_FLAGS_AND_STORE 4

    mov     eax, 6789abcdH
    mov     edx, 12345678H
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 5

    mov     eax, 12345678H
    mov     edx, 76543210H
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 6

    mov     eax, 76543210H
    mov     edx, 9abcdef0H
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 7

    mov     eax, 87654321H
    mov     edx, 01234567H
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 8

    mov     eax, 0ff000000H
    mov     edx, 0fe000000H
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 9

    ret

FlagsTestSet    endp

end
```

对于上述代码，我们这里举两个例子来具体分析一下。

首先是 `0x98765432 + 0x87654321`。如果将它们视为无符号数，看最高位十六进制数，9+8 = 17，已经超过了15（0xf），所以一目了然作为无符号整数结果，它已经溢出了，因而 CF 标志置1。而将它作为带符号数来看，由于这两个整数最高位都是1，因此为了容易计算，我们不妨将它们分别取负，然后做加法，看看结果的绝对值是否超出了带符号整数的范围。`0x98765432` 取负之后结果为 `0x6789ABCE`；`0x87654321` 取负结果为`0x789ABCDF`。然后两者相加，一目了然，最高位十六进制数相加 6+7 远远超过了一个带符号正整数所能表达的最高位十六进制数为7的范围，因此 OF 标志也置1。

然后我们再看一个减法数据：`0x76543210 - 0x9abcdef0`。如果将它们看作为无符号数，显然，被减数的最高位十六进制为7，减数的最高位十六进制数为9，不够减，需要借位，因此CF标志为1。而如果将它们视作为带符号整数，减数部分由于是一个负数，所以我们不妨将它转为一个正整数然后再用加法计算，这么一来其实就变成了：`0x76543210 + 0x65432110`，跟上面一样，两者最高位十六进制数相加，7 + 6 超过了7，因此也就显然超过了带符号整数所能表示的范围，因此这里 OF 标志也被置1。

下面即将讲述 ARM 架构的 NZCV 标志，不过由于对于普通程序员而言，要测 ARM 架构处理器会比较麻烦，一般会利用自己手头上的手机，无论是 iOS 设备还是 Android 设备，因此，这边就以 Android 端为例，给大家呈现后续的 ARM 架构的标志 demo。为了能让大家与 x86 架构的标志进行对比，这里先给出 Android 端上 x86 处理器的测试代码。

以下是 native-lib.c 文件中的代码内容：

```c
#include <jni.h>
#include <stdbool.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <syslog.h>
#include <cpu-features.h>

extern void FlagsTestSet(uint8_t resultFlags[]);

static void OutputFlagBits(uint8_t flags)
{
#if __x86_64__
    enum
    {
        CF_BIT = 1 << 0,
        PF_BIT = 1 << 2,
        OF_BIT = 1 << 3,
        AF_BIT = 1 << 4,
        ZF_BIT = 1 << 6,
        SF_BIT = 1 << 7
    };

    syslog(LOG_INFO, "CF = %d, PF = %d, OF = %d, AF = %d, ZF = %d, SF = %d\n",
           (flags & CF_BIT) != 0, (flags & PF_BIT) != 0, (flags & OF_BIT) != 0,
           (flags & AF_BIT) != 0, (flags & ZF_BIT) != 0, (flags & SF_BIT) != 0);
#endif

#if __aarch64__
    enum
    {
        V_BIT = 1 << 0,
        C_BIT = 1 << 1,
        Z_BIT = 1 << 2,
        N_BIT = 1 << 3
    };

    syslog(LOG_INFO, "N = %d, Z = %d, C = %d, V = %d\n",
           (flags & N_BIT) != 0, (flags & Z_BIT) != 0,
           (flags & C_BIT) != 0, (flags & V_BIT) != 0);
#endif
}

static void FlagsTest(void)
{
    uint8_t flags[16] = { 0 };
    FlagsTestSet(flags);

    syslog(LOG_INFO, "0x12345678 + 0x6789abcd: ");    // CF = 0, PF = 0, OF = 0, AF = 1, ZF = 0, SF = 0
    OutputFlagBits(flags[0]);

    syslog(LOG_INFO, "0x76543210 + 0x12345678: ");    // CF = 0, PF = 1, OF = 1, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[1]);

    syslog(LOG_INFO, "0x76543210 + 0x9abcdef0: ");    // CF = 1, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[2]);

    syslog(LOG_INFO, "0x87654321 + 0x12345678: ");    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[3]);

    syslog(LOG_INFO,"0x98765432 + 0x87654321: ");    // CF = 1, PF = 1, OF = 1, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[4]);

    syslog(LOG_INFO, "0x6789abcd - 0x12345678 = %d: ", 0x6789abcdU - 0x12345678U);    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[5]);

    syslog(LOG_INFO, "0x12345678 - 0x76543210 = %d: ", 0x12345678U - 0x76543210U);    // CF = 1, PF = 0, OF = 0, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[6]);

    syslog(LOG_INFO, "0x76543210 - 0x9abcdef0 = %d: ", 0x76543210U - 0x9abcdef0U);    // CF = 1, PF = 0, OF = 1, AF = 0, ZF = 0, SF = 1
    OutputFlagBits(flags[7]);

    syslog(LOG_INFO, "0x87654321 - 0x01234567 = %d: ", 0x87654321U - 0x01234567U);    // CF = 0, PF = 0, OF = 0, AF = 1, ZF = 0, SF = 1
    OutputFlagBits(flags[8]);

    syslog(LOG_INFO, "0xff000000 - 0xfe000000 = %d: ", 0xff000000U - 0xfe000000U);    // CF = 0, PF = 1, OF = 0, AF = 0, ZF = 0, SF = 0
    OutputFlagBits(flags[9]);
}

JNIEXPORT jstring JNICALL
Java_com_codelearning_project_MainActivity_stringFromJNI(JNIEnv* env, jobject thisObj)
{
    FlagsTest();

    const int cpuCount = android_getCpuCount();
    const int apiLevel = android_get_device_api_level();
    syslog(LOG_INFO, "CPU logical processor count: %d, API level: %d\n", cpuCount, apiLevel);

    const char* hello = "Hello Android Native!!";
    return (*env)->NewStringUTF(env, hello);
}
```

以下则是 GAS（GNU Assembly）汇编源文件的代码，文件名为 asm_x64.S：

```nasm
.text
.align 4
.intel_syntax noprefix

.globl FlagsTestSet

.macro LOAD_FLAGS_AND_STORE  disp:req

    lahf
    seto    al
    shl     al, 3
    or      ah, al
    mov     byte ptr [rdi + \disp], ah

.endm

// void FlagsTestSet(uint8_t resultFlags[])
FlagsTestSet:

    mov     eax, 0x12345678
    mov     edx, 0x6789abcd
    add     eax, edx
    LOAD_FLAGS_AND_STORE 0

    mov     eax, 0x76543210
    mov     edx, 0x12345678
    add     eax, edx
    LOAD_FLAGS_AND_STORE 1

    mov     eax, 0x76543210
    mov     edx, 0x9abcdef0
    add     eax, edx
    LOAD_FLAGS_AND_STORE 2

    mov     eax, 0x87654321
    mov     edx, 0x12345678
    add     eax, edx
    LOAD_FLAGS_AND_STORE 3

    mov     eax, 0x98765432
    mov     edx, 0x87654321
    add     eax, edx
    LOAD_FLAGS_AND_STORE 4

    mov     eax, 0x6789abcd
    mov     edx, 0x12345678
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 5

    mov     eax, 0x12345678
    mov     edx, 0x76543210
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 6

    mov     eax, 0x76543210
    mov     edx, 0x9abcdef0
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 7

    mov     eax, 0x87654321
    mov     edx, 0x01234567
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 8

    mov     eax, 0xff000000
    mov     edx, 0xfe000000
    sub     eax, edx
    LOAD_FLAGS_AND_STORE 9

    ret
```

<br />

## ARM架构下的标志位

在ARMv8架构参考指南中的 B1.2.2小节（Process state, PSTATE）描述了关于处理状态中的条件标志，称为 **NZCV**。以下为官方文档描述：

影响标志的指令将会设置以下这些标志。它们是：

- N  负数条件标志。如果一条指令的计算结果被看作为一个二进制补码的带符号整数，那么如果该结果是个负数，PE将它置1；否则将它清0。
- Z  零条件标志。如果指令计算的结果为0，那么将它置1；否则将它清0。
- C  进位标志。如果一条指令的结果引发进位条件，则将它置1，比如一个 **加法计算结果的无符号整数的溢出**；否则将它清0。
- V  溢出条件标志。如果一条指令的计算结果引发溢出，那么将它置1，比如一个 **加法计算结果的带符号整数溢出**；否则将它清0。

这一段描述看起来与 x86 的 SF、ZF、CF 和 OF 没多大差异，而事实也确实如此。不过 ARM 对影响标志位的减法的实现却会造成 x86 与 ARM 两个架构之间 CF 标志位的变化差异。

ARM中像 **`SUBS`** 指令的操作其实是将减数（即第二个源操作数）进行取负操作，然后再用加法得到最终结果。比如，一个 10 - 8 这样的操作，在 ARM 中其实是 10 + (-8) 这一过程来完成的，因此计算结果的 C 标志会与 x86 的 CF 标志正好相反。

因为对于x86而言，10 - 8 就是一个普通的减法操作，而CF标志则是将操作数与结果都看作为无符号数，因此 10 - 8 不会对高位产生任何借位，所以 CF 标志位肯定为0。

而 ARM 的操作是如何呢？10 + (-8)，对于32位无符号数而言相当于 `10 + 0xffff'fff8`，那我们可以很清晰地看到，这两个无符号整数加法的结果已经超出了 `0xffff'ffff`，因此引发了进位，所以此时 CF 标志位为1。而若是将它们看作为带符号整数，显然 10 + (-8) 结果为 +2，在正整数的范围内，没有产生溢出，因此 V 标志位为0。

下面给出 ARMv8 架构下的测试代码。首先是 Android 中的 native-lib.c 的 `FlagsTest` 函数：

```c
static void FlagsTest(void)
{
    uint8_t flags[16] = { 0 };
    FlagsTestSet(flags);

    syslog(LOG_INFO, "0x12340000 + 0x67890000: ");    // N = 0, Z = 0, C = 0, V = 0
    OutputFlagBits(flags[0]);

    syslog(LOG_INFO, "0x76540000 + 0x12340000: ");    // N = 1, Z = 0, C = 0, V = 1
    OutputFlagBits(flags[1]);

    syslog(LOG_INFO, "0x76540000 + 0x9abc0000: ");    // N = 0, Z = 0, C = 1, V = 0
    OutputFlagBits(flags[2]);

    syslog(LOG_INFO, "0x87650000 + 0x12340000: ");    // N = 1, Z = 0, C = 0, V = 0
    OutputFlagBits(flags[3]);

    syslog(LOG_INFO,"0x98760000 + 0x87650000: ");     // N = 0, Z = 0, C = 1, V = 1
    OutputFlagBits(flags[4]);

    syslog(LOG_INFO, "0x67890000 - 0x12340000 = %d: ", 0x67890000U - 0x12340000U);    // N = 0, Z = 0, C = 1, V = 0
    OutputFlagBits(flags[5]);

    syslog(LOG_INFO, "0x12340000 - 0x76540000 = %d: ", 0x12340000U - 0x76540000U);    // N = 1, Z = 0, C = 0, V = 0
    OutputFlagBits(flags[6]);

    syslog(LOG_INFO, "0x76540000 - 0x9abc0000 = %d: ", 0x76540000U - 0x9abc0000U);    // N = 1, Z = 0, C = 0, V = 1
    OutputFlagBits(flags[7]);

    syslog(LOG_INFO, "0x87650000 - 0x01230000 = %d: ", 0x87650000U - 0x01230000U);    // N = 1, Z = 0, C = 1, V = 0
    OutputFlagBits(flags[8]);

    syslog(LOG_INFO, "0xff000000 - 0xfe000000 = %d: ", 0xff000000U - 0xfe000000U);    // N = 0, Z = 0, C = 1, V = 0
    OutputFlagBits(flags[9]);
}
```

然后是基于 ARM64 的 GAS 汇编代码，文件名为：asm_arm64.S：

```nasm
.text
.align 4
.globl FlagsTestSet

.macro LOAD_FLAGS_AND_STORE  disp:req

    mrs     x3, nzcv
    lsr     w3, w3, #28
    strb    w3, [X0, #\disp]

.endm

// void FlagsTestSet(uint8_t resultFlags[])
FlagsTestSet:

    movz    w1, 0x1234, lsl #16
    movz    w2, 0x6789, lsl #16
    adds    w3, w1, w2
    LOAD_FLAGS_AND_STORE 0

    movz    w1, 0x7654, lsl #16
    movz    w2, 0x1234, lsl #16
    adds    w3, w1, w2
    LOAD_FLAGS_AND_STORE 1

    movz    w1, 0x7654, lsl #16
    movz    w2, 0x9abc, lsl #16
    adds    w3, w1, w2
    LOAD_FLAGS_AND_STORE 2

    movz    w1, 0x8765, lsl #16
    movz    w2, 0x1234, lsl #16
    adds    w3, w1, w2
    LOAD_FLAGS_AND_STORE 3

    movz    w1, 0x9876, lsl #16
    movz    w2, 0x8765, lsl #16
    adds    w3, w1, w2
    LOAD_FLAGS_AND_STORE 4

    movz    w1, 0x6789, lsl #16
    movz    w2, 0x1234, lsl #16
    subs    w3, w1, w2
    LOAD_FLAGS_AND_STORE 5

    movz    w1, 0x1234, lsl #16
    movz    w2, 0x7654, lsl #16
    subs    w3, w1, w2
    LOAD_FLAGS_AND_STORE 6

    movz    w1, 0x7654, lsl #16
    movz    w2, 0x9abc, lsl #16
    subs    w3, w1, w2
    LOAD_FLAGS_AND_STORE 7

    movz    w1, 0x8765, lsl #16
    movz    w2, 0x0123, lsl #16
    subs    w3, w1, w2
    LOAD_FLAGS_AND_STORE 8

    movz    w1, 0xff00, lsl #16
    movz    w2, 0xfe00, lsl #16
    subs    w3, w1, w2
    LOAD_FLAGS_AND_STORE 9

    ret
```

大家对比一下 x86 的测试结果可以看到，对于加法计算，x86 的 CF 标志与 OF 标志可以跟 ARM64 的 C 与 V 标志完全对应上。而对于减法操作，x86 的 OF 标志与 ARM64 的 V 标志完全对应上，而 CF 标志则与 ARM64 的 C 标志正好相反。所以 ARM 中还引入了 **`CMN`** 指令（Compare Negative），表示对第二个比较操作数做取相反数操作（即添加一个负号），用于跟 x86 的 **`CMP`** 的操作结果对标志位的影响取得一致。


