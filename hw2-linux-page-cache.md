# Homework 2---Linux Page Cache

**Author: Cayden Lund (u1182408)**

[[[TODO: Introduction]]]

Note that this document represents the current state of the Linux kernel and API at the time of writing, v6.11.0.


## Table of Contents

[[[TODO: ToC]]]


## Virtual Memory Areas

Linux systems use **virtual addresses** to abstractly represent addresses of physical memory.
This abstraction creates a large, contiguous, isolated address space for each process, which the kernel manages through various mechanisms, including page tables and virtual memory areas.

A process's memory management context is represented by the `struct mm_struct` datatype, defined in file [`include/linux/mm_types.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm_types.h#L779).
The relevant fields of this datatype are included below, with annotated descriptions:

```C
[[[TODO]]]
struct mm_struct {
    struct {
        /*
         * Fields which are often written to are placed in a separate
         * cache line.
         */
        struct {
            /**
             * @mm_count: The number of references to &struct
             * mm_struct (@mm_users count as 1).
             *
             * Use mmgrab()/mmdrop() to modify. When this drops to
             * 0, the &struct mm_struct is freed.
             */
            atomic_t mm_count;
        } ____cacheline_aligned_in_smp;

        struct maple_tree mm_mt;

        unsigned long mmap_base;    /* base of mmap area */
        unsigned long mmap_legacy_base;    /* base of mmap area in bottom-up allocations */

        unsigned long task_size;    /* size of task vm space */
        pgd_t * pgd;

        /**
         * @mm_users: The number of users including userspace.
         *
         * Use mmget()/mmget_not_zero()/mmput() to modify. When this
         * drops to 0 (i.e. when the task exits and there are no other
         * temporary reference holders), we also release a reference on
         * @mm_count (which may then free the &struct mm_struct if
         * @mm_count also drops to 0).
         */
        atomic_t mm_users;

        int map_count;            /* number of VMAs */

        spinlock_t page_table_lock; /* Protects page tables and some
                         * counters
                         */
        /*
         * With some kernel config, the current mmap_lock's offset
         * inside 'mm_struct' is at 0x120, which is very optimal, as
         * its two hot fields 'count' and 'owner' sit in 2 different
         * cachelines,  and when mmap_lock is highly contended, both
         * of the 2 fields will be accessed frequently, current layout
         * will help to reduce cache bouncing.
         *
         * So please be careful with adding new fields before
         * mmap_lock, which can easily push the 2 fields into one
         * cacheline.
         */
        struct rw_semaphore mmap_lock;

        struct list_head mmlist; /* List of maybe swapped mm's.    These
                      * are globally strung together off
                      * init_mm.mmlist, and are protected
                      * by mmlist_lock
                      */

        unsigned long hiwater_rss; /* High-watermark of RSS usage */
        unsigned long hiwater_vm;  /* High-water virtual memory usage */

        unsigned long total_vm;       /* Total pages mapped */
        unsigned long locked_vm;   /* Pages that have PG_mlocked set */
        atomic64_t    pinned_vm;   /* Refcount permanently increased */
        unsigned long data_vm;       /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
        unsigned long exec_vm;       /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
        unsigned long stack_vm;       /* VM_STACK */
        unsigned long def_flags;

        /**
         * @write_protect_seq: Locked when any thread is write
         * protecting pages mapped by this mm to enforce a later COW,
         * for instance during page table copying for fork().
         */
        seqcount_t write_protect_seq;

        spinlock_t arg_lock; /* protect the below fields */

        unsigned long start_code, end_code, start_data, end_data;
        unsigned long start_brk, brk, start_stack;
        unsigned long arg_start, arg_end, env_start, env_end;

        unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

        struct percpu_counter rss_stat[NR_MM_COUNTERS];

        struct linux_binfmt *binfmt;

        /* Architecture-specific MM context */
        mm_context_t context;

        unsigned long flags; /* Must use atomic bitops to access */

        struct user_namespace *user_ns;

        /* store ref to file /proc/<pid>/exe symlink points to */
        struct file __rcu *exe_file;

        /*
         * An operation with batched TLB flushing is going on. Anything
         * that can move process memory needs to flush the TLB when
         * moving a PROT_NONE mapped page.
         */
        atomic_t tlb_flush_pending;
        struct uprobes_state uprobes_state;

#ifdef CONFIG_IOMMU_MM_DATA
        struct iommu_mm_data *iommu_mm;
#endif
    } __randomize_layout;

    /*
     * The mm_cpumask needs to be at the end of mm_struct, because it
     * is dynamically sized based on nr_cpu_ids.
     */
    unsigned long cpu_bitmap[];
};
```

**Virtual memory areas** (VMAs) represent contiguous regions of virtual memory in a process's address space and are central to the way that the Linux kernel organizes memory allocation.
[[[TODO: ...]]]

A VMA is represented in the kernel by datatype `struct vm_area_struct`, defined in file [`include/linux/mm_types.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm_types.h#L664).
The relevant fields of this datatype are included below, with annotated descriptions:

```C
[[[TODO]]]
struct vm_area_struct {
    /* The first cache line has the info for VMA tree walking. */

    union {
        struct {
            /* VMA covers [vm_start; vm_end) addresses within mm */
            unsigned long vm_start;
            unsigned long vm_end;
        };
    };

    struct mm_struct *vm_mm;    /* The address space we belong to. */
    pgprot_t vm_page_prot;          /* Access permissions of this VMA. */

    /*
     * Flags, see mm.h.
     * To modify use vm_flags_{init|reset|set|clear|mod} functions.
     */
    union {
        const vm_flags_t vm_flags;
        vm_flags_t __private __vm_flags;
    };

    /*
     * For areas with an address space and backing store,
     * linkage into the address_space->i_mmap interval tree.
     *
     */
    struct {
        struct rb_node rb;
        unsigned long rb_subtree_last;
    } shared;

    /*
     * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
     * list, after a COW of one of the file pages.    A MAP_SHARED vma
     * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
     * or brk vma (with NULL file) can only be in an anon_vma list.
     */
    struct list_head anon_vma_chain; /* Serialized by mmap_lock &
                      * page_table_lock */
    struct anon_vma *anon_vma;    /* Serialized by page_table_lock */

    /* Function pointers to deal with this struct. */
    const struct vm_operations_struct *vm_ops;

    /* Information about our backing store: */
    unsigned long vm_pgoff;        /* Offset (within vm_file) in PAGE_SIZE
                       units */
    struct file * vm_file;        /* File we map to (can be NULL). */
    void * vm_private_data;        /* was vm_pte (shared mem) */

#ifdef CONFIG_SWAP
    atomic_long_t swap_readahead_info;
#endif

    struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
} __randomize_layout;
```

Each VMA has a set of operations that can be performed on it.
These operations can include the handling of page faults [[[TODO: ...]]].
In the Linux source code, the datatype `struct vm_operations_struct` defines the operations set for a particular VMA, defined in file [`include/linux/mm.h` (link to source)](https://elixir.bootlin.com/linux/v6.11/source/include/linux/mm.h#L588).

```C
[[[TODO]]]
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs.
 */
struct vm_operations_struct {
    void (*open)(struct vm_area_struct * area);
    /**
     * @close: Called when the VMA is being removed from the MM.
     * Context: User context.  May sleep.  Caller holds mmap_lock.
     */
    void (*close)(struct vm_area_struct * area);
    /* Called any time before splitting to check if it's allowed */
    int (*may_split)(struct vm_area_struct *area, unsigned long addr);
    int (*mremap)(struct vm_area_struct *area);
    /*
     * Called by mprotect() to make driver-specific permission
     * checks before mprotect() is finalised.   The VMA must not
     * be modified.  Returns 0 if mprotect() can proceed.
     */
    int (*mprotect)(struct vm_area_struct *vma, unsigned long start,
            unsigned long end, unsigned long newflags);
    vm_fault_t (*fault)(struct vm_fault *vmf);
    vm_fault_t (*huge_fault)(struct vm_fault *vmf, unsigned int order);
    vm_fault_t (*map_pages)(struct vm_fault *vmf,
            pgoff_t start_pgoff, pgoff_t end_pgoff);
    unsigned long (*pagesize)(struct vm_area_struct * area);

    /* notification that a previously read-only page is about to become
     * writable, if an error is returned it will cause a SIGBUS */
    vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);

    /* same as page_mkwrite when using VM_PFNMAP|VM_MIXEDMAP */
    vm_fault_t (*pfn_mkwrite)(struct vm_fault *vmf);

    /* called by access_process_vm when get_user_pages() fails, typically
     * for use by special VMAs. See also generic_access_phys() for a generic
     * implementation useful for any iomem mapping.
     */
    int (*access)(struct vm_area_struct *vma, unsigned long addr,
              void *buf, int len, int write);

    /* Called by the /proc/PID/maps code to ask the vma whether it
     * has a special name.  Returning non-NULL will also cause this
     * vma to be dumped unconditionally. */
    const char *(*name)(struct vm_area_struct *vma);

    /*
     * Called by vm_normal_page() for special PTEs to find the
     * page for @addr.  This is useful if the default behavior
     * (using pte_page()) would not find the correct page.
     */
    struct page *(*find_special_page)(struct vm_area_struct *vma,
                      unsigned long addr);
};
```


## Red-black Trees for VMAs

[[[TODO: red-black trees]]]


## Reverse Mapping

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
