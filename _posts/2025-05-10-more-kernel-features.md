---
layout: post
title: More kernel features - Migration to Limine
date: 2025-05-10 18:45:00 -0800
categories: locos programming
---

After another month of mostly weekends spent working on my kernel, we have a few major changes and additions.
- Migration to the limine bootloader
- Migration to flanterm console
- Apic and clock handling
- Frame allocator improvements

This post will go over the first migration to limine.

### What is limine?

Limine is a bootloader (the thing that loads up an OS) that supports a variety of features and boot protocols, including multiboot (for grub), linux's own boot protocol, and its own limine boot protocol.

Since my kernel depended on the bootloader crate before, I didn't have an existing multiboot header and therefore decided to use the limine boot protocol. 

### Why limine?

It's fast, full-featured, and stable (compared to the `bootloader` crate, at least). It seemed easier to setup then grub and a multiboot header, and is in active development, meaning more modern cool stuff in it.

### Migrating the build

The structure when using the `bootloader` crate was to have two crates, one for the image and one for the kernel code itself:
```
locos/
├── Makefile
├── bootimage/
│   ├── .cargo/
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── build.rs
│   └── src/
├── kernel/
│   ├── .cargo/
│   ├── Cargo.lock
│   ├── Cargo.toml
│   └── src/
└── rust-toolchain.toml
```

This didn't fit well with limine, however, as it itself isn't two separate rust crates. Moving most of the build, linking, and iso creation logic to `build.rs` and makefiles, I ended up with this directory structure:

```
locos/
├── Makefile
├── kernel/
│   ├── .cargo/
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── Makefile
│   ├── build.rs
│   ├── linker.ld
│   └── src/
├── limine.conf
└── rust-toolchain.toml
```

`build.rs` exists to make sure rustc uses the linker script linker.ld for the final `elf` binary. The linker script makes sure that the binary meets the requirements of the limine boot protocol, that the entry point is `kernel_main` and that the sections in which kernel code places requests (ie. bigger stack size, higher-half direct mapping) to limine is placed in `.data`.

```ld
/* We want the symbol kernel_main to be our entry point */
ENTRY(kernel_main)

...

SECTIONS {

    ...

    /* We want to be placed in the topmost 2GiB of the address space, for optimisations */
    /* and because that is what the Limine spec mandates. */
    /* Any address in this region will do, but often 0xffffffff80000000 is chosen as */
    /* that is the beginning of the region. */
    . = 0xffffffff80000000;

    ...

    .data : {
        *(.data .data.*)

        /* Place the sections that contain the Limine requests as part of the .data */
        /* output section. */
        KEEP(*(.requests_start_marker))
        KEEP(*(.requests))
        KEEP(*(.requests_end_marker))
    } :data
    
    ...
}
```

A lot of the linker script is taken from [the rust example for limine](https://github.com/jasondyoungberg/limine-rust-template/blob/trunk/kernel/linker-x86_64.ld) if you want to take a look.

After the linker script is done, the crate-level `Makefile` builds the kernel with `-Crelocation-model=static`, which disables PIC, or position-independant code, where all memory addresses are stored as offsets instead of exact addresses. Normally, a program uses this so that a dynamic linker from the OS can load it and make sure that all the memory offsets are substituted with real memory addresses. For our kernel, this isn't present so we need to make it use exact addresses where the kernel is placed in virtual memory - starting at `0xffffffff80000000`, stated in the linker script.

```makefile
all:
	RUSTFLAGS="-C relocation-model=static" cargo build --target $(RUST_TARGET) --profile $(RUST_PROFILE)
	mkdir -p $(BUILD_DIR) && cp target/$(RUST_TARGET)/$(RUST_PROFILE_SUBDIR)/kernel $(BUILD_DIR)/$(OUTPUT)
```

After this, the [root-level `Makefile`](https://github.com/Makonede/locos/blob/cdceaa90e001f5a0fe5df2193b7e0d87a6404432/Makefile) bundles everything together (limine uefi, bios images) and creates a single `locos.iso` containing the kernel. It also has some utilities for running the created image in qemu.

### Migrating the code

The main difference from limine to the `bootloader` crate is that you need to manually request different features of limine in a special `.requests` section in your binary. For example, here in `main.rs` is a request for the memory map, which is later needed to know which physical frames are availible and map them to kernel-space processes.

```rs
#[used]
#[unsafe(link_section = ".requests")]
static MEMORY_MAP_REQUEST: MemoryMapRequest = MemoryMapRequest::new();
```

You can see in `kernel_main` I later process this request into the needed memory regions and initialize my frame allocator from it.

```rs
let memory_regions = MEMORY_MAP_REQUEST
    .get_response()
    .expect("memory map request failed")
    .entries();
let mut frame_allocator = unsafe { BootInfoFrameAllocator::init(memory_regions) };
```

The bootloader functionality that I depended on from the `bootloader` crate was present in limine, so i only needed to change a few types and how I got the information from the bootloader. Overall, it was a moderately simple migration.

I later also changed the CI pipeline so that everything built on push.

If you're curious and want to see the code, the migration starts at [this commit](https://github.com/Makonede/locos/commit/67300a5ac43aa9f5099cfc4e9e41928ccafdc931) and ends at [this one](https://github.com/Makonede/locos/commit/87b4f67be7dc2348d798a580fdde70f4a6300204). Since then, I've changed quite a few things but I'll go over those in later posts.