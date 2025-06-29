---
layout: post
title: Better, harder, faster, stronger allocators
date: 2025-06-28 9:30:00 -0800
categories: locos programming featured
---

It's been a bit since the last post, but we are moving forward to the second step of multitasking. This blog is a bit behind of my development - at the time of writing this, I've already created a working kernel multitasking system and am starting to transition to userspace, but I hope this is useful nonetheless.

### Physical memory management

Before multitasking, we need to get a better frame allocator. Remember that a frame allocator manages the physical memory frames of a machine, allowing them to be mapped and making sure it doesn't give the same frame twice.

The first version of this allocator, in the very first days of locOS, was simply an iterator over the available frames given to the kernel by the bootloader. The issue with this, of course, was that it couldn't deallocate anything and eventually all memory would become used up.

### Freelist allocator

After using the simple allocator, I thought that it would be nice to have an allocator that could actually deallocate frames. The freelist allocator, the next step, was perfect for this.

A freelist allocator is simply a struct managing a list of free frames. Because my kernel had HHDM (Higher Half Direct Mapping, where all physical memory is mapped one-to-one with an offset in the higher half of the virtual memory space), I thought it would be a good idea to implement this as a linked list where each node was stored in place in the frame it represented itself. Even better, it kept the constant time properties of the previous allocator while adding on constant time deallocation, as all you needed to do was push or pop a frame to the list for deallocation and allocation respectively.

```rs
#[derive(Clone, Copy, Debug)]
struct FrameFreeList {
    head: Option<NonNull<FrameNode>>,
    len: usize,
}

impl FrameFreeList {
    // ...

    /// Pushes a frame onto the free list.
    const fn push(&mut self, ptr: NonNull<()>) {
        let node = ptr.cast::<FrameNode>();
        unsafe {
            node.write(FrameNode { next: self.head });
        }
        self.head = Some(node);
        self.len += 1;
    }

    /// Pops a frame from the free list.
    const fn pop(&mut self) -> Option<NonNull<()>> {
        if let Some(node) = self.head {
            self.head = unsafe { node.as_ref().next };
            self.len -= 1;
            Some(node.cast())
        } else {
            None
        }
    }
}
```

I could've stopped here. This allocator satisfied all the current needs of my kernel *in the moment*, but after doing a bit of research, I found out that this allocator was missing a key feature that would be needed in the future: contiguous blocks. Hardware such as network controllers, graphics cards, or storage controllers perform DMA (Direct Memory Access) and need large, contiguous blocks of physical memory to work with.

### The buddy allocator

I realize that the previous use of a buddy allocator for managing the kernel heap was a bit unconventional. This buddy allocator, however, is perfect for its use case.

A quick review on what a buddy allocator is: it splits memory into contiguous, power of 2 sized blocks, and breaks these in half to satisfy the size and alignment needs of allocations. When a block is freed, the allocator check if it can be merged with its buddy to form a block on the next level, reducing fragmentation.

Oh wait, I already have this allocator from the heap! Let's just yoink that one and adjust it a bit to fit our needs.

**WRONG!**

The previous method of storing free blocks in free lists results in O(1) allocation, but O(n log n) deallocation because we need to first search for the block in each list, and then merge it recursively upwards.

```rs
// old heap allocator
impl<const L: usize, const S: usize> BuddyAlloc<L, S> {
    // ...

    /// Merges buddies recursively.
    fn merge_buddies(&mut self, level: usize, ptr: NonNull<()>) {
        if level == 0 {
            self.free_lists[level].push(ptr);
            return;
        }

        let block_size = Self::block_size(level);
        let buddy = ptr.as_ptr() as usize ^ block_size;
        let buddy_nonnull = NonNull::new(buddy as *mut ()).unwrap();

        if self.free_lists[level].exists(buddy_nonnull) { // <- O(n) time!!!
            // remove buddies from the free list
            self.free_lists[level].remove(buddy_nonnull);

            // add merged block to next level
            let first_buddy = core::cmp::min(ptr, buddy_nonnull);

            self.merge_buddies(level - 1, first_buddy);
        } else {
            self.free_lists[level].push(ptr);
        }
    }

    // ...
}
```

I couldn't bear having such *horrible* inefficiency in my kernel, so I went in search for a better way.

### Improving the allocator

After some digging, I found a solution that worked. We store a node for *every single available frame*. This would use a lot of memory (although still <1% of the total memory), but it would be worth it for the performance gain.

For every single node, it would store the pointer to the next and previous node in the freelist that it belongs to (effectively making a circular linked list). This way, we can remove any node anywhere in constant time, and the node index can be calculated from the address of the frame that is about to be deallocated.

```rs
/// A frame buddy allocator that manages multiple free lists for frames
pub struct FrameBuddyAllocator<const L: usize = 26> {
    free_lists: [DoubleFreeList; L],
    levels: usize,
    virt_start: usize,
    virt_end: usize,
    page_list_start: usize,
}

impl<const L: usize> FrameBuddyAllocator<L> {
    /// Deallocates a contiguous block of frames, merging with buddies if possible.
    pub unsafe fn deallocate_contiguous_frames(&mut self, addr: u64, frames: usize) {
        let page_index = (addr as usize - self.page_list_start) / 4096; // index from start of this region's pages list
        // ^----- important part! O(1) time!
        
        // merge buddies ehre
    }
}
```

Using this, we can also quickly merge buddies as we just XOR the bit of the level that it's on.

```rs
impl<const L: usize> FrameBuddyAllocator<L> {
    // ...

    /// Recursively merges a freed block with its buddy if possible, to reduce fragmentation.
    fn merge_buddies(&mut self, level: usize, ptr: NonNull<DoubleFreeListNode>) {
        if level == 0 {
            self.free_lists[level].push(ptr, self.block_size(level));
            return;
        }
        let block_size = self.block_size(level) * align_of::<DoubleFreeListNode>(); // in bytes
        let base = self.page_list_start;
        let offset = (ptr.as_ptr() as usize) - base;
        let buddy_offset = offset ^ block_size;
        let buddy_addr = base + buddy_offset;
        let buddy_ptr = NonNull::new(buddy_addr as *mut DoubleFreeListNode).unwrap();

        if unsafe { self.free_lists[level].contains(buddy_ptr, self.block_size(level)) } {
            unsafe { self.free_lists[level].remove(buddy_ptr) };
            let merged_ptr = if buddy_addr < ptr.as_ptr() as usize {
                buddy_ptr
            } else {
                ptr
            };
            self.merge_buddies(level - 1, merged_ptr);
        } else {
            self.free_lists[level].push(ptr, self.block_size(level));
        }
    }
}
```

This creates an allocator with O(1) allocation and O(log n) deallocation, but in practice deallocation is much closer to O(1) because the number of levels doesn't scale much.

We're done, right?

### The last hurdle

Not quite. As one might recall, the available frames aren't exactly in one big block perfectly sized to a power of 2 for us to slot the buddy allocator in. In practice, it's a lot of fragmented chunks of arbitrary sizes.

My solution was to create a forest of buddy allocators, and further expressing each contiguous block of frames as a sum of powers of 2 until a limit to slot buddy allocators in. For example, a block of 100 frames would be split into 64 + 32 + 4, and each of those would be managed by it's own buddy allocator.

```rs
/// A forest of frame buddy allocators, each with its own free lists for different levels.
pub struct FrameBuddyAllocatorForest<const N: usize = 100, const L: usize = 26> {
    allocators: [Option<FrameBuddyAllocator<L>>; N],
    count: usize,
    hddm_offset: u64,
}
```

When a block is allocated, we find the first allocator that can satisfy the size, then allocate from it.

```rs
impl <const N: usize, const L: usize> FrameBuddyAllocatorForest<N, L> {
    #[inline]
    pub fn allocate_pages(&mut self, pages: usize) -> Option<VirtAddr> {
        assert!(pages.is_power_of_two(), "Number of pages must be a power of two");

        for allocator in self.allocators[..self.count].iter_mut().flatten() {
            if let Some(virt_addr) = allocator.allocate_contiguous_frames(pages) {
                return Some(VirtAddr::new(virt_addr));
            }
        }
        None
    }
}
```

and when a block is deallocated, we check which allocator it belongs to and deallocate using it. This is technically O(n) time, but hopefully in practice it's not as noticeable. If anyone has a better way to do this, I'd be happy to incorporate it.

```rs
impl <const N: usize, const L: usize> FrameBuddyAllocatorForest<N, L> {
    #[inline]
    pub unsafe fn deallocate_pages(&mut self, virt_addr: VirtAddr, pages: usize) {
        assert!(pages.is_power_of_two(), "Number of pages must be a power of two");
        let addr = virt_addr.as_u64() as usize;
        
        for allocator in self.allocators[..self.count].iter_mut().flatten() {
            if addr >= allocator.virt_start && addr < allocator.virt_end {
                unsafe { allocator.deallocate_contiguous_frames(virt_addr.as_u64(), pages) };
                return;
            }
        }
        panic!("Address {:#x} not managed by any allocator", addr);
    }
}
```

And that's basically it! We now have a (in my opinion) pretty good physical memory allocator.

Thanks for reading - the next post will be on kernel multitasking.