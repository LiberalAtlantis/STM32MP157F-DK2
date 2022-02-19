# U-Boot源码分析

## U-Boot 第一阶段 (Stage1)

### u-boot.lds

文件位置 [arch/arm/cpu/u-boot.lds]

~~~c
//设置输出文件大小端格式
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
//设置文件以ARM可执行文件格式输出
OUTPUT_ARCH(arm)
//指令以_start标号开始
ENTRY(_start)
SECTIONS
{
 /DISCARD/ : { *(.rel._secure*) }
 . = 0x00000000;
 . = ALIGN(4);
 .text :
 {
  *(.__image_copy_start)
  *(.vectors)
  //text段以start.o开始
  arch/arm/cpu/armv7/start.o (.text*)
 }
 .__efi_runtime_start : {
  *(.__efi_runtime_start)
 }
 .efi_runtime : {
  *(.text.efi_runtime*)
  *(.rodata.efi_runtime*)
  *(.data.efi_runtime*)
 }
 .__efi_runtime_stop : {
  *(.__efi_runtime_stop)
 }
 .text_rest :
 {
  *(.text*)
 }
 .__secure_start
 : {
  KEEP(*(.__secure_start))
 }
 .secure_text 0x2FFC0000 :
  AT(ADDR(.__secure_start) + SIZEOF(.__secure_start))
 {
  *(._secure.text)
 }
 .secure_data : AT(LOADADDR(.secure_text) + SIZEOF(.secure_text))
 {
  *(._secure.data)
 }
 .secure_stack ALIGN(ADDR(.secure_data) + SIZEOF(.secure_data),
       CONSTANT(COMMONPAGESIZE)) (NOLOAD) :
  AT(LOADADDR(.secure_data) + SIZEOF(.secure_data))
 {
  KEEP(*(.__secure_stack_start))
  . = . + 4 * (1 << 10);
  . = ALIGN(CONSTANT(COMMONPAGESIZE));
  KEEP(*(.__secure_stack_end))
  ASSERT((. - ADDR(.secure_text)) <= 0x00040000,
         "Error: secure section exceeds secure memory size");
 }
 . = LOADADDR(.secure_stack);
 .__secure_end : AT(ADDR(.__secure_end)) {
  *(.__secure_end)
  LONG(0x1d1071c);
 }
 . = ALIGN(4);
 //只读数据段，首先将所有文件的只读数据段以名字（即 asc 码排序），然后以对齐方式排序放到text段后面
 .rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }
 . = ALIGN(4);
 //所有的数据段
 .data : {
  *(.data*)
 }
 . = ALIGN(4);
 . = .;
 . = ALIGN(4);
 .u_boot_list : {
  KEEP(*(SORT(.u_boot_list*)));
 }
 . = ALIGN(4);
 .efi_runtime_rel_start :
 {
  *(.__efi_runtime_rel_start)
 }
 .efi_runtime_rel : {
  *(.rel*.efi_runtime)
  *(.rel*.efi_runtime.*)
 }
 .efi_runtime_rel_stop :
 {
  *(.__efi_runtime_rel_stop)
 }
 . = ALIGN(4);
 .image_copy_end :
 {
  *(.__image_copy_end)
 }
 .rel_dyn_start :
 {
  *(.__rel_dyn_start)
 }
 .rel.dyn : {
  *(.rel*)
 }
 .rel_dyn_end :
 {
  *(.__rel_dyn_end)
 }
 .end :
 {
  *(.__end)
 }
 _image_binary_end = .;
 . = ALIGN(4096);
 .mmutable : {
  *(.mmutable)
 }
 .bss_start __rel_dyn_start (OVERLAY) : {
  KEEP(*(.__bss_start));
  __bss_base = .;
 }
 .bss __bss_base (OVERLAY) : {
  *(.bss*)
   . = ALIGN(4);
   __bss_limit = .;
 }
 .bss_end __bss_limit (OVERLAY) : {
  KEEP(*(.__bss_end));
 }
 .dynsym _image_binary_end : { *(.dynsym) }
 .dynbss : { *(.dynbss) }
 .dynstr : { *(.dynstr*) }
 .dynamic : { *(.dynamic*) }
 .plt : { *(.plt*) }
 .interp : { *(.interp*) }
 .gnu.hash : { *(.gnu.hash) }
 .gnu : { *(.gnu*) }
 .ARM.exidx : { *(.ARM.exidx*) }
 .gnu.linkonce.armexidx : { *(.gnu.linkonce.armexidx.*) }
}
~~~

### vectors.S

文件位置 [arch/arm/lib/vectors.S]

~~~c
/* SPDX-License-Identifier: GPL-2.0+ */
/*
 *  vectors - Generic ARM exception table code
 *
 *  Copyright (c) 1998 Dan Malek <dmalek@jlc.net>
 *  Copyright (c) 1999 Magnus Damm <kieraypc01.p.y.kie.era.ericsson.se>
 *  Copyright (c) 2000 Wolfgang Denk <wd@denx.de>
 *  Copyright (c) 2001 Alex Züpke <azu@sysgo.de>
 *  Copyright (c) 2001 Marius Gröger <mag@sysgo.de>
 *  Copyright (c) 2002 Alex Züpke <azu@sysgo.de>
 *  Copyright (c) 2002 Gary Jennejohn <garyj@denx.de>
 *  Copyright (c) 2002 Kyle Harris <kharris@nexus-tech.net>
 */

#include <config.h>

/*
 * A macro to allow insertion of an ARM exception vector either
 * for the non-boot0 case or by a boot0-header.
 */
        .macro ARM_VECTORS
#ifdef CONFIG_ARCH_K3
    ldr pc, _reset
#else
    b   reset
#endif
    ldr pc, _undefined_instruction
    ldr pc, _software_interrupt
    ldr pc, _prefetch_abort
    ldr pc, _data_abort
    ldr pc, _not_used
    ldr pc, _irq
    ldr pc, _fiq
    .endm


/*
 *************************************************************************
 *
 * Symbol _start is referenced elsewhere, so make it global
 *
 *************************************************************************
 */

.globl _start

/*
 *************************************************************************
 *
 * Vectors have their own section so linker script can map them easily
 *
 *************************************************************************
 */

    .section ".vectors", "ax"

#if defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK)
/*
 * Various SoCs need something special and SoC-specific up front in
 * order to boot, allow them to set that in their boot0.h file and then
 * use it here.
 *
 * To allow a boot0 hook to insert a 'special' sequence after the vector
 * table (e.g. for the socfpga), the presence of a boot0 hook supresses
 * the below vector table and assumes that the vector table is filled in
 * by the boot0 hook.  The requirements for a boot0 hook thus are:
 *   (1) defines '_start:' as appropriate
 *   (2) inserts the vector table using ARM_VECTORS as appropriate
 */
#include <asm/arch/boot0.h>
#else

/*
 *************************************************************************
 *
 * Exception vectors as described in ARM reference manuals
 *
 * Uses indirect branch to allow reaching handlers anywhere in memory.
 *
 *************************************************************************
 */

_start:
#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
    .word   CONFIG_SYS_DV_NOR_BOOT_CFG
#endif
    ARM_VECTORS
#endif /* !defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK) */

/*
 *************************************************************************
 *
 * Indirect vectors table
 *
 * Symbols referenced here must be defined somewhere else
 *
 *************************************************************************
 */

    .globl  _reset
    .globl  _undefined_instruction
    .globl  _software_interrupt
    .globl  _prefetch_abort
    .globl  _data_abort
    .globl  _not_used
    .globl  _irq
    .globl  _fiq

#ifdef CONFIG_ARCH_K3
_reset: .word reset
#endif
_undefined_instruction: .word undefined_instruction
_software_interrupt:    .word software_interrupt
_prefetch_abort:        .word prefetch_abort
_data_abort:            .word data_abort
_not_used:              .word not_used
_irq:                   .word irq
_fiq:                   .word fiq

    .balignl 16,0xdeadbeef

/*
 *************************************************************************
 *
 * Interrupt handling
 *
 *************************************************************************
 */

/* SPL interrupt handling: just hang */

#ifdef CONFIG_SPL_BUILD

    .align  5
undefined_instruction:
software_interrupt:
prefetch_abort:
data_abort:
not_used:
irq:
fiq:
1:
    b   1b  /* hang and never return */

#else   /* !CONFIG_SPL_BUILD */

/* IRQ stack memory (calculated at run-time) + 8 bytes */
.globl IRQ_STACK_START_IN
IRQ_STACK_START_IN:
#ifdef IRAM_BASE_ADDR
    .word   IRAM_BASE_ADDR + 0x20
#else
    .word   0x0badc0de
#endif

@
@ IRQ stack frame.
@
#define S_FRAME_SIZE    72

#define S_OLD_R0    68
#define S_PSR       64
#define S_PC        60
#define S_LR        56
#define S_SP        52

#define S_IP        48
#define S_FP        44
#define S_R10       40
#define S_R9        36
#define S_R8        32
#define S_R7        28
#define S_R6        24
#define S_R5        20
#define S_R4        16
#define S_R3        12
#define S_R2        8
#define S_R1        4
#define S_R0        0

#define MODE_SVC 0x13
#define I_BIT   0x80

/*
 * use bad_save_user_regs for abort/prefetch/undef/swi ...
 * use irq_save_user_regs / irq_restore_user_regs for IRQ/FIQ handling
 */

    .macro  bad_save_user_regs
    @ carve out a frame on current user stack
    sub sp, sp, #S_FRAME_SIZE
    stmia   sp, {r0 - r12}  @ Save user registers (now in svc mode) r0-r12
    ldr r2, IRQ_STACK_START_IN
    @ get values for "aborted" pc and cpsr (into parm regs)
    ldmia   r2, {r2 - r3}
    add r0, sp, #S_FRAME_SIZE   @ grab pointer to old stack
    add r5, sp, #S_SP
    mov r1, lr
    stmia   r5, {r0 - r3}   @ save sp_SVC, lr_SVC, pc, cpsr
    mov r0, sp  @ save current stack into r0 (param register)
    .endm

    .macro  irq_save_user_regs
    sub sp, sp, #S_FRAME_SIZE
    stmia   sp, {r0 - r12}  @ Calling r0-r12
    @ !!!! R8 NEEDS to be saved !!!! a reserved stack spot would be good.
    add r8, sp, #S_PC
    stmdb   r8, {sp, lr}^   @ Calling SP, LR
    str lr, [r8, #0]    @ Save calling PC
    mrs r6, spsr
    str r6, [r8, #4]    @ Save CPSR
    str r0, [r8, #8]    @ Save OLD_R0
    mov r0, sp
    .endm

    .macro  irq_restore_user_regs
    ldmia   sp, {r0 - lr}^  @ Calling r0 - lr
    mov r0, r0
    ldr lr, [sp, #S_PC] @ Get PC
    add sp, sp, #S_FRAME_SIZE
    subs    pc, lr, #4  @ return & move spsr_svc into cpsr
    .endm

    .macro get_bad_stack
    ldr r13, IRQ_STACK_START_IN @ setup our mode stack

    str lr, [r13]   @ save caller lr in position 0 of saved stack
    mrs lr, spsr    @ get the spsr
    str lr, [r13, #4]   @ save spsr in position 1 of saved stack
    mov r13, #MODE_SVC  @ prepare SVC-Mode
    @ msr   spsr_c, r13
    msr spsr, r13   @ switch modes, make sure moves will execute
    mov lr, pc  @ capture return pc
    movs    pc, lr  @ jump to next instruction & switch modes.
    .endm

    .macro get_irq_stack            @ setup IRQ stack
    ldr sp, IRQ_STACK_START
    .endm

    .macro get_fiq_stack            @ setup FIQ stack
    ldr sp, FIQ_STACK_START
    .endm

/*
 * exception handlers
 */

    .align  5
    undefined_instruction:
    get_bad_stack
    bad_save_user_regs
    bl  do_undefined_instruction

    .align  5
    software_interrupt:
    get_bad_stack
    bad_save_user_regs
    bl  do_software_interrupt

    .align  5
    prefetch_abort:
    get_bad_stack
    bad_save_user_regs
    bl  do_prefetch_abort

    .align  5
    data_abort:
    get_bad_stack
    bad_save_user_regs
    bl  do_data_abort

    .align  5
    not_used:
    get_bad_stack
    bad_save_user_regs
    bl  do_not_used


    .align  5
    irq:
    get_bad_stack
    bad_save_user_regs
    bl  do_irq

    .align  5
    fiq:
    get_bad_stack
    bad_save_user_regs
    bl  do_fiq

#endif  /* CONFIG_SPL_BUILD */
~~~

### start.S

[arch/arm/cpu/armv7/start.S]

~~~c
/* SPDX-License-Identifier: GPL-2.0+ */
/*
 * armboot - Startup Code for OMAP3530/ARM Cortex CPU-core
 *
 * Copyright (c) 2004 Texas Instruments <r-woodruff2@ti.com>
 *
 * Copyright (c) 2001 Marius Gröger <mag@sysgo.de>
 * Copyright (c) 2002 Alex Züpke <azu@sysgo.de>
 * Copyright (c) 2002 Gary Jennejohn <garyj@denx.de>
 * Copyright (c) 2003 Richard Woodruff <r-woodruff2@ti.com>
 * Copyright (c) 2003 Kshitij <kshitij@ti.com>
 * Copyright (c) 2006-2008 Syed Mohammed Khasim <x0khasim@ti.com>
 */

#include <asm-offsets.h>
#include <config.h>
#include <asm/system.h>
#include <linux/linkage.h>
#include <asm/armv7.h>

/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * Do important init only if we don't start from memory!
 * Setup memory and board specific bits prior to relocation.
 * Relocate armboot to ram. Setup stack.
 *
 *************************************************************************/

    .globl  reset
    .globl  save_boot_params_ret
    .type   save_boot_params_ret,%function
#ifdef CONFIG_ARMV7_LPAE
    .global switch_to_hypervisor_ret
#endif

reset:
    /* Allow the board to save important registers */
    b   save_boot_params
save_boot_params_ret:
#ifdef CONFIG_ARMV7_LPAE
/*
 * check for Hypervisor support
 */
    mrc p15, 0, r0, c0, c1, 1       @ read ID_PFR1
    and r0, r0, #CPUID_ARM_VIRT_MASK    @ mask virtualization bits
    cmp r0, #(1 << CPUID_ARM_VIRT_SHIFT)
    beq switch_to_hypervisor
switch_to_hypervisor_ret:
#endif
    /*
        * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
        * except if in HYP mode already
        */
    mrs r0, cpsr
    and r1, r0, #0x1f       @ mask mode bits
    teq r1, #0x1a       @ test for HYP mode
    bicne   r0, r0, #0x1f       @ clear all mode bits
    orrne   r0, r0, #0x13       @ set SVC mode
    orr r0, r0, #0xc0       @ disable FIQ and IRQ
    msr cpsr,r0

/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
    /* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
    mrc p15, 0, r0, c1, c0, 0   @ Read CP15 SCTLR Register
    bic r0, #CR_V       @ V = 0
    mcr p15, 0, r0, c1, c0, 0   @ Write CP15 SCTLR Register

#ifdef CONFIG_HAS_VBAR
    /* Set vector address in CP15 VBAR register */
    ldr r0, =_start
    mcr p15, 0, r0, c12, c0, 0  @Set VBAR
#endif
#endif

    /* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
#ifdef CONFIG_CPU_V7A
    bl cpu_init_cp15
#endif
#ifndef CONFIG_SKIP_LOWLEVEL_INIT_ONLY
    bl cpu_init_crit
#endif
#endif

    bl  main

/*------------------------------------------------------------------------------*/

ENTRY(c_runtime_cpu_setup)
/*
 * If I-cache is enabled invalidate it
 */
#if !CONFIG_IS_ENABLED(SYS_ICACHE_OFF)
    mcr p15, 0, r0, c7, c5, 0   @ invalidate icache
    mcr     p15, 0, r0, c7, c10, 4  @ DSB
    mcr     p15, 0, r0, c7, c5, 4   @ ISB
#endif

    bx  lr

ENDPROC(c_runtime_cpu_setup)

/*************************************************************************
 *
 * void save_boot_params(u32 r0, u32 r1, u32 r2, u32 r3)
 *  __attribute__((weak));
 *
 * Stack pointer is not yet initialized at this moment
 * Don't save anything to stack even if compiled with -O0
 *
 *************************************************************************/
ENTRY(save_boot_params)
    b   save_boot_params_ret        @ back to my caller
ENDPROC(save_boot_params)
    .weak   save_boot_params

#ifdef CONFIG_ARMV7_LPAE
ENTRY(switch_to_hypervisor)
    b   switch_to_hypervisor_ret
ENDPROC(switch_to_hypervisor)
    .weak   switch_to_hypervisor
#endif

/*************************************************************************
 *
 * cpu_init_cp15
 *
 * Setup CP15 registers (cache, MMU, TLBs). The I-cache is turned on unless
 * CONFIG_SYS_ICACHE_OFF is defined.
 *
 *************************************************************************/
ENTRY(cpu_init_cp15)
    /*
        * Invalidate L1 I/D
        */
    mov r0, #0      @ set up for MCR
    mcr p15, 0, r0, c8, c7, 0   @ invalidate TLBs
    mcr p15, 0, r0, c7, c5, 0   @ invalidate icache
    mcr p15, 0, r0, c7, c5, 6   @ invalidate BP array
    mcr     p15, 0, r0, c7, c10, 4  @ DSB
    mcr     p15, 0, r0, c7, c5, 4   @ ISB

    /*
        * disable MMU stuff and caches
        */
    mrc p15, 0, r0, c1, c0, 0
    bic r0, r0, #0x00002000	@ clear bits 13 (--V-)
    bic r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)
    orr r0, r0, #0x00000002	@ set bit 1 (--A-) Align
    orr r0, r0, #0x00000800	@ set bit 11 (Z---) BTB
#if CONFIG_IS_ENABLED(SYS_ICACHE_OFF)
    bic r0, r0, #0x00001000 @ clear bit 12 (I) I-cache
#else
    orr r0, r0, #0x00001000 @ set bit 12 (I) I-cache
#endif
    mcr p15, 0, r0, c1, c0, 0

#ifdef CONFIG_ARM_ERRATA_716044
    mrc p15, 0, r0, c1, c0, 0   @ read system control register
    orr r0, r0, #1 << 11    @ set bit #11
    mcr p15, 0, r0, c1, c0, 0   @ write system control register
#endif

#if (defined(CONFIG_ARM_ERRATA_742230) || defined(CONFIG_ARM_ERRATA_794072))
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 4     @ set bit #4
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_743622
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 6     @ set bit #6
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_751472
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 11    @ set bit #11
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
#ifdef CONFIG_ARM_ERRATA_761320
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 21    @ set bit #21
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_845369
    mrc     p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr     r0, r0, #1 << 22    @ set bit #22
    mcr     p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

    mov r5, lr  @ Store my Caller
    mrc p15, 0, r1, c0, c0, 0   @ r1 has Read Main ID Register (MIDR)
    mov r3, r1, lsr #20     @ get variant field
    and r3, r3, #0xf        @ r3 has CPU variant
    and r4, r1, #0xf        @ r4 has CPU revision
    mov r2, r3, lsl #4      @ shift variant field for combined value
    orr r2, r4, r2      @ r2 has combined CPU variant + revision

/* Early stack for ERRATA that needs into call C code */
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
    ldr r0, =(CONFIG_SPL_STACK)
#else
    ldr r0, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
    bic r0, r0, #7  /* 8-byte alignment for ABI compliance */
    mov sp, r0

#ifdef CONFIG_ARM_ERRATA_798870
    cmp r2, #0x30       @ Applies to lower than R3p0
    bge skip_errata_798870      @ skip if not affected rev
    cmp r2, #0x20       @ Applies to including and above R2p0
    blt skip_errata_798870      @ skip if not affected rev

    mrc p15, 1, r0, c15, c0, 0  @ read l2 aux ctrl reg
    orr r0, r0, #1 << 7         @ Enable hazard-detect timeout
    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_l2aux_ctrl
    isb             @ Recommended ISB after l2actlr update
    pop {r1-r5}         @ Restore the cpu info - fall through
skip_errata_798870:
#endif

#ifdef CONFIG_ARM_ERRATA_801819
    cmp r2, #0x24       @ Applies to lt including R2p4
    bgt skip_errata_801819      @ skip if not affected rev
    cmp r2, #0x20       @ Applies to including and above R2p0
    blt skip_errata_801819      @ skip if not affected rev
    mrc p15, 0, r0, c0, c0, 6   @ pick up REVIDR reg
    and r0, r0, #1 << 3     @ check REVIDR[3]
    cmp r0, #1 << 3
    beq skip_errata_801819	@ skip erratum if REVIDR[3] is set

    mrc p15, 0, r0, c1, c0, 1   @ read auxilary control register
    orr r0, r0, #3 << 27    @ Disables streaming. All write-allocate
                    @ lines allocate in the L1 or L2 cache.
    orr r0, r0, #3 << 25    @ Disables streaming. All write-allocate
                    @ lines allocate in the L1 cache.
    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}         @ Restore the cpu info - fall through
skip_errata_801819:
#endif

#ifdef CONFIG_ARM_CORTEX_A15_CVE_2017_5715
    mrc p15, 0, r0, c1, c0, 1   @ read auxilary control register
    orr r0, r0, #1 << 0     @ Enable invalidates of BTB
    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}         @ Restore the cpu info - fall through
#endif

#ifdef CONFIG_ARM_ERRATA_454179
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR

    cmp r2, #0x21       @ Only on < r2p1
    orrlt   r0, r0, #(0x3 << 6) @ Set DBSM(BIT7) and IBE(BIT6) bits

    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}         @ Restore the cpu info - fall through
#endif

#if defined(CONFIG_ARM_ERRATA_430973) || defined (CONFIG_ARM_CORTEX_A8_CVE_2017_5715)
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR

#ifdef CONFIG_ARM_CORTEX_A8_CVE_2017_5715
    orr r0, r0, #(0x1 << 6) @ Set IBE bit always to enable OS WA
#else
    cmp r2, #0x21       @ Only on < r2p1
    orrlt   r0, r0, #(0x1 << 6) @ Set IBE bit
#endif
    push    {r1-r5}     @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}         @ Restore the cpu info - fall through
#endif

#ifdef CONFIG_ARM_ERRATA_621766
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR

    cmp r2, #0x21       @ Only on < r2p1
    orrlt   r0, r0, #(0x1 << 5) @ Set L1NEON bit

    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}         @ Restore the cpu info - fall through
#endif

#ifdef CONFIG_ARM_ERRATA_725233
    mrc p15, 1, r0, c9, c0, 2   @ Read L2ACR

    cmp r2, #0x21       @ Only on < r2p1 (Cortex A8)
    orrlt   r0, r0, #(0x1 << 27)    @ L2 PLD data forwarding disable

    push    {r1-r5}         @ Save the cpu info registers
    bl  v7_arch_cp15_set_l2aux_ctrl
    pop {r1-r5}         @ Restore the cpu info - fall through
#endif

#ifdef CONFIG_ARM_ERRATA_852421
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 24    @ set bit #24
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_852423
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 12    @ set bit #12
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

    mov	pc, r5          @ back to my caller
ENDPROC(cpu_init_cp15)

#if !defined(CONFIG_SKIP_LOWLEVEL_INIT) && \
    !defined(CONFIG_SKIP_LOWLEVEL_INIT_ONLY)
/*************************************************************************
 *
 * CPU_init_critical registers
 *
 * setup important registers
 * setup memory timing
 *
 *************************************************************************/
ENTRY(cpu_init_crit)
    /*
        * Jump to board specific initialization...
        * The Mask ROM will have already initialized
        * basic memory. Go here to bump up clock rate and handle
        * wake up conditions.
        */
    b   lowlevel_init       @ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
#endif
~~~
