# Memory Tracing in Virtualization

In virtualization, it is often necessary to track the _VM's_ wanted hardware state, without necessarily committing it into actual hardware, 
as to control the real behavior, 
but allow the _VM_ to indirectly influence it so that it gets the behavior it is expecting.
In _x86_ architecture, much of this state exists in memory and not special hardware locations. Structures like the _Global Descriptor Table_, 
_Interrupt Descriptor Table_, _Page Table_ and more all reside within memory, but need to be tracked instead of committed into actual hardware registers (via `lgdt`, `lidt` or _cr3_). 
At least this is true when no hardware support for virtualization exists. This is no longer true, with the introduction of _Intel's_ and _AMD's_ hardware virtualization extensions which separate the guest and host hardware configuration and does allow the guest to commit those to hardware while still adhering to _VMM_ behavior requirements.

However, one should not discount the possibilities that tracking guest changes to those structures or others may bring. Despite the hardware support, it is still limited to pre-designed capabilities that may not give a _VMM_ all the tools it needs to accomplish a certain task. Nesting is a particular situation where that may be true, 
with the hardware not actually supporting such a case, and structures like _VMCS_ or _EPT_ (as called in _Intel's_ implementation) which belong to the nested _VMM_ again needing tracking to provide a complete solution. 

This brings us to a technique called _memory tracing_, which was used quite effectively by early software virtualization implementations for _x86_ (namely, those of _VMWare_). With this technique, a _VMM_ can use hardware page protection mechanisms to get "alerts" 
when a memory page is read or written to (or even executed). In those early implementations, the technique was applied using the _Page Table_; With hardware extensions with can instead use the _second layer translation mechanisms_ (namely _EPT_) for a similar, but much easier solution.

In this text we will be discussing how to implement this technique in various forms and look at some examples for its possible usage. We will be mainly dealing with _x86_ and _Intel's VT-x_, although the same can be replicated on other hardwares with some modification.

## EPT-Based Tracing

### Preface: The Extended Page Table

_Intel's_ solution for hardware virtualization of the _MMU_ brings another dimension into the translation of virtual address to physical addresses, whereby guest physical addresses are no longer considered the actual physical addresses, but rather a guest-specific abstraction 
which must be translated through another level - the _EPT_ - to get the physical address to be used. This is, of course, completely transparent to the guest which still uses its _Page Table_ and believes its physical addresses are the ones used to access the memory space. Adding another translation dimension to the _MMU_ does bring an overhead with it, 
but being implemented in hardware helps alleviate the strain, but more importantly, it doesn't need a constant intervention from the _VMM_ to function. 

Thus, a _VMM_ merely needs to construct, configure and maintain the _EPT_ structure - much like the _OS would do with a _Page Table_. The structure itself is even quite similar to a _Page Table_, using similar terminology and layout.

For a complete understanding on structure, use and behavior, see _Intel Software Development Manual Vol. 3C "VMX Support for Address Translation"_.

#### Structure

A single _EPT_ is composed out of 4 levels of translation
- The _Page Mapping Level 4_ (_PML4_) is the highest level. There is only one _PML4_, which consists of 512 entries (_PML4e_), each capable of mapping 512 GB of memory. Each entry is 8 bytes long and contains configuration bits like permissions, and a _Page Frame Number_ pointing at the next level structure.
- A _Page Directory Pointer Table_ (_PDPT_) contains 512 entries (_PDPTe_), each capable of mapping 1 GB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry can either point at the next level structure, or point directly at the physical address space to represent a single _Huge Page_ of 1 GB.
- A _Page Directory_ (_PD_) contains 512 entries (_PDe_), each capable of mapping 2 MB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry can either point at the next level structure, or point directly at the physical address space to represent a single _Large Page_ of 2 MB.
- A _Page Table_ (_PT_) contains 512 entries (_PTe_), each capable of mapping 4 KB of memory. Each entry is 8 bytes long and structured similarly to the other levels. Such entry points directly at the physical address space to represent a single _Small Page_ of 4 KB.

Some configuration bits that an entry can contain include
- Read, write or execute permission for the memory under this entry. Several combinations are possible, but depend on hardware support (older hardware requires read permission to give write or execute, newer allows without read).
- Accessed and dirty flags can be used if configured so and supported by the hardware.
- Memory Type and PAT control for the memory region.
- Guest Paging control in the memory region.
- And more.

#### Violations

Like _Page Tables_ and _Page Faults_, _EPT_ can trigger _EPT Violations_ exits into the _VMM_ when the guest attempts a memory access which violates the configuration bits, discovered in the _MMU_ when translating the address (or through the _TLB_). There are 3 basic reasons for such violations (not including subtle implementation details and more specialized violations like shadow-stack).
- Read attempt to a page with the read bit cleared in any of the entries in the hierarchy of translation.
- Write attempt to a page with the write bit cleared in any of the entries in the hierarchy of translation.
- Execute attempt to a page with the execute bit cleared in any of the entries in the hierarchy of translation.

### Tracing

discuss:
- different types of tracing (watchpoint, ?)
- granularity of tracing
- continuing execution after trace (resume with emulation, single step, removal, redirection/trampoline or more)
- quick violation handling
- handling multiple cores (per-core ept, sync mechanisms, a middlepoint?)

For tracing to be possible, we must have some sort of indication, a trap, to the action we seek to trace. This could be a _vmexit_, giving the _VMM_ running context to address this, but other options are possible:
- An _exit_ into root.
- A hook or trampoline to code in the guest.
- If response isn't needed on the event, then recording it for later observation opens more options. There are many mechanisms that may be of use for such cases: page modification logging, accessed and dirty flags, perhaps event _Intel PT_.

#### Watchpoints

For basic tracing conditions like on memory read, write or execute, it is possible to leverage the protection bits of _EPT_ entries, making it trigger a violation which will cause an exit. 
Utilizing the hardware as the trigger for our trace makes the job much much simpler - no memory need be modified to achieve the most basic traps. We will refer to these as _Watchpoints_ 
triggered based on permission condition.

The smallest region of memory that can be controller is a single _Small Page_ of 4k bytes. This inherently means that any permission changes to install the trace would affect the entire page. Traces that encompass a whole page can easily work with this, 
but anything with a finer granularity becomes difficult, though not impossible.

When a smaller part of a page is the target of a trace, altering the permissions for the entire page is unavoidable using the _EPT_ approach. Any violations that occur for the trace region can be handled as normal, 
but violations from the rest of the page could also occur. In such cases, it is possible to simply not trigger the trace handling and invoke the continuation logic (see "Continuing After Exiting Trace"). It unfortunately means that 
we can expect a drop in performance caused by the exits triggered on the rest of the page, dictated by the amount of access to the page, but functionally it would work.

_Watchpoints_ can be incredibly useful, not unlike setting something similar with a debugger, but driven by the _EPT_. Among the most useful cases include catching read or write to particular memory (say, _Page Table_). 
Using _execute_ watch is also possible but has severe limitations due to granularity.

#### Hooks

#### Trampolines

#### Continuing After Exiting Trace

After receive the trace exit and handling it as wanted, the core must proceed with execution. Resuming the guest simply executes the instruction again, 
causing the same violation again.

Emulation of the faulting instructions in the _VMM_ is possible after handling the trace trigger, eventually resuming into the guest at the next instructions. Emulation is slow however, and can be difficult to implement given the complexity of the instruction set. 
But it does solve the issue, since the _VMM_ is not affected by the permissions in the _EPT_ the permissions do not need to be changed.

Permissions could also be given back to the page after the trace. Resuming the guest at the instruction will execute it again and succeed. This is simpler and faster than emulation, but has a big issue: for the trace to continue functioning, 
permissions must be revoked again. Without some trigger that returns to the _VMM_, it cannot do this change. Several options stand here
- Keep the permissions, essentially removing the trace. Single use traces can be enough in some cases, but it is not very useful.
- Remove the permissions on the next _vmexit_ (any exit for any cause), essentially removing the trace temporarily. Although simple, the loss of tracking for an uncontrolled period of time renders the tracing not that effective.
- Set the _Monitor Trap Flag_ setting before executing the instruction again. After the instruction an _MTF_ exit will be triggered, giving us the context we need to remove the permissions again. Though this both has a performance cost due to the additional exit; Which is minor, given, but on time-sensitive or highly used guest code this may be unacceptable.
- Adding permissions for a period and revoking is problematic in a multi-core machine sharing the _EPT_ between cores. But this will be discussed later.

#### Multi-Core Considerations

## Tracer Design

**use flow diagrams and discuss a recommended option for a tracer design based on the earlier section**
