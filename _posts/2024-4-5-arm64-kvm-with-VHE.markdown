---
layout: post
title:  "arm64 kvm with VHE"
date:   2024-4-6 7:00:16 +0800
categories: virtualization 
---
**作者: flynn**

### VHE (virtualization host extension)

![vhe_0](/assets/virt/vhe_0.png)

Virtualization host extension was introduced in ARMv8.1. See figure 1 (c), this extension allow hypervisor kernel and kvm run at EL2, which has following advantages:

1. transitioning from the VM to the hypervisor no longer requres saving and restoring the kernel mode register state.
2. the hypervisor OS kernel can program hardware state used by the VM directly when needed, avoid extra copying to and from intermediate data structures.
3. hypervisor and hypervisor OS kernel share the same memory address space, so no need to set up a separate address space for hypervisor, and hypervisor can use OS kernel functions directly, thus, simplify the design of hypervisor.

VHE can be enabled by setting HCR_EL2.E2H bit.

### Let host kernel run at EL2

we cannot let host kernel run at EL2 without software and hardware modifications, because following restriction:

1. EL2 uses a separate control register then the EL1 control register, such as, ESR_EL1/ESR_EL2, to make matter worse, some register have different layouts or semantics. so we have to change kernel instructions to let kernel run at EL2. 
2. need to handle exceptions from EL0 directly to EL2, e.g., system calls, hardware interrupts, and page faults, HCR.TGE could be set to enable route all exceptions from EL0 directly to EL2, but setting this bit would disable stage 1 MMU in EL0.
3. EL2 use different page table format and only support single virtual address range, controlled by TTBR0_EL2.
4. no ASID support in EL2.

![vhe_1](/assets/virt/vhe_1.png)

let's see what does VHE do to solve previous restrictions:

1. VHE introduces additional EL2 registers to provide the same functionality available in EL1, which means, there is a corresponding EL2 system registers for each EL1 system registers.
2. when HCR.TGE is set, no longer disable stage1 MMU.
3. change EL2 page table format to use the same format as used in EL1, and provide additional separate virtual address space.
4. Add ASID support.

VHE also use **Re-directing register accesses** to let the same kernel binary run at EL1 or EL2(with VHE), because we want to run unmodified hypervisor kernel at EL2, so EL1 system registers are changed to acess EL2 system registers, when VHE is enabled, cpu access xxx_EL1 would be translate to acess xxx_EL2.

![vhe_2](/assets/virt/vhe_2.png)

However, this rediretion lead to a new problem, A hypervisor still needs to access the real _EL1 registers to prepare VM's execution context. To solve this, VHE adds new set of the register aliases, the _EL12 or _EL02 suffix. when used at EL2, with **E2H == 1**,  these give access to the EL1 register.

![vhe_3](/assets/virt/vhe_3.png)

All these changes could allow the host kernel to run at EL2 with very little effort, only have to modify the early boot code, see below code

```assembly
diff --git a/arch/arm64/kernel/head.S b/arch/arm64/kernel/head.S
index 917d981..6f2f377 100644
--- a/arch/arm64/kernel/head.S
+++ b/arch/arm64/kernel/head.S
@@ -30,6 +30,7 @@ 
 #include <asm/cache.h>
 #include <asm/cputype.h>
 #include <asm/kernel-pgtable.h>
+#include <asm/kvm_arm.h>
 #include <asm/memory.h>
 #include <asm/pgtable-hwdef.h>
 #include <asm/pgtable.h>
@@ -464,9 +465,27 @@  CPU_LE(	bic	x0, x0, #(3 << 24)	)	// Clear the EE and E0E bits for EL1
 	isb
 	ret
 
+2:
+#ifdef CONFIG_ARM64_VHE
+	/*
+	 * Check for VHE being present. For the rest of the EL2 setup,
+	 * x2 being non-zero indicates that we do have VHE, and that the
+	 * kernel is intended to run at EL2.
+	 */
+	mrs	x2, id_aa64mmfr1_el1
+	ubfx	x2, x2, #8, #4
+#else
+	mov	x2, xzr
+#endif
+
 	/* Hyp configuration. */
-2:	mov	x0, #(1 << 31)			// 64-bit EL1
+	mov	x0, #HCR_RW			// 64-bit EL1
+	cbz	x2, set_hcr
+	orr	x0, x0, #HCR_TGE		// Enable Host Extensions
+	orr	x0, x0, #HCR_E2H
+set_hcr:
 	msr	hcr_el2, x0
+	isb
 
 	/* Generic timers. */
 	mrs	x0, cnthctl_el2
@@ -526,6 +545,13 @@  CPU_LE(	movk	x0, #0x30d0, lsl #16	)	// Clear EE and E0E on LE systems
 	/* Stage-2 translation */
 	msr	vttbr_el2, xzr
 
+	cbz	x2, install_el2_stub
+
+	mov	w20, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
+	isb
+	ret
+
+install_el2_stub:
 	/* Hypervisor stub */
 	adrp	x0, __hyp_stub_vectors
 	add	x0, x0, #:lo12:__hyp_stub_vectors
```

### KVM Redesign to fit VHE

VHE can also facilitate the redesign of KVM, making it cleaner and simpler, the major changes are as follow:

1. since host kernel is running at EL2, while guest kernel at EL1, there is no need to restore/save cpu state while guest_enter/guest_exit happens, but, while migrating vcpu to another physical cpu, the cpu state(at EL1) still need to be save/restored.

2. avoid enabling and disabling virtualization features(stage 2 translation, virtual interrupt, traps on sensitive instructions) on every transition between VM and the hypervisor, because host kernel running at EL2 has the full access to the hardware resources, the detail is hide in `kvm_vcpu_run_vhe`

   ```c
   int kvm_arch_vcpu_ioctl_run(struct kvm_vcpu *vcpu, struct kvm_run *run)
   {
       ...
    	if (has_vhe()) {
   		kvm_arm_vhe_guest_enter();
      		ret = kvm_vcpu_run_vhe(vcpu);
      		kvm_arm_vhe_guest_exit();
   	} else {
      		ret = kvm_call_hyp(__kvm_vcpu_run_nvhe, vcpu);
   	}
       ...
   }    
   ```

3. Data transfer between EL1(host kernel) and EL2(lowvisor) can be removed, thus no need to create identity mapping.

   ```c
   int create_hyp_mappings(void *from, void *to, pgprot_t prot)
   {
       phys_addr_t phys_addr;
       unsigned long virt_addr;
       unsigned long start = kern_hyp_va((unsigned long)from);
       unsigned long end = kern_hyp_va((unsigned long)to);
   n o
       if (is_kernel_in_hyp_mode())
           return 0;
   ...
   }
   ```

4. host kernel's control transfer to kvm no longer through `hvc`

<img src="/assets/virt/vhe_4.png" alt="vhe_4" style="zoom:50%;" />

1. the hypervisor must be modified to replace all EL1 access instructions that should continue to access VM's EL1 register with the new xxx_EL12 access instructions when using VHE.

   ```c
   +#define read_sysreg_elx(r,nvh,vh)					\
   +	({								\
   +		u64 reg;						\
   +		asm volatile(ALTERNATIVE("mrs %0, " __stringify(r##nvh),\
   +					 "mrs_s %0, " __stringify(r##vh),\
   +					 ARM64_HAS_VIRT_HOST_EXTN)	\
   +			     : "=r" (reg));				\
   +		reg;							\
   +	})
   +#define read_sysreg_el0(r)	read_sysreg_elx(r, _EL0, _EL02)
   +#define write_sysreg_el0(v,r)	write_sysreg_elx(v, r, _EL0, _EL02)
   +#define read_sysreg_el1(r)	read_sysreg_elx(r, _EL1, _EL12)
   +#define write_sysreg_el1(v,r)	write_sysreg_elx(v, r, _EL1, _EL12)
   ```

## Reference

1. [Optimizing the Design and Implementation of the Linux ARM Hypervisor](https://www.usenix.org/system/files/conference/atc17/atc17-dall.pdf)

2. [arm64: VHE: Add support for running Linux in EL2 mode](https://patchwork.kernel.org/project/kvm/patch/1454522416-6874-23-git-send-email-marc.zyngier@arm.com/)
