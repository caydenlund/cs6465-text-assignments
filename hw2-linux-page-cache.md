# Homework 2---Linux Page Cache

**Author: Cayden Lund (u1182408)**

[[[TODO: Introduction]]]

Note that this document represents the current state of the Linux kernel and API at the time of writing, v6.11.0.


## Table of Contents

[[[TODO: ToC]]]


## Virtual Memory Areas

Linux systems use **virtual addresses** to abstractly represent addresses of physical memory.
This abstraction creates a large, contiguous, isolated address space for each process, which the kernel manages through various mechanisms, including page tables and virtual memory areas.

A process is represented in Linux by a `struct task_struct`, which I won't discuss in detail because it doesn't store any information that isn't in other data structures that I'll be diving into; the key property of it, though, is that it holds a `struct mm_struct`, which holds information about the process's virtual memory layout, defined in file [`include/linux/mm_types.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm_types.h#L779).
The relevant fields of this datatype are included below, with annotated descriptions:

```C
// Represents the virtual memory layout of a process.
struct mm_struct {
    // These are placed first because they're frequently updated
    // (cache optimization).
    struct {
        struct {
            // An atomic count of the number of references to this `mm_struct`.
            // When the count reaches 0, the `mm_struct` is freed.
            atomic_t mm_count;
        } ____cacheline_aligned_in_smp;

        // The maple tree (compare to the old red-black tree) of VMAs.
        struct maple_tree mm_mt;

        // The base address for `mmap()` mappings.
        unsigned long mmap_base;
        // The base address for `mmap()` mappings using legacy ("bottom-up")
        // allocation methods.
        unsigned long mmap_legacy_base;
        // The total size of the process's virtual memory space.
        unsigned long task_size;

        // A pointer to the top-level page directory for this process.
        pgd_t * pgd;

        // Tracks the number of user-space processes or threads using the memory.
        // When it drops to 0, it releases a reference on `mm_count`.
        atomic_t mm_users;

        // The total number of VMAs in the process.
        int map_count;

        // A simple spinlock to protect modifications to page tables
        // and some counters.
        spinlock_t page_table_lock;

        // A reader-writer semaphore to synchronize access to the VMA tree.
        // Follows the "concurrent readers, exclusive writer" model.
        struct rw_semaphore mmap_lock;

        // These store high-watermarks for the process's memory usage.
        // Tracks the highest "resident set size", the amount of physical memory
        // used by the process.
        unsigned long hiwater_rss;
        // Tracks the highest amount of virtual memory allocated.
        unsigned long hiwater_vm;

        // The total number of VM pages mapped by the process.
        unsigned long total_vm;
        // The number of pages that have been locked into physical memory
        // and can't be swapped out.
        unsigned long locked_vm;
        // A 64-bit atomic counter tracking the number of pages whose reference count
        // has been permanently incremented (i.e., "pinned").
        atomic64_t    pinned_vm;

        // A lock for the following fields.
        spinlock_t arg_lock;

        // The beginning and end of the executable and initialized data segments
        // of the program (respectively).
        unsigned long start_code, end_code, start_data, end_data;
        // The current end of the heap.
        unsigned long start_brk, brk, start_stack;

        // Architecture-specific memory management information.
        mm_context_t context;

        // Various flags related to memory management.
        unsigned long flags;

        // The user namespace associated with the process.
        struct user_namespace *user_ns;

        // A reference to the executable file for the process.
        struct file __rcu *exe_file;
    } __randomize_layout;

    // A dynamically-sized array tracking the CPUs that have executed parts
    // of the process.
    unsigned long cpu_bitmap[];
};
```

**Virtual memory areas** (VMAs) represent contiguous regions of virtual memory in a process's address space and are central to the way that the Linux kernel organizes memory allocation.
VMAs are used to map files, allocate memory, and set up shared memory.

A VMA is represented in the kernel by datatype `struct vm_area_struct`, defined in file [`include/linux/mm_types.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm_types.h#L664).
The relevant fields of this datatype are included below, with annotated descriptions:

```C
// Describes a virtual memory area.
// There is one of these for each virtual memory area, for each task.
struct vm_area_struct {
    struct {
        The VMA covers virtual address starting a `vm_start` and going up to `vm_end - 1`.
        unsigned long vm_start;
        unsigned long vm_end;
    };

    // This is a pointer to the address space that owns us.
    struct mm_struct *vm_mm;
    // Defines the access permissions of this VMA.
    pgprot_t vm_page_prot;

    // Various different virtual memory flags.
    const vm_flags_t vm_flags;

    // For some areas with an address space, a red-black tree is still used.
    // In those cases, this is a linkage into that tree.
    struct {
        struct rb_node rb;
        unsigned long rb_subtree_last;
    } shared;

    // A chain of anonymous VM areas.
    struct list_head anon_vma_chain;
    // A pointer to the descriptor of this area's anonymous memory,
    // if this is an anonymous VMA.
    struct anon_vma *anon_vma;

    // Various different functions that are designed to work with this VMA.
    const struct vm_operations_struct *vm_ops;

    // Information about the backing store:
    // Offset in multiples of `PAGE_SIZE`.
    unsigned long vm_pgoff;
    // If this is a file-mapped page, then this is the file that we map to.
    // Otherwise, this is `NULL`.
    struct file * vm_file;
    void * vm_private_data;
};
```

Each VMA has a set of operations that can be performed on it, depending on the type of memory that it represents.
The kernel uses the functions defined for this VMA to perform tasks such as handling page faults, managing memory access, and making sure that the memory is properly synchronized with the backing store.
In the Linux source code, the datatype `struct vm_operations_struct` defines the operations set for a particular VMA, defined in file [`include/linux/mm.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm.h#L588).
I've included this datatype below, with in-depth annotations:

```C
// These are the functions for a virtual memory area, as a set of function pointers.
struct vm_operations_struct {
    // This is called whenever a new reference is created to a VMA
    // (e.g., through a process calling `fork()`).
    // Counter/lock setup is done here.
    void (*open)(struct vm_area_struct * area);

    // This is called when the VMA is being removed from the memory map,
    // such as when a process terminates or calls `munmap()`.
    // Releases resources, etc.
    void (*close)(struct vm_area_struct * area);

    // Before the kernel attempts to split a VMA into multiple regions,
    // this function is called to check whether splitting is allowed.
    // Returns 0 if the split is allowed, or an error status code otherwise.
    int (*may_split)(struct vm_area_struct *area, unsigned long addr);

    // Called when `mremap()` is resizing the memory region.
    int (*mremap)(struct vm_area_struct *area);

    // Does driver-specific or filesystem-specific checks to see whether
    // the protection flags of the VMA can safely be changed.
    int (*mprotect)(struct vm_area_struct *vma, unsigned long start,
            unsigned long end, unsigned long newflags);

    // Called when a page fault occurs.
    vm_fault_t (*fault)(struct vm_fault *vmf);
    // Like `fault`, but for huge pages.
    vm_fault_t (*huge_fault)(struct vm_fault *vmf, unsigned int order);

    // Allows for the kernel to map multiple contiguous pages at once.
    vm_fault_t (*map_pages)(struct vm_fault *vmf,
            pgoff_t start_pgoff, pgoff_t end_pgoff);

    // Returns the size of the pages in the VMA.
    // (Usually 4 KiB, unless it's huge pages.)
    unsigned long (*pagesize)(struct vm_area_struct * area);

    // Called when a previously-read-only page is about to become writable
    // due to a page fault.
    vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);

    // Like `page_mkwrite` but for VMAs with specific flags that map
    // physical page frame numbers directly into userspace.
    vm_fault_t (*pfn_mkwrite)(struct vm_fault *vmf);

    // Called when the kernel needs to access a memory range.
    // Used for (for example) profiling and debugging.
    int (*access)(struct vm_area_struct *vma, unsigned long addr,
              void *buf, int len, int write);

    // A special name assigned to this VMA, or `NULL`.
    const char *(*name)(struct vm_area_struct *vma);

    // Finds the page for a particular address in the case that regular
    // reverse mapping is insufficient.
    struct page *(*find_special_page)(struct vm_area_struct *vma,
                      unsigned long addr);
};
```


## Maple Trees for VMAs

I'm excited to talk about this, because this is a very recent change!
Before now, the Linux kernel used a red-black tree to identify a particular virtual memory area quickly.
The VMAs are dynamically managed and need to be accessed, inserted, and deleted frequently, and the self-balancing red-black tree was very effective at searching for an entry in the tree and performing updates to the tree with little overhead.

Historically, the Linux kernel also kept a doubly-linked list of VMAs, sorted from lowest address to highest address. However, in kernel release v6.1.0, the red-black tree and the linked list were both removed and replaced with a new data structure: the **maple tree**, which was fast enough that the extra work for managing the linked list outweighed the potential benefits.
The maple tree is related to B-trees, and as such the nodes in the tree contain multiple elements.
Specifically, leaf nodes can hold up to 16 elements, and internal nodes can hold up to 10.
This has a profound impact on runtime speed: elements can be inserted and removed frequently without having to re-balance the tree, because it can be done in this already-allocated space.
It also allows the tree to take advantage of CPU caches more efficiently: nodes in the tree require at most 256 bytes, which is a multiple of common cache line sizes.
Finally, perhaps the best performance advantage is that it was designed to work locklessly, using a read-copy-update model.

An interesting detail about the maple tree is that the developers found that the implementation was so performant that it was as fast as the VMA cache that was in use, which tracked most-recently-accessed VMAs for quick lookup.
Therefore, as a part of the introduction of the maple tree to manage VMAs, the VMA cache was completely removed.

In the kernel, maple trees are represented in the (very simple) data structure `struct maple_tree`, which is defined in file [`include/linux/maple_tree.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/maple_tree.h#L219).
The fields of this datatype are included below, with annotated descriptions:

```C
// A highly-performant balanced search tree
// for handling ranges of memory addresses.
struct maple_tree {
    // A mutex for synchronization.
    union {
        // Either use an internal spinlock, like so...
        spinlock_t ma_lock;

        // ...or use one externally.
        lockdep_map_p ma_external_lock;
    };

    // Flags providing information about the tree.
    unsigned int ma_flags;

    // A pointer to the root of the tree.
    void __rcu *ma_root;
};
```

The maple tree logic is defined in a few functions, which I work through below.

First, insertions are done using the `mtree_insert` function, which is implemented in file [`lib/maple_tree.c` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/lib/maple_tree.c#L6446).

```C
// Inserts the given entry at the given index in the given maple tree,
// if there is not already such a value.
// Arguments:
//   - mt:     The maple tree.
//   - index:  The index where to store the value.
//   - entry:  The entry to store in the tree.
//   - gfp:    The `GFP_FLAGS` to use for allocations.
//
// Returns:
//   - `0` on success
//   - `-EEXISTS` if the range is occupied
//   - `-EINVAL` on an invalid request
//   - `-ENOMEM` if memory couldn't be allocated.
int mtree_insert(struct maple_tree *mt, unsigned long index, void *entry, gfp_t gfp) {
    // This simply defers to the `mtree_insert_range` implementation,
    // which inserts a range of values, and we give it a "range" containing
    // only this index.
    return mtree_insert_range(mt, index, index, entry, gfp); }
}
```

We'll jump to the [`mtree_insert_range` implementation](https://elixir.bootlin.com/linux/v6.11/source/lib/maple_tree.c#L6411):

```C
int mtree_insert(struct maple_tree *mt, unsigned long index, void *entry, gfp_t gfp) {
    // Initialize a new `mas_state` object named `ms`.
    // This macro basically works like a constructor, initializing the fields to default values.
    // I've included this data structure below for reference.
    MA_STATE(ms, mt, first, last);

    // If the given `entry` is _advanced_, meaning that it's an invalid state
    // or was modified in an invalid way, then return.
    if (WARN_ON_ONCE(xa_is_advanced(entry)))
        return -EINVAL;

    // If the start of the range is after the end of the range,
    // then this is an invalid request.
    if (first > last)
        return -EINVAL;

    // Locks the modification of the maple tree.
    mtree_lock(mt);

    // Try to insert `entry` into the maple tree.
retry:
    mas_insert(&ms, entry);

    // Did it fail because of a memory allocation issue?
    // If so, try again.
    if (mas_nomem(&ms, gfp))
        goto retry;

    // Now that the maple tree has been updated, unlock it so that others
    // can modify it now if needed.
    mtree_unlock(mt);

    // If there was an error, then report it to the user...
    if (mas_is_err(&ms))
        return xa_err(ms.node);

    // ...otherwise, return 0 ("OK").
    return 0;
}
```

The `ma_state` datatype is defined in file [`include/linux/maple_tree.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/maple_tree.h#L426):

```C
struct ma_state {
    // A pointer to the tree that we're working in.
    struct maple_tree *tree;

    // The index where we're working---start of the range.
    unsigned long index;
    // The last index of the range where we're working.
    unsigned long last;

    // The maple node containing this entry.
    struct maple_enode *node;

    // The minimum index of this node.
    // This is an implied pivot minimum.
    unsigned long min;
    // The maximum index of this node.
    // This is an implied pivot maximum.
    unsigned long max;

    // The nodes that were allocated for this operation.
    struct maple_alloc *alloc;

    // The current state of the operation.
    enum maple_status status;

    // The depth of the tree descent.
    unsigned char depth;

    unsigned char offset;
    unsigned char mas_flags;

    // The end of the maple node (byte index).
    unsigned char end;
};
```

Fetching values from the maple tree works in a similar way.
Function `mtree_load` is implemented in file [`lib/maple_tree.c` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/lib/maple_tree.c#L6314)

```C
// Gets the value at the given index from the given maple tree.
// Returns the relevant entry, or `NULL`.
void *mtree_load(struct maple_tree *mt, unsigned long index) {
    // Once again, initialize a new `mas_state` object.
    // This time, it's named `mas` instead of `ms`.
    // I'm curious whether that's intentional.
    MA_STATE(mas, mt, index, index);

    // This will hold the value to return.
    void *entry;

    // This is probably for performance analysis or debugging.
    trace_ma_read(__func__, &mas);

    // Lock the tree for read access (under the RCU model).
    rcu_read_lock();

retry:
    // Start the traversal of the maple tree.
    // It returns the value if found, or reports the status.
    entry = mas_start(&mas);

    // If there was no entry at that index, then unlock the tree
    // and return `NULL`.
    if (unlikely(mas_is_none(&mas)))
        goto unlock;

    // If the current entry is a pointer that might contain further data structures:
    // if `index` is non-zero, then unlock and return `NULL`.
    // Otherwise, just unlock and return the entry.
    if (unlikely(mas_is_ptr(&mas))) {
        if (index)
            entry = NULL;

        goto unlock;
    }

    // Otherwise, perform the actual tree traversal.
    entry = mtree_lookup_walk(&mas);
    // If the lookup failed for some reason, then the traversal is still at the start,
    // so try it again.
    if (!entry && unlikely(mas_is_start(&mas)))
        goto retry;

unlock:
    // Unlock the maple tree.
    rcu_read_unlock();

    // Return the entry if found, or `NULL` otherwise.
    if (xa_is_zero(entry))
        return NULL;

    return entry;
}
```

I could talk about maple trees all day.
I find this really interesting.


## Reverse Mapping

At a high level, **reverse mapping** provides the ability to trace which virtual pages are mapped to which physical pages, of which there are two cases: _anonymous memory_ and _file-mapped memory_.
This is, of course, used to resolve addresses for loads and stores, but it is also used for some more advanced operations such as page reclaiming, moving pages between NUMA nodes, swapping to disk, and memory defragmentation.

**File-mapped memory** is memory that is backed by files, used for shared memory regions (such as for shared libraries) and memory-mapped files.
This is allocated with `mmap()` with a file descriptor.
When a VMA is file-mapped memory, then the `vm_file` field of the associated `struct vm_area_struct` representing that VMA is set to the relevant file.

On the other hand, **anonymous memory** is not backed by a file, such as pages allocated for the heap or the stack.
This is usually allocated with `brk()`, `sbrk()`, or `mmap()` without a file descriptor.
When a VMA is anonymous memory, then the `anon_vma` field of the the associated `struct vm_area_struct` representing that VMA is set to the relevant `struct anon_vma` describing that anonymous memory area.

To make this more clear, I've prepared an illustration of this:


<p align="center">
    <img alt="(Reverse mapping of `vm_area_struct`)" src="./hw2-reverse-mapping.png" width="75%" />
    <br />
    <i>If the <code>vm_area_struct</code> is mapped to a file, then the <code>vm_file</code> field will point to the mapped file.</i>
    <br />
    <i>Otherwise, the <code>vm_area_struct</code> represents anonymous memory, and the <code>anon_vma</code> field points to the relevant <code>anon_vma</code> that describes it.</i>
</p>

When performing reverse mapping, you first identify the VMA in which the virtual address resides.
This is done quite quickly using the maple tree, as I discussed earlier.
Then, once we know the area and have the corresponding `vm_area_struct`, we can check whether it's mapped to a file or anonymously by checking whether fields `vm_file` and `anon_vma` are set.

For file-mapped memory, the kernel uses the `page->mapping` field, which points to the `address_space` structure associated with the file.
Let's examine this data structure. 
It represents the mapping between a file and its memory pages.
It's defined in file [`include/linux/fs.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/fs.h#L466).
I've included the relevant fields below, with annotations:

```C
// Represents the mapping between a file and its memory pages.
struct address_space {
    // A pointer to the `inode` structure that represents the file or device
    // associated with this `address_space`.
    // The `inode` structure contains metadata about the file,
    // including its size, permissions, and timestamps.
    struct inode *host;

    // A radix-tree-based data structure.
    // It can efficiently store and retrieve pages of memory.
    // It holds the mapping from the file offset to the corresponding
    // in-memory pages.
    struct xarray i_pages;

    // A read-write semaphore used to protect this `address_space`
    // when invalidating pages (e.g., when unmapping pages from the page cache).
    struct rw_semaphore invalidate_lock;

    // The "Get Free Pages" flags---they specify how memory allocations for this space
    // should be performed.
    // E.g., states whether pages should be allocated from the page cache
    // or from elsewhere.
    gfp_t gfp_mask;

    // A counter that tracks how many writable memory-mapped regions are associated
    // with this `address_space`.
    atomic_t i_mmap_writable;

    // Represents a cache of red-black binary search tree entries in the VMA tree
    // that maps this address space.
    // I guess they didn't remove it from the data structure here, even though I read
    // that they stopped doing the red-black tree cache when they stopped using the RB tree
    // in kernel v6.1.
    struct rb_root_cached i_mmap;

    // The number of pages currently loaded into memory for this `address_space`.
    // (I.e., how many pages from the file are currently in the RAM.)
    unsigned long nrpages;

    // The page index where writeback operations should resume
    // (i.e., writing dirty pages from memory back to disk).
    pgoff_t writeback_index;

    // Function pointers that define operations for this `address_space`.
    // E.g., reading, writing, memory management functions.
    const struct address_space_operations *a_ops;

    // Flags that manage the state of this `address_space`.
    // E.g., whether it's dirty or clean, etc.
    unsigned long flags;

    // Tracks writeback errors that occur while writing data to the file
    // associated with this `address_space`.
    errseq_t wb_err;

    // A lock for the `i_private_list` field.
    spinlock_t i_private_lock;
    // A linked list of private data associated with the `address_space`.
    // E.g., pages, other metadata external to the mapped data that need
    // to be managed outside the normal page cache.
    struct list_head i_private_list;

    // A read-write semaphore for the `i_mmap` tree.
    struct rw_semaphore i_mmap_rwsem;

    // A generic pointer to private data associated with this `address_space`.
    void * i_private_data;
};
```

### Reverse Mapping for File-Mapped Memory

[[[TODO: reverse mapping for anonymous and file-mapped memory]]]


## Address Spaces

[[[TODO: address spaces]]]


## Buffer Cache

[[[TODO]]]


## Swap and Paging

[[[TODO]]]


## Page Faults

[[[TODO: page faults]]]

[[[TODO: swap in/swap out operations]]]
