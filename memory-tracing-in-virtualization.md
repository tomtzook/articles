discuss different techniques and utilization for memory tracing (with page protection mechanisms) for virtualization in x86.

# Memory Tracing in Virtualization

In virtualization, it is often necessary to track the _VM's_ wanted hardware state, without necessarily committing it into actual hardware, as to control the real behavior, but allow the _VM_ to indirectly influence it so that it gets the behavior it is expecting.
In _x86_ architecture, much of this state exists in memory and not special hardware locations. Structures like the _Global Descriptor Table_, _Interrupt Descriptor Table_, _Page Table_ and more all reside within memory, but need to be tracked instead of committed into actual hardware registers (via `lgdt`, `lidt` or _cr3_). At least this is true when no hardware support for virtualization exists. This is no longer true, with the introduction of _Intel's_ and _AMD's_ hardware virtualization extensions which separate the guest and host hardware configuration and does allow the guest to commit those to hardware while still adhering to _VMM_ behavior requirements.

However, one should not discount the possibilities that tracking guest changes to those structures or others may bring. Despite the hardware support, it is still limited to pre-designed capabilities that may not give a _VMM_ all the tools it needs to accomplish a certain task. Nesting is a particular situation where that may be true, with the hardware not actually supporting such a case, and structures like _VMCS_ or _EPT_ (as called in _Intel's_ implementation) which belong to the nested _VMM_ again needing tracking to provide a complete solution. 

This brings us to a technique called _memory tracing_, which was used quite effectively by early software virtualization implementations for _x86_ (namely, those of _VMWare_). With this technique, a _VMM_ can use hardware page protection mechanisms to get "alerts_ when a memory page is read or written to (or even executed). In those early implementations, the technique was applied using the _Page Table_; With hardware extensions with can instead use the _second layer translation mechanisms_ (namely _EPT_) for a similar, but much easier solution.

In this text we will be discussing how to implement this technique in various forms and look at some examples for its possible usage. We will be mainly dealing with _x86_ and _Intel's VT-x_, although the same can be replicated on other hardwares with some modification.

## Preface: The Extended Page Table

_Intel's_ solution for hardware virtualization of the _MMU_ brings another dimension into the translation of virtual address to physical addresses, whereby guest physical addresses are no longer considered the actual physical addresses, but rather a guest-specific abstraction which must be translated through another level - the _EPT_ - to get the physical address to be used. This is, of course, completely transparent to the guest which still uses its _Page Table_ and believes its physical addresses are the ones used to access the memory space. Adding another translation dimension to the _MMU_ does bring an overhead with it, but being implemented in hardware helps alleviate the strain, but more importantly, it doesn't need a constant intervention from the _VMM_ to function. 

Thus, a _VMM_ merely needs to construct, configure and maintain the _EPT_ structure - much like the _OS would do with a _Page Table_. The structure itself is even quite similar to a _Page Table_, using similar terminology and layout.

### Structure

A single _EPT_ is composed out of 4 levels of translation
- The _Page Mapping Level 4_ (_PML4_) is the highest level. There is only one _PML4_, which consists of 512 entries (_PML4e_), each capable of mapping 512 GB of memory. Each entry is 8 bytes long and contains of configuration bits like permissions, and a _Page Frame Number_ pointing at the next level structure.
- A _Page Directory Page Table_ (_PDPT_) contains 512 entries (_PDPTe_), each capable of mapping 1 GB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry can either point at the next level structure, or point directly at the physical address space to represent a single _Huge Page_ of 1 GB.
- A _Page Directory_ (_PD_) contains 512 entries (_PDe_), each capable of mapping 2 MB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry can either point at the next level structure, or point directly at the physical address space to represent a single _Large Page_ of 2 MB.
- A _Page Table_ (_PT_) contains 512 entries (_PTe_), each capable of mapping 4 KB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry points directly at the physical address space to represent a single _Small Page_ of 4 KB.

Some configuration bits that an entry can contain include
- Read, write or execute permission for the memory under this entry. Several combinations are possible, but depend on hardware support (older hardware requires read permission to give write or execute, newer allows without read).
- Accessed and dirty flags can be used if configured so and supported by the hardware.
- Memory Type and PAT control for the memory region.
- Guest Paging control in the memory region.
- And more.



### Violations

## Tracing

discuss:
- different types of tracing (watchpoint, ?)
- granularity of tracing
- continuing execution after trace (resume with emulation, single step, removal or more)
- quick violation handling
- handling multiple cores (per-core ept, sync mechanisms, a middlepoint?)

### Multi-Core Considerations
