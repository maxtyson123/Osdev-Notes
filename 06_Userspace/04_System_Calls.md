# System Calls
System calls are a way for a user mode program to request something from the kernel, or other supervisor code. For this chapter we're going to focus on a user program calling the kernel directly. If the kernel being written is a micro-kernel, system calls can be more complicated, as they might be redirected to other supervisor (or even user) programs, but we're not going to talk about that here.

On `x86_64` there are a few ways to perform a system call. The first is to dedicate an interrupt vector to be used for software interrupts. This is the most common, and straightforward way. The other main way is to use the dedicated instructions (`sysenter` and friends), however these are rather niche and have some issues of their own. This is discussed below.

There are other obscure ways to perform syscalls. For example, executing a bad instruction will cause the cpu to trigger a #UD exception. This transfers control to supervisor code, and could be used as an entry to a system call. While not recommended for beginners, there was one hobby OS kernel that used this method.

## The System Call ABI
A stable ABI is always a good thing, but especially in the case of system calls. If we start to write user code that uses your system calls, and then the ABI changes later, all of the previous code will break. Therefore it's recommended to take some time to design the core of the ABI before implementing it.

Some common things to consider include:

- How does the user communicate which system call they want?
- How does the user pass arguments to the kernel?
- How does the kernel return data to the user?
- How to pass larger amounts of data?

On `x86_64`, using registers to pass arguments/return values is the most straightforward way. Which registers specifically are left as an exercise to the reader. Note there's no right or wrong answer here (except maybe `rsp`), it's a matter of preference.

Since designing an ABI can be a rather tricky thing to get *just right* the first time, an example one is discussed in the next chapter. Along with the methodology used to create it.

## Using Interrupts

We've probably heard of `int 0x80` before. That's the interrupt vector used by Linux for system calls. It's a common choice as it's easy to identify in logs, however any (free) vector number can be used. Some people like to place the system call vector up high (0xFE, just below the LAPIC spurious vector), others place it down low (0x24).

*What about using multiple vectors for different syscalls?* Well this is certainly possible, but it's more points of entry into supervisor code. System calls are also not the only thing that require interrupt vectors (for example, a single PCI device can request upto 32!), and with enough hardware you may find yourself running out of interrupt vectors to allocate. So only using a single vector is recommended.

### Using Software Interrupts

Now we've selected which interrupt vector to use for system calls, we can install an interrupt handler. On `x86_64`, this is done via the IDT like any other interrupt handler. The only difference is the DPL field.

As mentioned before, the DPL field is the highest ring that is allowed to call this interrupt from software. By default it was left as 0, meaning only ring 0 can trigger software interrupts. Other rings trying to do this will trigger a general protection fault. However since we want ring 3 code to call this vector, we'll need to set its DPL to 3.

Now we have an interrupt that can be called from software in user mode, and a handler that will be called on the supervisor side.

### A Quick Example

We're going to use vector `0xFE` as our system call handler, and assume that the interrupt stub pushes all registers onto the stack before executing the handler (we're taking this as the `cpu_status_t*` argument).
We'll also assume that `rdi` is used to pass the system call number, and `rsi` passes the argument to that syscall. These are things that should be decided when writing the ABI for your kernel.

First we'll set up things on the kernel side:

```c
//left to the user to implement
void set_idt_entry(uint8_t vector, void* handler, uint8_t dpl);

//see below
cpu_status_t* syscall_handler(cpu_status_t* regs);

void setup_syscalls()
{
    set_idt_entry(0xFE, syscall_handler, 3);
}
```

Now on the user side, we can use `int $0xFE` to trigger a software interrupt. If we try to trigger any other interrupts, we'll still get a protection fault.

```c
__attribute__((naked))
size_t execute_syscall(size_t syscall_num, size_t arg)
{
    asm("int $0xFE"
        : "=a"(arg)
        : "D"(syscall_num), "S"(arg));

    return arg;
}
```

There's a few tricks happening with the inline assembly above. First is the `naked` attribute. This is not strictly necessary, but since we're only doing inline assembly in the function it's a nice optimization hint to the compiler. It tells the compiler not to generate the prologue/epilogue sequences for this function. This is stuff like creating the stack frame.

Next we're using some special constraints for the input and output operands. "a" means to use the `rax` register, while "S" and "D" are the source and destination registers (`rsi` and `rdi`) registers. This means the compiler will ensure that they are loaded with the values we specify before the assembly body is run. The compiler will then also put the value of "a" (`rax`) into `arg` after the assembly body has run. This is where we'll be placing the return value of the system call, hence why the `return arg` line below.

For more details on inline assembly, see the dedicated appendix on it, or check the compiler's manual.

Now assuming everything is setup correctly, running the above code in user mode should trigger the kernel's system call handler. In the example below, the `syscall_handler` function should end up running, and we've just implemented system calls!

```c
cpu_status_t* syscall_handler(cpu_status_t* regs)
{
    log("Got syscall %lx, with argument %lx", regs->rdi, regs->rsi);

    //remember rdi is our syscall number
    switch (regs->rdi)
    {
        //syscall 2 only wants the argument
        case 2:
            do_syscall_2(regs->rsi);
            break;

        //syscall 3 wants the full register state
        case 3:
            do_syscall_3(regs);
            break;

        //no syscall with that id, return an error
        default:
            regs->rsi = E_NO_SYSCALL;
            break;
    }

    return regs;
}
```

## Using Dedicated Instructions

On `x86_64` there exists a pair of instructions that allow for a "fast supervisor entry/exit". The reason these instructions are considered fast is they bypass the whole interrupt procedure. Instead, they are essentially a pair of far-jump/far-return instructions, with the far-jump to kernel code using a fixed entry point.

This is certainly faster as the instruction only needs to deal with a handful of registers, however it leaves the rest of the context switching up to the kernel code.

Upon entering the kernel, you will be running with ring 0 privileges and certain flags will be cleared, and that's it. You must perform the stack switch yourself, as well as collecting any information the kernel might need (like the user RIP, stack, SS/CS).

*Authors Note: While these instructions are covered here, `syscall` can actually result in several quite nasty security bugs if not used carefully. These issues can be worked around of course, but at that point you've lost the speed benefit offered by using these instead of an interrupt. We consider using these instructions an advanced topic. If you do find yourself in a position where the speed of the system call entry is a bottleneck, then these instructions are likely not the solution, and you should look at why you require so many system calls. - DT*

### Compatibility Issues

As we hinted at before, there are actually two pairs of instructions: a pair designed by Intel and a pair by AMD. Unfortunately neither could agree on which to use, so we're left in an awkward situation. On `x86` (32-bit) Intel created their instructions first, and AMD honoured this by supporting them. These instructions are `sysenter` and `sysexit`, making use of three MSRs. If interested in these, all the relevant details can be found in the intel manuals.

Since AMD designed the 64-bit version of `x86` (`x86_64`), they made their instructions architectural and deprecated Intel's. For 64-bit platforms, we have the `syscall` and `sysret` instructions. Functionally very similar to the other pair, they do have slight differences. Since we're focused on `x86_64`, we'll only discuss the 64-bit versions.

In summary: if the kernel is on a 32-bit platform, use `sysenter`/`sysexit`, for 64-bit use `syscall`/`sysret`.

### Using Syscall & Sysret

Before using these instructions we'll need to perform a bit of setup. 

First things first, the `syscall`/`sysret` pair uses three MSRs: `STAR` (0xC0000081), `LSTAR` (0xC0000082) and `FMASK` (0xC0000084).

`STAR` has three fields that specify the selectors used for userspace and kernelspace. The low 32 bits are reserved for the handler of 32-bit variant of `syscall`. It's safe to set it to zero if you don't target 32-bit userspace code. The next 16 bits (47:32) hold the kernel CS/SS base. When `syscall` is executed, the new CS is set to `base` and the new SS is set to `base + 8`. The last 16 bits (64:48) hold the userspace CS/SS base. Because it supports jumping back to compatibility mode, `sysret` requires a very specific GDT layout. It expects the 32-bit CS to be at `base`, the SS (either 64-bit or 32-bit) at `base + 8` and the 64-bit CS to be at `base + 16`. If you don't target 32-bit code, it's safe to omit the 32-bit segment from the GDT, but note that the 64-bit CS will still be fetched from `base + 16`.

`LSTAR` holds the RIP of the handler.

`FMASK` holds a bit mask used to disable certain flags when transitioning control to kernel. Basically, if a bit in `FMASK` is set, the corresponding flag is guaranteed to be cleared when entering the kernel.

An example of properly configured GDT and MSRs for 64-bit-only operation:

* GDT:
|Offset|Name|Access byte|
|------|----|-----------|
|0x0|NULL|`0`|
|0x8|Kernel Code|`0b10011010`|
|0x10|Kernel Data|`0b10010010`|
|0x18|User Data|`0b11110010`|
|0x20|User Code|`0b11111010`|

* MSRs:
|MSR|Name|Value|
|---|----|-----|
|`0xC0000081`|`STAR`| `0x0013000800000000` |
|`0xC0000082`|`LSTAR`| address of the handler |
|`0xC0000084`|`FMASK`| `0xFFFFFFFFFFFFFFFD` | 

In this configuration, `STAR`(47:32) is set to `0x8`. It corresponds to the kernel code segment. Thus, on a `syscall` the CS will be `0x8` and SS `0x10`. Next, bits (64:48) are set to `0x13`. After `sysret`, the CS will be `0x23` (`0x13 + 16`) and SS will be `0x1B` (`0x13 + 8`). Note that we have to set the lower two bits that represent the privilege level! And last, `FMASK` clears all flags while keeping bit 1 (reserved) set.

As an aside, the `sysret` instruction determines which mode to return to based on the operand size. By default all operands are 32-bit, to specify a 64-bit operand (i.e. return to 64-bit long mode) just add the `q` suffix in GNU as, or the `o64` prefix in NASM.

```x86asm
//GNU as
sysretq

//NASM
o64 sysret
```

Finally, we need to tell the CPU we support these instructions and have done all of the above setup. Like many extended features on `x86`, there is a flag to enable them at a global level. For `syscall`/`sysret` this is the system call extensions flag in the `EFER` (0xC0000080) MSR, which is bit 0. After setting this, the CPU is ready to handle these instructions!

### Handler Function

We're not done with these instructions yet! We can get to and from our handler function, but there are a few critical things to know when writing a handler function:

- The stack *selector* has been changed, but the stack itself has not. We're still operating on the user's stack. The stack need to be loaded manually.
- Since the flags register is modified by `FMASK`, the previous flags are stored in the `r11` register. The value of `r11` is also used to restore the flags register when `sysret` is executed.
- The saved user instruction pointer is available in rcx.
- Although interrupts are disabled, machine check exceptions and non maskable interrupts can (and will) still occur. Since the  handler will already have ring 0 selectors loaded, the CPU won't automatically switch stacks if an interrupt occurs during its execution. This is pretty dangerous if one of these interrupts happens before the kernel stack is loaded, as that means we'll be running supervisor code on a user stack, and may not be aware of it. The solution is to use the interrupt stack tables for these interrupts, so they will always have a new stack loaded.
