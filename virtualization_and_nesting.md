# Virtualization

The concept of creating several _simulated_ and _isolated_ environment on top of a singel hardware was first demonstrated in 1960 by IBM, creating a thin layer of software called _hypervisor_. It promised the ability to
- **Maximize resource utilization** of a single hardware by sharing them between different users.
- **Isolation of the environment**, providing data protection and execution stability (i.e. a failure in one environment will not affect others).

This was meant to be done with a _Virtual Machine Monitor_ (_VMM_ or _hypervisor_) which would control and simulate real hardware to a set of _Virtual Machine_, each thinking it is the only one existing on the hardware.

Although interest waned at first **TODO WHY**, the increase in hardware capacity on a single machine rekindled interest in the concept, especially as a base for the implementation of cloud computing.

_Popek and Goldberg_ in their _Formal Requirements for Virtualizable Third Generation Architectures_ defined that virtualization must provide
- **Equivalence**: a program running under a _VMM_ should behave in an essentially identical manner to one running on real hardware. Any effects on behavior introduced by virtualization would essentially render it non-viable for wide spread use.
- **Safety**: a _VMM_ must have complete control over all virtualized resources, thus protecting the _VM_s from other _VM_s accessing their resources.
- **Efficency**: Most instructions must be executed without _VMM_ intervention, so as to minimize the performance penalty introduced by virtualization.

It can be pretty intuative to compare the architecture of _VMM_ and their relationship with the guest OS as analougues to _kernel_ and _user space_
- A _kernel_ manages the system's hardware and manages multiple _processes_ in the _user space_ which use those resources.
- A _VMM_ similarly manages the hardware and manages multiple _VM_.
- But, whereas the _user process_ is aware of the _kernel_, the _VM_ is typically not aware of the _VMM_.

<img width="739" height="252" alt="image" src="https://github.com/user-attachments/assets/5de57122-2e42-426d-8ab3-7f980a40aba5" />

Essentially then, one can look at virtualization, as a means to multiplex a system to different users. This is what allows cloud computing to function, with a _VMM_ on a single server, running several _VM_ and ensuring they are isolated from one another.

Although recently, it has seen more and more applications for security of a machine, with the realization that the _kernel_ was not actually secure, some companies started pushing security mechanism one level lower, into an _hypervisor_. Microsoft's _Hyper-V_ and _Virtualization Based Security_ are prime examples.

## Virtualization Techniques

If following _Popek and Goldberg_ Theorem 1, we can define an architecture as virtualizable if
- Most instructions, labeled as **non-sensetive** (i.e. no direct hardware resource configuration, like `add` or `mov`) will execute directly on the processor.
- Instructions interacting with the hardware, labeled as **sensetive** (e.g. `lgdt`) are all a sub set of **privileged** instructions.
- **Privileged** instructions cause a **trap** (e.g. an interrupt).

This leads to the _trap-and-emulate_ approach (also called _classic virtualization_) 
- When **non-sensetive** instructions execute in the _VM_, they cause a **trap**. This is because they are **privileged** instructions executed in a non previleged environment.
- This **trap** leads to the _VMM_, which emulates the instruction and returns to the guest.

We would call this _Hardware-Assisted Virtualization_, since it depends on the hardware providing the **trap** via its architectural behavior.

An example for this would be the `lgdt` instruction, part of the x86 architecture. This instruction seeks to load a new _Global Descriptor Table_ into the processor. We would consider this instruction as **sensetive**, as it affects the behavior of memory access. In reality, we may not want the provided _GDT_ to be the real one. Luckily, `lgdt` is a privileged instruction, so its execution by the guest _VM_ will yield an _interrupt_ (`#GP(0)`). Our _VMM_ will receive this interrupt and choose what to really load.

However, not all _ISA_ fit these requirements. _x86_ is actually at fault as well. It features several **sensetive** instructions which are not **privileged**. One of these is `sgdt`, which reads the _GDT_. Considering the fact the we chose what _GDT_ to place when doing `lgdt`, the call to `sgdt` will allow the user to see the _real GDT_.
