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

It can be pretty intuative to compare the architecture of _VMM_s and their relationship with the guest OS as analougues to _kernel_ and _user space_
- A _kernel_ manages the system's hardware and manages multiple _processes_ in the _user space_ which use those resources.
- A _VMM_ similarly manages the hardware and manages multiple _VM_s.
- But, whereas the _user process_ is aware of the _kernel_, the _VM_ is typically not aware of the _VMM_.

Essentially then, one can look at virtualization, as a means to multiplex a system to different users. This is what allows cloud computing to function, with a _VMM_ on a single server, running several _VM_s and ensuring they are isolated from one another.

Although recently, it has seen more and more applications for security of a machine, with the realization that the _kernel_ was not actually secure, some companies started pushing security mechanism one level lower, into an _hypervisor_. Microsoft's _Hyper-V_ and _Virtualization Based Security_ are prime examples.

## Virtualization Techniques
