# Homework 4---Paravirtualizing xv6

**Author: Cayden Lund (u1182408)**

For this assignment, we're asked to design a paravirtualization plan for xv6, running it within an xv6 machine.


## 1. Boot Protocol

We need to remove the real-mode boot code for the `xv6-pv` kernel.
Because it isn't actually booting from BIOS, the system will be in protected mode already.
The `xv6-pv` version of `bootasm.S` will need to be simplified with this change.
There will also need to be a new entry point in `main.c`, instead of jumping to the regular `main()` function directly, that initializes segmentation and paging, like how `bootasm.S` does for the regular `xv6` kernel.

The host kernel will allocate memory for `xv6-pv` and load the `xv6-pv` ELF segments, set up initial page tables, and jump to the new `pv_boot()` function, passing a special parameter `struct boot_info` describing key information.

It would look something like this:

```
enum memmap_entry_type {
    MEMMAP_TYPE_USABLE   = 1,  // Available memory
    MEMMAP_TYPE_RESERVED = 2,  // Reserved/unusable
    MEMMAP_TYPE_KERNEL   = 3,  // Constains kernel binary
};

struct memmap_entry {
    uint32_t base_addr;           // Start of memory region
    uint32_t length;              // Length of memory region
    enum memmap_entry_type type;  // Memory type
};

struct boot_info {
    uint32_t vcpu_count;
    struct memmap_entry* memmap; // Array of memory regions
    uint32_t memmap_size;        // Length of memmap array
    uint32_t pgd_paddr;          // Page directory physical address
    uint32_t free_mem_start;     // Start of free memory after kernel binary
};

// Host calls this to launch xv6-pv
void launch_pv() {
    // Load guest binary
    void* 

    struct bootinfo bi;
    bi.vcpu_count = 1;
    bi.pgd_paddr = /* ... */;
}

void pv_boot(struct boot_info* boot_info) {
    pv_segmentation_init();
    pv_paging_init(); 
    main();
}
```


## 2. Memory Layout

### Memory Layout Strategy

xv6's normal kernel virtual memory layout starts at `KERNBASE`, defined as `0x80000000`.
xv6-pv, on the other hand, will run in a lower address space as a user process.

We'll create a contiguous region of virtual memory that will serve as xv6-pv's "physical" memory pool.
The user-space memory region will be much smaller, but the sizes of the kernel space will remain the same.

First of all, here is the (unmodified) xv6 memory layout, in format `(virtual range -> physical range)`:
- `0x00000000..0x80000000 -> (dynamic)`: User space.
- `0x80000000..0x80100000 -> 0x0000000..0x00100000`: I/O space.
- `0x80100000..data -> 0x00100000..(data-0x80000000)`: Kernel text + R/O data. `data` is defined in `kernel.ld` as the address at the beginning of the following data section (i.e., after the end of the text + R/O data).
- `data..0x8E000000 -> (data-0x80000000)..0x0E000000`: R/W data + free physical memory.
- `0xFE000000..0 -> 0xFE000000..0`: Memory-mapped devices.

This is the new xv6-pv memory layout, again in format `(virtual range -> guest "physical" range)`:

- `0x60000000..0x70000000 -> (dynamic)` xv6-pv user space.
- `0x70000000..0x70100000 -> 0x00000000..0x00100000`: Virtualized I/O space.
- `0x70100000..PV_DATA -> 0x0010000..(PV_DATA-0x70000000)`: Kernel text + R/O data. `PV_DATA` is defined as the address at the beginning of the following data section.
- `PV_DATA..0x7e000000 -> (PV_DATA-0x70000000)..0x0E000000`: R/W data + free "physical" memory.
- `0x7E000000..0x80000000 -> 0x7E000000..0x80000000`: Virtualized memory-mapped devices.

This will require modifying `vm.c` to include a `pv_regions[]` array, describing the different regions as above.
Also, we'll define analogous `PV_P2V(pa)` and `PV_V2P(va)` macros.
We'll also need to write corresponding `setupkvm_pv` and `setupuvm_pv` functions to initialize the memory regions on booting xv6-pv.

We'll also need to update the `kernel.ld` linker script to define `PV_DATA`.

### Implementation of Isolation

The xv6-pv kernel process is running in user space, which means that it can't run privileged instructions.
When it tries to run a privileged instruction (`cli`, `sti`, `lgdt`, `lidt`, etc.), the CPU issues a GPF, which xv6 catches and emulates.
This allows xv6 to ability to enforce isolation: it can check whether an instruction would break the isolation invariants before running it.
This also allows xv6-pv to issue system calls that are intercepted and translated to host system calls.

xv6-pv processes may not access memory outside of the `0x60000000..0x80000000` range.
The host xv6 kernel will enforce this through its page tables; a new `mappages_pv` method will emulate the `mappages` call, operating on the host's page tables and checking that the mapped pages are in the right address range.

Each process will also need to track an associated virtual process state, tracking register `eflags`, `idt`, `gdt`, privilege level `mode`, and trapframe `tf`.

`trap.c` will need to be modified to handle general protection faults in `trap_pv`, as well as a new `mappages_pv` method, `syscall_pv` method, `init_process_pv` method, and `trap_interrupt_pv` method.

### How Page Tables and Segmentation Are Used

I've discussed paging in detail above.

Segmentation is virtualized in xv6-pv.
The virtualized GDT will describe code and data segments for the xv6-pv kernel, and user segments for the user space.
This GDT needs to be initialized per-process.
The `syscall_pv` function in the xv6 host will need to perform privilege checks based on the virtual GDT.

I've already described the changes that need to be done for emulating `lgdt` and other privileged instructions.
With these changes, page tables and interrupts will be virtualized and emulated.


## 3. Execution Environment

Creating the execution environment for an xv6-pv user process will involve several layers of abstraction.

xv6-pv user processes are isolated from the host system's processor and run in a controlled user-space environment.
Each xv6-pv user process will then need to map to a host xv6 process to get actual CPU time, where these host processes will effectively act as containers for the virtualized user-space program.

Each xv6-pv user process will have its own address space within the xv6-pv memory layout.
xv6-pv will create page tables for each of its user processes within the constraints of the memory space defined above (`0x60000000..0x80000000`).
xv6-pv will then need to define a `setupuvm_pv` function to set up the user virtual memory space, as well as `allocproc` and `freeproc` in `proc.c` to deal with the associated allocations and deallocations.


## 4. System Calls and Interrupts

System calls and privilege management are described in section [Implementation of Isolation](#implementation-of-isolation).


## 5. Context Switch

In xv6-pv, each guest user process must maintain its own context, including registers, page tables, (emulated) privilege level, guest trapframe, and page directory.

A new `pv_data` structure in file `proc.h` will need to be added describing this information.
The `proc` structure in file `proc.h` will also need to be modified to include a pointer to a `pv_data` object, which will be equal to `NULL` if not a virtualized process.

`switchuvm_pv` and `switchkvm_pv` functions will be added in `vm.c`, loading a guest's page directory and kernel page table.

The xv6-pv scheduler will also need to save and load the child processes' trapframes and page directories.


## 6. Devices and I/O

Because xv6-pv is running in user mode, it needs to interact with devices throught the host xv6 kernel.

Each device will have a corresponding virtual driver traps I/O operations from the guest to the host kernel through the system call strategy described above, maps xv6-pv's I/O space and operations to those on the host, and provides a consistent device interface.
This will require modifying specific xv6-pv device driver files `ide.c`, `uart.c`, and `console.c` to redirect requests to the host while maintaining the appearance of direct device access.

Disk read/write requests will use shared buffers to transfer data between xv6-pv and the host.

In xv6, the keyboard is handled through interrupts, which places characters into a buffer that processes read.
xv6-pv will have an analogous shared keyboard buffer: when a key is pressed, the host writes the character into the buffer and issues a virtualized interrupt.

The serial line will be implemented similarly: there will be a shared buffer for input and a shared buffer for output.
The guest xv6-pv kernel can poll from the input buffer, and the host xv6 kernel will periodically flush the output buffer through a call to a new `sys_uartwrite_pv` function.
