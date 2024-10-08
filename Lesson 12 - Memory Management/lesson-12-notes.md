# Lesson 12: Memory Management

- Topics to be covered in this lesson:
  - Physical and virtual memory management
  - Review of memory management mechanisms
  - Illustration of advanced services

## Visual Metaphor

- **OS** and **toy shops** each have **memory (part) management systems**:
  - Uses intelligently sized containers:
    - Memory pages or segments
  - Not all memory is needed at once:
    - Tasks operate on subset of memory
  - Optimized for performance:
    - Reduce time to access state in memory (better performance)

## Memory Management: Goals

- Virtual vs physical memory:
  - Allocate: Allocation, replacement, etc.
  - Arbitrate: Address translation and validation
- Page-based memory management:
  - Allocate: Pages to page frames
  - Arbitrate: Page tables
- Segment-based memory management:
  - Allocate: Segments
  - Arbitrate: segment registers

## Memory Management: Hardware Support

- MMU (memory management unit):
  - Translate **VA (virtual address)** to **PA (physical addresses)**
  - Reports faults: illegal access, permission not present in memory
- Registers:
  - Pointers to page table
  - Base and limit size, number of segments, etc.
- Cache - TLB (translation look-aside buffer):
  - Valid virtual address to physical address translations: TLB
- Translation:
  - Actual PA generation done in hardware

## Page Tables

- Virtual memory pages and physical memory page frames are the same size to keep track of the translation of every single 
individual virtual address
- Useful acronyms for page tables:
  - VPN (virtual page number)
  - PFN (physical frame number)
- Page table has allocation on first touch physical memory for the array in the virtual address space is only 
allocated when the process is first trying to access it!
- Unused pages are reclaimed:
  - Mapping invalid
  - Hardware will fault
  - Re-establish on re-access
- In summary, the OS creates a page table for every process:
  - On context switch, switch to valid page table
  - Update register

## Page Table Entry

- Flags:
  - Present (valid/invalid)
  - Dirty (written to)
  - Accessed (for read or write)
  - Protection bits (read, write, and execute)
- **Page fault**:
  - Two options for PFN:
    - Generate physical address and access
    - Generate error code on kernel stack (trap into kernel)
  - Page fault handler determines action based on error code and faulting address:
    - Bring page from disk to memory
    - Protection error (`SIGSEGV`)
    - Error code from PTE flags
    - Faulting address CR2 register

## Page Table Size

- **64-bit Architecture**:
  - PTE (Page Table Entry): 8 bytes including PFN + flags
  - VPN: 2^64 / page size
  - Page size: (2^64 / 2^12) \* 8 bytes = 32 petabytes per process
- **Process** does not use entire address space:
  - Even on 32-bit architecture will not always use all of 4 GB
  - But **page table** assumes an entry per **VPN**, regardless of whether corresponding virtual memory is needed or not

## Multi-level Page Tables

- **Outer page table**: page table directory
- **Internal page table**: only for valid virtual memory regions
- **Additional layers**:
  - Page table directory pointer (third level)
  - Page table directory point map (fourth level)
    - Important on 64-bit architectures
    - Larger and more sparse
    - Larger gaps could save more internal page table components
- **Multi-level page tables**:
  - Pros:
    - Smaller internal page tables/directories
    - Granularity of coverage (potential reduced page table size)
  - Cons:
    - More memory access required for translation (increased translation latency)

## Speeding Up Translation TLB

- Overhead of address translation:
  - Single-level page table:
    - 1x access to page table entry
    - 1x access to memory
  - Four-level page table:
    - 4x accesses to page table entries
    - 1x access to memory (can lead to slow down!)
- **Page table cache (TLB)**:
  - MMU-level address translation cache
  - On TLB miss: page table access from memory
  - Has protection/validity bits
  - Small number of cached address, high TLB hit rate and temporal and spatial locality

## Inverted Page Tables

- Another way of organizing the address translation process (see lecture for the inverted page table diagram):
  - Components:
    - Logical address
    - Physical address
    - Physical memory
    - Page table
    - Search
  - TLB to catch memory references
- Inverted page tables use hashing page tables (see lecture for diagram) to optimize efficiency:
  - Speeds up linear search process and narrows it down to few possible entries into the inverted page table, this speeds up address translation

## Segmentation

- **Segments** are arbitrary and granular:
  - E.g., code, heap, data, stack, etc.
  - Address is equivalent to the segment selector + offset
- **Segment** is a contiguous physical memory:
  - Segment size is equivalent to segment base + limit registers
- **Segmentation + paging**:
  - IA x86_32: segmentation + paging
    - Linux up to 8K per process / global segment
  - IA 86x_64: paging

## Page Size

- 10-bit offset: 1 KB page size
- 12-bit offset: 4 KB page size
- Below is a table detailing _large vs huge_ pages

|                                       | Large   | Huge    |
| ------------------------------------- | ------- | ------- |
| Page size                             | 2 MB    | 1 GB    |
| Offset bits                           | 21 bits | 30 bits |
| Reduction factor (on page table size) | x512    | x1024   |

- In general, for larger pages:
  - Pros: fewer page table entries, smaller page tables, more TLB hits, etc.
  - Cons: internal fragmentation (wastes memory)

## Memory Allocation

- **Memory allocator**:
  - Determines VA to PA mapping, address translation, page tables, etc.
  - Simply determine PA from VA and check validity/permissions
- **Kernel-level allocators**:
  - Kernel state, static process state
- **User-level allocators**:
  - Dynamic process state (heap), malloc/free
  - E.g., `dimalloc`, `jemalloc`, `hoard`, `tcmalloc`

## Memory Allocation Challenges

- Problem: **external fragmentation**
  - Occurs with multiple interleaved allocate and free operations, and as a result of them, we have holes of free memory that is not contiguous
  - Requests for larger contiguous memory allocations cannot be satisfied
- Solution:
  - When pages are freed, the allocator can aggregate adjacent areas of free pages into one larger free area, this allows for larger future requests

## Allocators in the Linux Kernel

- The Linux kernel relies on two basic allocation mechanisms:
  - **Buddy**:
    - Starts with consecutive memory region that's free (2^x area)
    - On request, sub-divide into 2^x chunks and find smallest 2^x chunk that can satisfy request (fragmentation still there)
    - On free:
      - Check buddy to see if it can aggregate into a larger chunk
      - Aggregate more up the tree (aggregation works well and fast)
  - **Slab**:
    - Addresses 2^x granularity in Buddy
    - Addresses internal fragmentation
    - **Slab allocator**:
      - Caches for common object types/sizes, on top of contiguous memory
    - Pros:
      - Internal fragmentation avoided
      - External fragmentation not an issue

## Demand Paging

- Virtual memory >> physical memory:
  - Virtual memory page note always in physical memory
  - Physical page frame saved and restored to/from secondary storage
- **Demand paging**: pages swapped in/out of memory and a swap partition
  - Original PA is not equal to PA after swap
  - If page is _pinned_ swapping disabled

## Freeing Up Physical Memory

- When should pages be swapped out?
  - OS runs page (out) daemon:
    - When memory usage is above threshold (high watermark)
    - When CPU usage is below threshold (low watermark)
- Which pages should be swapped out?
  - Pages that won't be used
  - History-based prediction:
    - LRU (least-recently used) policy: access bit to track if page is referenced
  - Pages that don't need to be written out
    - Dirty bit to track if modified
  - Avoid non-swappable pages
- In Linux:
  - Parameters to tune thresholds: target page count, etc.
  - Categorize pages into different types: e.g., claimable, swappable, etc.
  - _Second chance_ variation of LRU

## Copy On Write

- MMU Hardware:
  - Perform translation, track access, enforce protection, etc.
  - Useful to build other services and optimizations
- **COW (copy-on-write)**:
  - On process creation:
    - Copy entire parent address space
    - Many pages are static, don't change (why keep multiple copies?)
  - On create:
    - Map new VA to original page
    - Write protect original page
    - If only read: save memory and time to copy
  - On write:
    - Page fault copy
    - Pay copy cost only if necessary

## Failure Management Check-pointing

- **Check-pointing**: failure and recovery management technique
  - Periodically save process state
  - Failure may be unavoidable but can restart from checkpoint so recovery much faster
- **Simple approach**: pause and copy
- **Better approach**:
  - Write-protect and copy everything once
  - Copy diffs of _dirtied_ pages for incremental checkpoints, rebuild from mutiple diffs, or in background
- **Debugging**:
  - RR (rewind-replay)
  - Rewind means to restart from checkpoint
  - Gradually go back to older checkpoints until error found
- **Migration**:
  - Continue on another machine
  - Disaster recovery
  - Consolidation
  - Repeated checkpoints in a fast loop until pause-and-copy becomes acceptable (or unavoidable)
