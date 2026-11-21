# Virtualization

The concept of creating several _simulated_ and _isolated_ environment on top of a singel hardware was first demonstrated in 1960 by IBM, creating a thin layer of software called _hypervisor_. It promised the ability to
- **Maximize resource utilization** of a single hardware by sharing them between different users.
- **Isolation of the environment**, providing data protection and execution stability (i.e. a failure in one environment will not affect others).

This was meant to be done with a _Virtual Machine Monitor_ (_VMM_ or _hypervisor_) which would control and simulate real hardware to a set of _Virtual Machine_, each thinking it is the only one existing on the hardware.

Although interest waned at first, but the increase in hardware capacity on a single machine rekindled interest in the concept, especially as a base for the implementation of cloud computing.

_Popek and Goldberg_ in their _Formal Requirements for Virtualizable Third Generation Architectures_ defined that virtualization must provide
- **Equivalence**: a program running under a _VMM_ should behave in an essentially identical manner to one running on real hardware. Any effects on behavior introduced by virtualization would essentially render it non-viable for wide spread use.
- **Safety**: a _VMM_ must have complete control over all virtualized resources, thus protecting the _VM_s from other _VM_s accessing their resources.
- **Efficency**: Most instructions must be executed without _VMM_ intervention, so as to minimize the performance penalty introduced by virtualization.

It can be pretty intuative to compare the architecture of _VMM_ and their relationship with the guest OS as analougues to _kernel_ and _user space_
- A _kernel_ manages the system's hardware and manages multiple _processes_ in the _user space_ which use those resources.
- A _VMM_ similarly manages the hardware and manages multiple _VM_.
- But, whereas the _user process_ is aware of the _kernel_, the _VM_ is typically not aware of the _VMM_.

<img width="739" height="252" alt="image" src="https://github.com/user-attachments/assets/5de57122-2e42-426d-8ab3-7f980a40aba5" />

Essentially then, one can look at virtualization, as a means to multiplex a system to different users. This is what allows cloud computing to function, with a _VMM_ on a single server, running several _VM_ and ensuring they are isolated from one another. It is thus paramount for the _VMM_ to ensure that the virtualized hardware acts in a manner fitting the expectation of the _VM_. 

Although recently, it has seen more and more applications for security of a machine, with the realization that the _kernel_ was not actually secure, some companies started pushing security mechanism one level lower, into an _hypervisor_. Microsoft's _Hyper-V_ and _Virtualization Based Security_ are prime examples.

## Virtualization Techniques

### Hardware-Assisted Virtualization

If following _Popek and Goldberg_ Theorem 1, we can define an architecture as virtualizable if
- Most instructions, labeled as **non-sensetive** (i.e. no direct hardware resource configuration, like `add` or `mov`) will execute directly on the processor.
- Instructions interacting with the hardware, labeled as **sensetive** (e.g. `lgdt`) are all a sub set of **privileged** instructions.
- **Privileged** instructions cause a **trap** (e.g. an interrupt).

This leads to the _trap-and-emulate_ approach (also called _classic virtualization_) 
- When **sensetive** instructions execute in the _VM_, they cause a **trap**. This is because they are **privileged** instructions executed in a non previleged environment.
- This **trap** leads to the _VMM_, which emulates the instruction and returns to the guest.
- **non-sensetive** instructions are just executed by the hardware as is.

We would call this _Hardware-Assisted Virtualization_, since it depends on the hardware providing the **trap** via its architectural behavior.

An example for this would be the `lgdt` instruction, part of the x86 architecture. This instruction seeks to load a new _Global Descriptor Table_ into the processor. We would consider this instruction as **sensetive**, as it affects the behavior of memory access. In reality, we may not want the provided _GDT_ to be the real one. Luckily, `lgdt` is a privileged instruction, so its execution by the guest _VM_ will yield an _interrupt_ (`#GP(0)`). Our _VMM_ will receive this interrupt and choose what to really load.

From this example, one could understand that there may be a need for shadow structures. These structures are derived from the guest's structures (like _Page Tables_ and _Global Descriptor Table_). The guest may seek a specific memory mapping with physical memory, but because of the presence of more _VM_, the _VMM_ cannot give it free reign in mapping the physical memory. This would then require the _VMM_ to intercept the guest _Page Table_ and load another one in reality that can service all the present _VM_. This requires use of hardware page protection mechanism. This can be used to trap specific memory accesses that require emulation. Among these are _memory-mapped IO_ operations and hardware structure modification. When the guest modifies its _Page Table_, the _VMM_ must react appropriately to provide the guest with the expected virtual addresses. It would thus have to use page protection to force traps when the guest modifies its _Page Table_.

### Software Virtualization

However, not all _ISA_ fit these requirements. _x86_ is actually at fault as well. It features several **sensetive** instructions which are not **privileged**. One of these is `sgdt`, which reads the _GDT_. Considering the fact the we chose what _GDT_ to place when doing `lgdt`, the call to `sgdt` will allow the user to see the _real GDT_ instead of the one it thinks it loaded. This is a functional problem at its core, since the OS will expect a very specific _GDT_ and may not handle it properly when a different _GDT_ is retreived, breaking the **Equivalence** requirement.

Altering our approach, we could use an _interpreter_ to translate and catch all **sensetive** instructions and replacing them with a trap, or shadow structure when necessary. An example for this was done by _VMWare_, where they used _binary translation_ to translate the guest code so as to enable proper virtualization with x86. We would refer to this as _Software Virtualization_.

We could tackle `sgdt` as an example. Since this is a **sensetive** but non-privileged instruction, we must address it manually to acheive proper virtualization. When cosidering a translation approach, we may translate `sgdt eax` instruction into `mov eax, 0x0ff123`, where `0x0ff123` is the address of the "fake" _GDTR_ the guest is expecting. Thus the guest received the information it expected.

### Intel VT-x

To address the problems with virtualization in _x86_ both _Intel_ and _AMD_ released extensions to the architectures to fascilitate full hardware-assisted virtualization. For _Intel_, this came in the form of _VT-x_.

When in virtualization mode, the x86 processor as two new operating modes:
- **root**: the _VMM_, esentially the new **privileged** mode.
- **non-root**: the _guest_.

When in **non-root** mode, certain instructions now trigger special virtualization behavior. This can come in the form of a **trap** (called **vmexit**), or a unique hardware behavior emulating real hardware behavior. Unlike previous _trap-and-emulate_ in x86, the hardware takes much more of the load, and what is considered **sensetive** has changed. 

The processor uses the _Virtual Machine Control Structure_ to allow the _VMM_ to define virtualization behavior with the guest. Providing more find-grained control, while leveraging new hardware circuits to limit the need for the _VMM_ to perform emulation manually.

One example for this is the new _Extended Page Tables_ added with the _Nehalem_ microarchitecture. This is an extension to the standard _Page Table_ and is integrated directly with the _MMU_. _VMM_ can use this to define the true virtual-physical memory mapping for each _guest_, removing the need to trace _Page Table_ modification in the guest. 

Thus, virtualization is done in two ways:
- hardware assisted emulation, including things like APIC and IO virtualization. These are configured in the _VMCS_ by the _VMM_ before executing the guest.
- software emulation, in response to **vmexit**, essentially specific **trap** triggered when _VT-x_ is unable to emulate on its own with hardware (or specifically configured not to do so). Including things like _MSR_ and _CR_ access.

The following is the list of all **vmexit** reasons:
```
    exception_or_nmi = 0,
    external_interrupt = 1,
    triple_fault = 2,
    init_signal = 3,
    startup_ipi = 4,
    io_smm_interrupt = 5,
    other_smi = 6,
    interrupt_window = 7,
    nmi_windows = 8,
    task_switch = 9,
    cpuid = 10,
    getsec = 11,
    hlt = 12,
    invd = 13,
    invlpg = 14,
    rdpmc = 15,
    rdtsc = 16,
    rsm = 17,
    vmcall = 18,
    vmclear = 19,
    vmlaunch = 20,
    vmptrld = 21,
    vmptrst = 22,
    vmread = 23,
    vmresume = 24,
    vmwrite = 25,
    vmxoff = 26,
    vmxon = 27,
    cr_access = 28,
    mov_dr = 29,
    io = 30,
    rdmsr = 31,
    wrmsr = 32,
    vmentry_invalid_guest_state = 33,
    vmentry_msr_load_failure = 34,
    mwait = 36,
    monitor_trap_flag = 37,
    monitor = 39,
    pause = 40,
    vmentry_machine_check = 41,
    tpr_below_threshold = 43,
    apic_access = 44,
    virtualized_eoi = 45,
    access_gdtr_idtr = 46,
    access_ldtr_tr = 47,
    ept_violation = 48,
    ept_misconfiguration = 49,
    invept = 50,
    rdtscp = 51,
    vmx_preemption_timer_expired = 52,
    invvpid = 53,
    wbinvd = 54,
    xsetbv = 55,
    apic_write = 56,
    rdrand = 57,
    invpcid = 58,
    vmfunc = 59,
    encls = 60,
    rdseed = 61,
    page_mod_log_full = 62,
    xsaves = 63,
    xrstors = 64,
```

The following are the fields in the _VMCS_:
```
    ctrl_virtual_processor_identifier = 0x0,
    ctrl_posted_interrupt_notification_vector = 0x2,
    ctrl_eptp_index = 0x4,
    guest_es_selector = 0x800,
    guest_cs_selector = 0x802,
    guest_ss_selector = 0x804,
    guest_ds_selector = 0x806,
    guest_fs_selector = 0x808,
    guest_gs_selector = 0x80a,
    guest_ldtr_selector = 0x80c,
    guest_tr_selector = 0x80e,
    guest_interrupt_status = 0x810,
    guest_pml_index = 0x812,
    host_es_selector = 0xc00,
    host_cs_selector = 0xc02,
    host_ss_selector = 0xc04,
    host_ds_selector = 0xc06,
    host_fs_selector = 0xc08,
    host_gs_selector = 0xc0a,
    host_tr_selector = 0xc0c,
    ctrl_io_bitmap_a_address = 0x2000,
    ctrl_io_bitmap_b_address = 0x2002,
    ctrl_msr_bitmap_address = 0x2004,
    ctrl_vmexit_msr_store_address = 0x2006,
    ctrl_vmexit_msr_load_address = 0x2008,
    ctrl_vmentry_msr_load_address = 0x200a,
    ctrl_executive_pointer = 0x200c,
    ctrl_pml_address = 0x200e,
    ctrl_tsc_offset = 0x2010,
    ctrl_virtual_apic_address = 0x2012,
    ctrl_apic_access_address = 0x2014,
    ctrl_posted_interrupt_descriptor_address = 0x2016,
    ctrl_vmfunc_controls = 0x2018,
    ctrl_ept_pointer = 0x201a,
    ctrl_eoi_exit_bitmap_0 = 0x201c,
    ctrl_eoi_exit_bitmap_1 = 0x201e,
    ctrl_eoi_exit_bitmap_2 = 0x2020,
    ctrl_eoi_exit_bitmap_3 = 0x2022,
    ctrl_ept_pointer_list_address = 0x2024,
    ctrl_vmread_bitmap_address = 0x2026,
    ctrl_vmwrite_bitmap_address = 0x2028,
    ctrl_virtualization_exception_information_address = 0x202a,
    ctrl_xss_exiting_bitmap = 0x202c,
    ctrl_encls_exiting_bitmap = 0x202e,
    ctrl_tsc_multiplier = 0x2032,
    guest_physical_address = 0x2400,
    guest_link_pointer = 0x2800,
    guest_debugctl = 0x2802,
    guest_pat = 0x2804,
    guest_efer = 0x2806,
    guest_perf_global_ctrl = 0x2808,
    guest_pdpte0 = 0x280a,
    guest_pdpte1 = 0x280c,
    guest_pdpte2 = 0x280e,
    guest_pdpte3 = 0x2810,
    host_pat = 0x2c00,
    host_efer = 0x2c02,
    host_perf_global_ctrl = 0x2c04,
    ctrl_pin_based_vm_execution_controls = 0x4000,
    ctrl_processor_based_vm_execution_controls = 0x4002,
    ctrl_exception_bitmap = 0x4004,
    ctrl_pagefault_error_code_mask = 0x4006,
    ctrl_pagefault_error_code_match = 0x4008,
    ctrl_cr3_target_count = 0x400a,
    ctrl_vmexit_controls = 0x400c,
    ctrl_vmexit_msr_store_count = 0x400e,
    ctrl_vmexit_msr_load_count = 0x4010,
    ctrl_vmentry_controls = 0x4012,
    ctrl_vmentry_msr_load_count = 0x4014,
    ctrl_vmentry_interruption_information_field = 0x4016,
    ctrl_vmentry_exception_error_code = 0x4018,
    ctrl_vmentry_instruction_length = 0x401a,
    ctrl_tpr_threshold = 0x401c,
    ctrl_secondary_processor_based_vm_execution_controls = 0x401e,
    ctrl_ple_gap = 0x4020,
    ctrl_ple_window = 0x4022,
    vm_instruction_error = 0x4400,
    exit_reason = 0x4402,
    vmexit_interruption_information = 0x4404,
    vmexit_interruption_error_code = 0x4406,
    idt_vectoring_information = 0x4408,
    idt_vectoring_error_code = 0x440a,
    vmexit_instruction_length = 0x440c,
    vmexit_instruction_info = 0x440e,
    guest_es_limit = 0x4800,
    guest_cs_limit = 0x4802,
    guest_ss_limit = 0x4804,
    guest_ds_limit = 0x4806,
    guest_fs_limit = 0x4808,
    guest_gs_limit = 0x480a,
    guest_ldtr_limit = 0x480c,
    guest_tr_limit = 0x480e,
    guest_gdtr_limit = 0x4810,
    guest_idtr_limit = 0x4812,
    guest_es_access_rights = 0x4814,
    guest_cs_access_rights = 0x4816,
    guest_ss_access_rights = 0x4818,
    guest_ds_access_rights = 0x481a,
    guest_fs_access_rights = 0x481c,
    guest_gs_access_rights = 0x481e,
    guest_ldtr_access_rights = 0x4820,
    guest_tr_access_rights = 0x4822,
    guest_interruptibility_state = 0x4824,
    guest_activity_state = 0x4826,
    guest_smbase = 0x4828,
    guest_sysenter_cs = 0x482a,
    guest_vmx_preemption_timer_value = 0x482e,
    sysenter_cs = 0x4c00,
    ctrl_cr0_guest_host_mask = 0x6000,
    ctrl_cr4_guest_host_mask = 0x6002,
    ctrl_cr0_read_shadow = 0x6004,
    ctrl_cr4_read_shadow = 0x6006,
    ctrl_cr3_target_value_0 = 0x6008,
    ctrl_cr3_target_value_1 = 0x600a,
    ctrl_cr3_target_value_2 = 0x600c,
    ctrl_cr3_target_value_3 = 0x600e,
    exit_qualification = 0x6400,
    io_rcx = 0x6402,
    io_rsx = 0x6404,
    io_rdi = 0x6406,
    io_rip = 0x6408,
    exit_guest_linear_address = 0x640a,
    guest_cr0 = 0x6800,
    guest_cr3 = 0x6802,
    guest_cr4 = 0x6804,
    guest_es_base = 0x6806,
    guest_cs_base = 0x6808,
    guest_ss_base = 0x680a,
    guest_ds_base = 0x680c,
    guest_fs_base = 0x680e,
    guest_gs_base = 0x6810,
    guest_ldtr_base = 0x6812,
    guest_tr_base = 0x6814,
    guest_gdtr_base = 0x6816,
    guest_idtr_base = 0x6818,
    guest_dr7 = 0x681a,
    guest_rsp = 0x681c,
    guest_rip = 0x681e,
    guest_rflags = 0x6820,
    guest_pending_debug_exceptions = 0x6822,
    guest_sysenter_esp = 0x6824,
    guest_sysenter_eip = 0x6826,
    host_cr0 = 0x6c00,
    host_cr3 = 0x6c02,
    host_cr4 = 0x6c04,
    host_fs_base = 0x6c06,
    host_gs_base = 0x6c08,
    host_tr_base = 0x6c0a,
    host_gdtr_base = 0x6c0c,
    host_idtr_base = 0x6c0e,
    host_sysenter_esp = 0x6c10,
    host_sysenter_eip = 0x6c12,
    host_rsp = 0x6c14,
    host_rip = 0x6c16
```

The following is a simplified demonstration of a _VMM_ lifecycle

<img width="537" height="696" alt="image" src="https://github.com/user-attachments/assets/33a30ff6-b7b3-4e43-9934-d84dad309b78" />

The following shows the processor reaction to a guest trying to read an _MSR_ with _VT-x_

<img width="535" height="706" alt="image" src="https://github.com/user-attachments/assets/1aa0beab-59cb-47e6-b0f9-3093d6422b54" />

## Nesting with Intel VT-x

_Nested Virtualization_ is the ability to virtualize a _VMM_ which itself is virtualizing one or more _VM_, making the _VMM_ to be another _VM_. Just as a _VM_ expects specific behavior from the hardware which the _VMM_ has to emulate, so does a _VMM_ expect specific behavior.

The _virtualized VMM_ itself is actually **unprivileged** in this situation. And although it may _think_ it is running a _VM_, it in fact does not do so directly. The only **privileged** _VMM_ is the one doing the nesting. So in reality both **L1** and **L2** are guests of **L0**. 

<img width="547" height="250" alt="image" src="https://github.com/user-attachments/assets/4e422049-4f5f-46a0-9006-a18bec466a7f" />

Thus, the general method of virtualization remains similar to standard practice. However, to properly virtualize a _VMM_ and its associated _VM_, the hypervisor must take care to emulate the relationship between the two, allowing the nested _VMM_ the illusion of control over the _VMs_, such that its emulation does affect the _VMs_, but still keeping it in **unprivileged** state.

This creates 3 levels of _virtualization_:
- **L0**: the underlying _VMM_, which interacts with the real hardware.
- **L1**: the virtualized _VMM_, thinking it is the real hypervisor.
- **L2**: the guest machines for the virtualized _VMM_. 

<img width="450" height="334" alt="image" src="https://github.com/user-attachments/assets/002b8b10-ae7f-49f6-af90-84bcfa52e5b5" />


For _Intel VT-x_, this exact behavior is described rather precisely in _Intel's SDM Vol 3C_ in the chapters discussing _Virtual Machine Control Structures_, _VM Entries_ and _VM Exits_.

To keep control over the virtualization of **L2** guests, but also emulate the expected VM behavior by **L1**, it is necessary to track the _VMCS_ used by **L1**. However, keeping it **unpriviledged** requires that it will not directly touch the _real VMCS_. It is necessary, then, to create a _shadow VMCS_ for it, and track any changes it makes to it.
_VT-x_ actually helps us in that regard, as executing _VMX_ instruction in a guest causes _vmexit_ with the relevant informstion. This gives us the perfect opportunity to emulate the relavent instructions to our liking. In the case of _VMCS_ instructions (_vmread_, _vmwrite_, _vmptrld_, _vmptrst_, _vmclear_) we can perform the emulation by working with our _shadow VMCS_, keeping the real one untouched.

Because our _VMM_ is the only real _VMM_ (hardware wise), it will have to be the one actually running all the _VMs_. But in order to provide **L1** with the emulation of virtualization, with must consider its wanted behavior. This is where the _shadow VMCS_ comes into action. The _VMCS_ actually loaded when running **L2** is still ours, however it must be constructed based on both the _shadow VMCS_ (what **L1** wants) and our needs. This in turn makes for 3 _VMCS_:
- The _VMCS_ we use to run **L1** - **vmcs01**
- The _shadow VMCS_ - **vmcs12**
- The _VMCS_ used to run **L2** - **vmcs02**; Built out of **vmcs12**.

Aside from the _VMCS_, nesting requires us to take care of two additional parts. First we must take care of emulating any behavior defined by the _VMCS_. Normally, if properly configured, this will be done by the hardware. But some instances will require our intervention. Among those are directing _vmexits_ into **L1** - as all exits actually go to us. Additinally, we must also emulate all _VMX_ instructions, not only those that deal with _VMCS_.

### Flow

Nesting flow is quite similar to how we run our own _VMM_, but looking from the perspective of hardware. It all surrounds the use of _VMX_ instructions.

Recall the steps taking to initialize _VMM_:
- Enter _VMX-root_ with _vmxon_.
- Create _VMCS_ with _vmptrld_, _vmclear_ and _vmwrite_.
- Launch first guest with _vmlaunch_.

From there, the hypervisor switches to a reactive flow, receiving and reaponding to _vmexits_, which typically include:
- Entry through the _VMCS_ defined _vmexit_ handler and saving guest registers.
- Querying information on the exit from _VMCS_ with _vmread_.
- Handling the exit, which typically include modifications to the _VMCS_ with _vmwrite_, or editing of the guest registers.
- Returning to the guest with restoration of guest registers and _vmresume_.

All of these operations, which we can expect from **L1**, can be handled by properly emulating the relevant instructions and host<->guest transitions.

This is generally the entirety of the flow. We must expect the switching of _VMCS_ and thus _VMs_ from **L1**, or the creation of new guests. Although all of those operations are done with _VMX_ instructions.

For our _VMM_, nesting does not exist until the first _vmxon_ from the guest. Until then, we're just a normal hypervisor dealing with our own _VMs_.

The following illustrates the process of handling the startup and creation of **L2** by **L1**

<img width="1052" height="743" alt="image" src="https://github.com/user-attachments/assets/3ddf2ead-bf4f-4c94-ae8c-9eb45b8434ab" />

### VM Exits From L2

Because our hypervisor is the actual host for **L2**, all _vmexits_ go through us. Since **L1** is the one meant to control **L2**, we must transfer atleast some of those into **L1**. This would require proper emulation of _vmexit_ behavior as defined by the architecture.

Other than that, it is necessary to first determine which exits must be handled from **L0** and which from **L1** This is very dependent on how _vmcs02_ and _vmcs12_ are configured; Something decided by what kind of behavior each _VMM_ wants. 
There is no one correct approach here, but it is generally advised to start by letting **L1** determine everything (where _vmcs02_ is almost identical to _vmcs12_) and handle everything (where all exits are transferred to **L1** for handling). From there, changes can made to give **L2** more control as needed (by both not directing all exits, and setting _vmcs02_ differently from _vmcs12_).

The following illustrates the flow for handling _vmexits_ from **L2** with **L1**

<img width="1138" height="484" alt="image" src="https://github.com/user-attachments/assets/3b587a1f-d4d3-42d5-8bc0-6e791c41e239" />

The following illustrates flow for handling an exiting _rdmsr_ with **L1**

<img width="1610" height="710" alt="image" src="https://github.com/user-attachments/assets/f98ccbae-2bed-4c58-b67b-b69b7d1d29cd" />

Regardless, once it is decided to direct the exit into **L1**, it becomes necessary to emulate exit behavior into **L1**. Here, we must
- Edit _vmcs01_ to reflect expected state of registers, msrs or others, in accordance with expected _vmexit_ behavior. Also involves infromation from _vmcs12's_ host state which should be configured in _vmcs01's_ guest state.
- Edit _vmcs12's_ guest state and _vmexit_ fields with information about the exit, taken mostly from _vmcs02_.
- Restore guest registers saved from **L2** to be collected by **L1** (as some registers are transferred between guest and host on transition)
- And _vmresume_ into **L1**. 

The exact behavior that must be replicated is defined in _Intel SDM Vol. 3C Chapter "VM Exit"_. 

**GO OVER SOME OF THE THINGS THAT MUST BE DONE**

### VM Entries Into L2

In the other direction, a _vmlaunch_ or _vmresume_ from **L1** would require the entry into **L2**. Unlike with _vmexits_ though, there is no need to emulate the architectual behavior of _vmentry_, as **L0** needs to perform this itself to enter **L2** in any case.

Instead, work here focuses on the proper configuration of _vmcs02_, based on changes made to _vmcs12_ as well as desired behavior by **L0**. Reconciling the two is an open question that can be approached in several ways. A good starting point is
- _vmcs02_ guest state = _vmcs12_ guest state.
- _vmcs02_ host state = _vmcs01_ host state (if the same host entry is wanted as with **L1**).
- _vmcs02_ control fields = mostly _vmcs12_ control fields, but slightly modified if a different behavior is desired.

Once _vmcs02_ is ready, a simple _vmresume_ or _vmlaunch_ is enough to finish the transition. Do remember to restore the guest register values **as they were** in the exit from **L1**.

The exact behavior that must be replicated is defined in _Intel SDM Vol. 3C Chapter "VM Entry"_.

**GO OVER SOME OF THE FIELDS THAT NEED WORK**.

### Emulating VMX Instructions

Part of a good nesting implementation lies in the proper emulation of the _vmx_ instructions an **L1** would have to use. This does not mean invocation of the real instructions, as the only real _VMM_ is supposed to be **L0**, but rather the illusion of that through an _equivalent_ behavior. **TODO: MORE ON THIS**

The following was taken from _Intel SDM Vol. 3C Chapter "VMX Instruction Reference"_ and illustrates the exact behavior of _vmxon_.

<img width="1003" height="749" alt="image" src="https://github.com/user-attachments/assets/62d54f60-9b83-499c-894e-9a7181bb40a6" />

When emulating this instruction, we seek to create an almost identical behavior, which can be translated to the following flow

<img width="876" height="743" alt="image" src="https://github.com/user-attachments/assets/be31b40e-0008-44ba-939b-43fc7e367319" />

Only that instead of really doing _vmxon_ (because it was already done by **L0**) we merely saved information and set ourselves up to the incoming execution of **L1** and eventually **L2**. The correctness checks, although tedious, must not be ignored for true transparency. It is even possible for some _VMM_ implementations to actually test this correct behavior - e.g. trying to do _vmread_ on a field and expecting failure if the field is actually not readable.

Let us look at _vmptrld_ as another example

<img width="869" height="380" alt="image" src="https://github.com/user-attachments/assets/7765cfad-28ff-4cd7-8fa2-5819dff47fc5" />

with the flow being

<img width="994" height="727" alt="image" src="https://github.com/user-attachments/assets/31ac12ce-fe0f-40fd-9999-7fb3b26564d4" />

And _vmwrite_

<img width="1035" height="470" alt="image" src="https://github.com/user-attachments/assets/d1a172cc-b993-413e-ba08-782ef36460cb" />

flow 

<img width="869" height="721" alt="image" src="https://github.com/user-attachments/assets/e5232682-3a8f-4cca-9e7d-99cc119cc206" />

One can quickly understand that because so many actions revolve around these instructions, we can do a lot of the nesting work by handling them properly.

### Multi-Core Considerations 

With the advent of more cores in processors, it is impossible to ignore the complexities of handling all these cores. Luckily however, _VMX_ is handled separately on each core, with each having its own state, _VMCS_, execution flow and such. A bare-bones _VMM_, can then handle each core on its own without much regards to the others. As long as the states and operations made affect only the current core, no manner if synchronization is necessary.

Moreover, when it comes to nesting, any manner of cross processor interaction - namely external interrupts, can be left to **L1** to handle (for the most part). **L0** still needs to address such things when occuring **in L1**, but can defer handling when it comes to **L2**. When considering **equivelence**, doing so is actually preferred - to maintain transparency to **L1**.

As such, other the. having to maintain several per-core structures (like _VMCS_), and handling each core independently, multi-core may not be felt much. Only when **L0** introduces shared structures and operations, does synchronization need to be addressed. How is very much dependent on the exact nature of operation, structure or its usage.

### Multiple L2 VMs

Dealing with multiple **L2** guests may sound intimidating, but it is actually quite simple once a single **L2** works. Understand that in _VT-x_, the only thing seperating _VMs_ in the eyes of the hardware is the _VMCS_. A _VMM_ switching _VMs_ can either overwrite the current _VMCS_ with data for the other _VM_, or more commonly, switch _VMCS_ with _vmptrld_. Thus, for **L0** it is only necessay to keep track of the _current VMCS_, and recreate _vmcs02_ as a result of **L1** executing _vmwrite_ or _vmptrld_. This is assuming that **L0** is content to leave most of the _VM_ management to **L1**, something that would make sense when nesting.

Since when loading a _VMCS_ (_vmptrld_), the _VMM_ must provide a memory region to contain its data, it is possible to use this region to read and write fields as requested by **L1**. This approach allows **L0** to forgo maintaining multiple _vmcs02_ copies and trust this responsibility to **L1**, which must maintain one or more _vmcs12_. Something that would require copying from the new _vmcs12_ into _vmcs02_ when a guest is replaced. It is possible to avoid this by keeping a copy per _vmcs12_ and use its memory address as a sort of identifier, which would only require editing _vmcs02_ when _vmcs12_ is edited (_vmwrite_), but not when replaced (_vmptrld_). Either approach works, choosing is an exercise in balancing memory use (maintaining copies) and cpu usage (copying on switch).

### Extended Page Tables

### Performance
