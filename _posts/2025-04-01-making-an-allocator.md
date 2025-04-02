---
layout: post
title: Making an allocator for my kernel
date: 2025-04-01 19:45:00 -0800
categories: locos programming
---

The long awaited (at least by me) post... has finally come. ~Now, the obvious reason why this is on april fools day is because my kernel is a joke.~

It has taken this long because I have given up on waiting for cargo maintainers to fix the previous [issue](https://github.com/rust-lang/cargo/issues/10444) and have decided to just migrate my project into two separate crates.

This post will go over the inspiration behind an allocator, and the steps taken to implement it.

The actual implementation is [here](https://github.com/Makonede/locos/blob/main/kernel/src/memory/alloc.rs), in github.

### Inspiration

I'm making my own operating system, which hopefully will become something actually usable in the future. A lot of the foundational pieces were given by Phil Opperman's [blog posts](https://os.phil-opp.com/) on the topic, albeit my version has changes including

- structured into two crates, the kernel and the bootimage
- Using a framebuffer console instead of a heap (next post will be on improving it)
- **Not using a simple bump, block, or linked-list allocator**
- (in the future) not using the legacy interrupt controller

While reading through his posts, I found that he gave a few different implementations of memory allocators for the kernel heap, yet all of them had quite distinct drawbacks, most notably fragmentation. In the end, I decided on using a simple buddy allocator system, which would give decent performance and less memory fragmentation.

### What is a buddy allocator?

A memory allocator is simple. You give it a block of memory (think of a long tape measure), and let it manage it when you later need memory of a specific size (think the width) and alignment (think "the start of this can only be placed at marks of a multiple of *X*"). It also handles freeing the memory ("hey allocator, I'm not gonna use this anymore, you can give it to some other caller") when you don't need it anymore.

The interface for this in rust used by the `alloc` crate, which provides all the heap-allocated types including `Vec<T>` and `Box<T>`, is `[GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html)`. This means that a memory allocator must implement `alloc`, which returns a pointer to the memory they request, and `dealloc`, which frees it.

There are many different ways to handle memory allocation. The simplest way is a [bump allocator](https://rust-hosted-langs.github.io/book/chapter-simple-bump.html), which gives out blocks of memory, one after the other. The drawback is that you can only allocate memory after the last block - also meaning that you can only truely free the entire block at once, leading to *memory fragmentation* as more and more space is wasted as blocks inside are unused.

There's also [linked (free) list allocation](https://en.wikipedia.org/wiki/Free_list), which stores pointers in the free regions of memory pointing to the next free region, making a linked list. The problem with this approach, while it saves memory, is that allocation and deallocation can be both O(n), and there can be a lot of memory fragmentation.

An upgraded version of this would be a block allocator, or [memory pool](https://en.wikipedia.org/wiki/Memory_pool), where you have predetermined blocks of memory with fixed sizes. this reduces memory fragmentation a bit and makes it so that, if you have multiple linked lists for each size, you can get allocations in constant time.

Now, we get the [buddy allocator](https://en.wikipedia.org/wiki/Buddy_memory_allocation). A buddy allocation system repeatedly splits blocks of memory in half until the size of the needed memory's next power of two is reached. In doing these splits, each split block of memory is called a buddy. During deallocation, if the deallocated memory block's buddy is also free, they become merged together into one block recursively. By doing this, it decreases fragmentation by a ton.

### Implementation

First, I found a [reference implementation](https://nfil.dev/kernel/rust/coding/rust-buddy-allocator/). Huge thanks to Nikos' blog.

Because I wanted to just keep the bump allocator that manages page frames in my implementation, I needed to tweak the buddy allocator a bit.

This is where I made a big mistake: I thought, "hm. he's using a Vec, I'll just make an array that can always fit the free blocks!"

```rs
#[global_allocator]
pub static ALLOCATOR: Locked<BuddyAlloc<14, 16, 8192>> = Locked::new(BuddyAlloc::new(
    VirtAddr::new(HEAP_START as u64),
    VirtAddr::new(HEAP_START as u64 + HEAP_SIZE as u64),
));

// ...

/// A simple wrapper around spin::Mutex to provide safe interior mutability
pub struct Locked<A> {
    inner: spin::Mutex<A>,
}

// ...

/// # Type Parameters
/// * `L`: Number of levels in the buddy system
/// * `S`: Size of the smallest block in bytes
/// * `N`: Maximum number of blocks at each level (fixed to avoid const generics)
/// ...
pub struct BuddyAlloc<const L: usize, const S: usize, const N: usize> {
    heap_start: VirtAddr,
    _heap_end: VirtAddr,
    free_lists: [[usize; N]; L],
    counts: [usize; L],
}
```

This had a glaring issue. It was basically the equivalent of putting the heap ... in static memory. And then using that to point to another heap.

After some thought and a convo with some helpful folks on the [rust discord](https://discord.com/invite/rust-lang-community), I decided to use free lists for each level. That way, only the head of the struct would be stored in static memory, and the rest of the nodes would simply be in the free blocks. 

This required some re-writing of the implementation, as well as [a struct for the freelist](https://github.com/Makonede/locos/blob/c0bfda6899ba9ec011b6b23312771f8c471035bf/kernel/src/memory/alloc.rs#L71). 

```rs
/// A linked list of free memory blocks used in the buddy allocator
#[derive(Clone, Copy, Debug)]
struct FreeList {
    head: Option<NonNull<Node>>,
    len: usize,
}

impl Freelist {
    /// push a node to the start of the list in constant time
    pub const fn push(&mut self, ptr: NonNull<()>)
    /// pop a node off the start of the list in constant time
    pub const fn pop(&mut self) -> Option<NonNull<()>>
    /// traverse the list to check if a node exists
    pub fn exists(&self, ptr: NonNull<()>) -> bool
    /// remove a node from the list
    pub fn remove(&mut self, ptr: NonNull<()>)
}
/// A node in the free list
#[derive(Clone, Copy, Debug)]
struct Node {
    next: Option<NonNull<Node>>,
}
```

[`BuddyAlloc`](https://github.com/Makonede/locos/blob/c0bfda6899ba9ec011b6b23312771f8c471035bf/kernel/src/memory/alloc.rs#L182) was changed to just store the heads of the free lists
```rs
pub struct BuddyAlloc<const L: usize, const S: usize, const N: usize> {
    heap_start: VirtAddr,
    _heap_end: VirtAddr,
    free_lists: [FreeList; L],
}
```

The implementation of [`BuddyAlloc`](https://github.com/Makonede/locos/blob/c0bfda6899ba9ec011b6b23312771f8c471035bf/kernel/src/memory/alloc.rs#L193) was changed to deal with the pointers themselves, rather then the indices. An example would be [`merge_buddies`](https://github.com/Makonede/locos/blob/c0bfda6899ba9ec011b6b23312771f8c471035bf/kernel/src/memory/alloc.rs#L278)

```rs
/// Recursively merges a freed block with its buddy if possible
fn merge_buddies(&mut self, level: usize, ptr: NonNull<()>) {
    if level == 0 {
        self.free_lists[level].push(ptr);
        return;
    }

    let block_size = Self::block_size(level);
    let buddy = ptr.as_ptr() as usize ^ block_size;
    let buddy_nonnull = NonNull::new(buddy as *mut ()).unwrap();

    
    if self.free_lists[level].exists(buddy_nonnull) {
        // remove buddies from the free list
        self.free_lists[level].remove(buddy_nonnull);

        // add merged block to next level
        let first_buddy = core::cmp::min(ptr, buddy_nonnull);

        self.merge_buddies(level - 1, first_buddy);
    } else {
        self.free_lists[level].push(ptr);
    }
}
```

Now, you might have noticed an issue with this. Merging the buddies still takes `O(n)` time! 

To be honest, I'm not very clear on how we could fix this. I've seen implementations that either have `O(n)` alloc and constant time dealloc, `O(log n)` alloc and dealloc, or what both Nikos and I have - constant time alloc and `O(n)` dealloc. I might have just not researched it enough. A friend suggested I hash it to index an array containing all the free blocks, but I'm not sure of the implementation or the space tradeoff of that. Likewise, some implementations use a header in each memory block, but that uses space as well.

In the end, I decided to settle with this allocator for now. In the future, I might optimize it more or switch to a pre-made allocator, as making this was more for the knowledge itself.

Thanks for reading, and if you have any improvements (either for the post or the allocator), shoot me a message on my discord `jayandjeff`.
