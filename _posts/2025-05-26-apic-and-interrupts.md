---
layout: post
title: Modern kernel interrupts with the x2apic
date: 2025-05-26 9:30:00 -0800
categories: locos programming featured
---

I outlined the topics I was about to make in my last post. Although it's a bit late (I was planning to write this one two weeks ago) and finals are looming over the horizon, here's the long awaited second post.

### Hardware interrupts

Say you have a operating system kernel. It can manage memory, write stuff to the screen, and handle cpu exceptions. Now what?

The next logical step would be to find a way to get input from the outside world - maybe the keyboard, the mouse, or the hardware timers.

Or maybe you'd like to find a way to multitask. A kernel really isn't much if it can only run itself.

Fortunately, you won't need to decide between these two, because they both require the same exact thing. A way to get hardware interrupts, or those that aren't directly generated by the cpu.

### The old PIC system

Hardware interrupts must all pass through some sort of interrupt controller that manages them and decides when to send them to your kernel. 

In the olden days, this was done through the [**PIC**](https://en.wikipedia.org/wiki/Programmable_interrupt_controller) (**P**rogrammable **I**nterrupt **C**ontroller). In fact, most modern systems start in the legacy PIC mode and are fully compatible with this protocol. 

The PIC was two components that each mapped interrupts from hardware and channeled it to the cpu. The drawback of this, however, was that because each one only had 8 lines, there was a total of 15 (as one on the second, or the slave PIC, in the cascade was connected to the first one).

The PIC also doesn't support multi-core systems and was slower then the modern interrupt controller because you accessed ports to signal EOI (end of interrupt). Plus, it overlaps with the interrupt vectors for CPU exceptions by default.

This is why the first thing to do is to remap and disable the legacy PIC:

```rs
pub fn disable_legacy_pics() {
    init_and_remap_pics();
    mask_all_irqs();
}

fn init_and_remap_pics() {
    unsafe {
        let mut master_port = Port::new(PIC1_COMMAND);
        master_port.write(0x11u8); // ICW 1 starts init sequence
        let mut master_data_port = Port::new(PIC1_DATA);
        master_data_port.write(PIC1_OFFSET); // Remap offset to 32
        master_data_port.write(0x04); // Tell PIC1 that there is slave PIC
        master_data_port.write(0x01); // set operational modes

        let mut slave_port = Port::new(PIC2_COMMAND);
        slave_port.write(0x11u8);
        let mut slave_data_port = Port::new(PIC2_DATA);
        slave_data_port.write(PIC2_OFFSET); // Remap offset to 40
        slave_data_port.write(0x02); // Tell PIC2 it's the slave
        slave_data_port.write(0x01);
    }
}

fn mask_all_irqs() {
    unsafe {
        let mut master_port = Port::new(PIC1_DATA);
        master_port.write(ALL_INTERRUPTS_MASK);
        let mut slave_port = Port::new(PIC2_DATA);
        slave_port.write(ALL_INTERRUPTS_MASK);
    }
}
```

### The new APIC

After this, the next step is to setup the [**APIC**](https://wiki.osdev.org/APIC) (**A**dvanced **P**rogrammable **I**nterrupt **C**ontroller). The system is comprised of two separate structures - the (I/O apic)[https://wiki.osdev.org/IOAPIC], which routes external interrupts to cores, and the local apic (lapic) which handles them, alongside having a timer in itself.

There's also been multiple APIC standards, the most recent two being xapic (released 2000) and x2apic (released 2008) - the main differences are that x2apic bumps the lapic ID to 32 bits to support more cores, and also makes all the apic registers [MSR](https://wiki.osdev.org/Model_Specific_Registers)s (model specific registers, accessible through the `rdmsr` and `wrmsr` instructions) instead of [MMIO](https://en.wikipedia.org/wiki/Memory-mapped_I/O_and_port-mapped_I/O).

After fiddling around with setting the lapic up manually, I luckily found the `x2apic` crate that abstracts away a lot of the setup involved with it. First, we setup the lapic's interrupt vectors
- timer (a programmable local timer)
- error (when the lapic malfunctions itself)
- spurious (used when the lapic receives a request, but it's handeled by another core before it can pass it on)

```rs
#[allow(static_mut_refs)]
pub unsafe fn setup_apic(rsdp_addr: usize) {
    disable_legacy_pics();

    let mut builder = LocalApicBuilder::new();
    let mut lapic = builder
        .timer_vector(LAPIC_TIMER_VECTOR as usize)
        .error_vector(LAPIC_ERROR_VECTOR as usize)
        .spurious_vector(LAPIC_SPURIOUS_VECTOR as usize);

    // check to see if it's x2 or xapic, error if its not x2apic

        let mut final_lapic = lapic.build().unwrap();

    // set interrupt handlers for each interrupt
    unsafe {
        (*IDT.as_mut_ptr())[LAPIC_TIMER_VECTOR].set_handler_fn(lapic_timer_handler);
        (*IDT.as_mut_ptr())[LAPIC_ERROR_VECTOR].set_handler_fn(lapic_error_handler);
        (*IDT.as_mut_ptr())[LAPIC_SPURIOUS_VECTOR].set_handler_fn(spurious_handler);
    }

    unsafe { final_lapic.enable() }; // turn on lapic timer
}
```

You might've noticed that I'm using a global static mut, which is heavily frowned upon. If somebody finds a better way to do it, I'll be happy to implement it. A mutx won't work as loading the IDT needs it to be `'static`, and changing it needs a mutable reference.

In each of the interrupt handlers, we need to make sure to aknowledge the interrupt by sending an arbitrary value to the EOI MSR.

```rs
extern "x86-interrupt" fn lapic_timer_handler(_stack_frame: InterruptStackFrame) {
    // ... task switch, do stuff
    unsafe {
        Msr::new(X2APIC_EOI_MSR).write(0);
    };
}
```

### The IO APIC

A bulk of the 400+ line apic handling is taken up by the setup for the IO apic, which I won't be diving too deep into. 

An outline of what happens - we first find the location of the IO APIC's MMIO (or multiple IO APIC's) by parsing ACPI tables, provided by the bootloader. Even with a crate to help with this, it still takes a while.

Then, we map the MMIOs to virtual memory in order to access it, we check to see what interrupts each IO APIC is handling, then map them correctly.

It was only halfway through setting up the IO APIC that I realized that it is slowly becoming less and less useful on modern systems, which allow peripheral devices (usb bus, nvme) to skip the IO APIC entirely and message lapics through [MSI/MSI-x](https://en.wikipedia.org/wiki/Message_Signaled_Interrupts)s.

Because of sunk cost, I decided to finish it anyways and setup the [`PIT`](https://wiki.osdev.org/Programmable_Interval_Timer). You can see my code [here](https://github.com/Makonede/locos/blob/781aeb962fb1fd93266ef50e7270f5bd9c0bc892/kernel/src/interrupts/apic.rs) if you want.

### Enabling interrupts

In finale, interrupts need to be enabled.

```rs
#[unsafe(no_mangle)]
unsafe extern "C" fn kernel_main() -> ! {
    // ...

    unsafe { setup_apic(rsdp_addr) }; // address of root system description pointer, points to madt table which points to io apics
    x86_64::instructions::interrupts::enable();

    // ...
}
```

### Next steps

Using the lapic timer, preemptive task switching can finally be implemented, where the kernel gives amounts of time to each task, regulated by timer interrupts.

The next post will most likely be on refactoring my entire memory allocation scheme to make it more efficient and durable, drawing inspiration from the way linux does it. After that, I'll setup multitasking using some simple scheduling algorithm.

The steps after that might be to setup the USB bus to get keyboard input, write a filesystem driver, or enter user-land.

Thanks for the read.