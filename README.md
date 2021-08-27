# KVM patch for Samsung SM-A600FN (exynos7870)

This repository contains a patch for Samsung's official kernel sources that provides KVM support on exynos boards. This can probably be applied to other Samsung exynos kernels with minor changes.

## How to use

1. Download official kernel sources from [opensource.samsung.com](https://opensource.samsung.com/uploadList?menuItem=mobile&classification1=mobile_phone).
2. Apply the patch
3. Copy `/proc/config.gz` from your device, and unzip to `.config`
4. Install and configure a cross-compiler: for aarch64 CPU you can do `git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9` then `git checkout ndk-release-r19` and add `bin` directory in the repo to `$PATH`.
5. In `make menuconfig` disable everything about TIMA (?) and RKP under "Boot Options" (they are incompatible with KVM), and enable KVM under "Virtualization".
6. Build and flash your shiny KVM-enabled kernel!

## Known bugs

* Older versions of Samsung's TrustZone do not play well with this. They probably implemented big.LITTLE scheduling outside of the kernel, which breaks KVM's assumption that VM's do not suddenly jump from one core to another. This seems to have been fixed in the firmware update based on Android 10, open an issue if this persists. The symptom is that the phone **sometimes** instantly reboots when a VM is launched.
* The register `cntfrq_el0`, which the bootloader should have configured, is set to zero. ARM documentation states that is's only writeable from the highest privilege level (i.e.TrustZone), but readable from any privilege level (impossible to trap accesses), and guest OSes will expect it to hold the architected timer frequency (26.0 MHz). There also does not seem to be an `smc` call to set this value. This means that guest OSes will need to be patched until a TrustZone exploit is found to initialize the register properly.
* Timer handling is somehow broken. Linux boots (with a custom DTB that specifies timer frequency explicitly), but OVMF hangs on the first sleep due to interrupts not arriving.

## Technical details

Normally Linux needs to be booted in EL2 (HYP mode in ARM terminology) to be able to utilize the virtualization extensions. SBOOT boots the Linux kernel in EL1, but fortunately for us they implemented a backdoor in TrustZone to load and execute custom code in EL2. This interface is utilized by `init/vmm.c` in Samsung's kernel to load the proprietary RKP hypervisor, and looks as follows:

```
#define VMM_64BIT_SMC_CALL_MAGIC 0xC2000400
#define VMM_STACK_OFFSET 4096
#define VMM_MODE_AARCH64 1
status = _vmm_goto_EL2(VMM_64BIT_SMC_CALL_MAGIC, entry_point, VMM_STACK_OFFSET, VMM_MODE_AARCH64, vmm, vmm_size);
```

Here `_vmm_goto_EL2` is a simple wrapper around `smc #0`, `entry_point` is a physical address of the initialization routine, and the last two parameters are passed to it in `x0` and `x1` registers. To return, the initialization routine calls `smc #0` with `x0=0xc2000401, x1=status` (the only piece of information that was obtained by disassembling the proprietary hypervisor).

The semantics of this interface are as follows:
* `x1` (`status`) is passed to the kernel as the return value of `_vmm_goto_EL2`. If it is nonzero, the firmware resets the HYP state to default, and further `hvc` calls result in an exception.
* EL2 initialization code only runs on the boot CPU. When further CPUs are brought up, it's EL2 state is copied from the one established by the initialization routine, with one exception: `sp` is set to `bootcore.sp + VMM_STACK_OFFSET * core_index`. The numbering used is the same as in Linux kernel.
* The firmware saves the value of `vbar_el2` at exit from the initialization routine, and restores it to this value at some random (unknown) points. This means that the address of the exception vector can not be changed later by EL2 code.

Normal KVM/ARM bootstrap process:
* The code in `head.S` detects being booted in EL2 and sets the EL2 exception vector to a so-called "HYP stub" (basically a backdoor), and drops to EL1 to continue booting.
* When the KVM subsystem begins initialization, it calls the backdoor to run its initialization code, and changes `vbar_el2` to point to the real exception vector.

KVM/ARM bootstrap process with this patch:
* After `mm_init()` is called from `start_kernel`, a new function `preinit_hyp_mode()` is called, that calls the KVM initialization code via the TrustZone backdoor (initialization code itself was also changed to exit via `smc #0` instead of `eret`). This point is chosen because before that that code would fail on a memory allocation, and if done too late other cores could be already running.
* When the normal ("late") KVM initialization routine starts, it initializes **everything except** the EL2 state.
* The check for EL2 boot in `arch/arm64/include/asm/virt.h` is replaced with a simple `return 1;`

**What did not work:**
* In the early bootup code, use the backdoor to enter EL2, and continue booting from there, imitating normal EL2 boot. This probably fails later when secondary cores boot up, causing a sanity check in `arch/arm64/include/asm/virt.h` to fail (boot CPU booted in EL2, others in EL1).
* Use the backdoor early to bootstrap a valid-looking HYP stub, then let KVM boot normally. Does not work due to the custom `vbar_el2` handling, see above.

## Prebuilt kernel by @sleirsgoevy
The prebuilt kernel include Magisk patch and KVM. You can download it [here](https://mega.nz/file/d8lGhY7b#NKQZEL3G6bT7SetrHLh4rNgmgg0L5EXJ0Lir_QjAebA)
