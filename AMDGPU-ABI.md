# AMDGPU Compute Application Binary Interface

Version 0.40 (March 2016)

Table of Contents
=================

* [Introduction](#introduction)
* [Finalizer, Code Object, Executable and Loader](#finalizer-code-object-executable-and-loader)
* [Kernel dispatch](#kernel-dispatch)
* [Hardware registers setup](#hardware-registers-setup)
* [Initial kernel register state](#initial-kernel-register-state)
* [Kernel prolog code](#kernel-prolog-code)
* [Global/Readonly/Kernarg segments](#globalreadonlykernarg-segments)
* [Scratch memory swizzling](#scratch-memory-swizzling)
* [Flat addressing](#flat-addressing)
* [Flat scratch](#flat-scratch)
* [M0 Register](#m0-register)
* [Dynamic call stack](#dynamic-call-stack)
* [Memory model](#memory-model)
  * [Memory model overview](#memory-model-overview)
  * [Memory operation constraints for global segment](#memory-operation-constraints-for-global-segment)
  * [Memory operation constraints for group segment](#memory-operation-constraints-for-group-segment)
  * [Memory fence constraints](#memory-fence-constraints)
* [Instruction set architecture](#instruction-set-architecture)
* [AMD Kernel Code](#amd-kernel-code)
  * [AMD Kernel Code Object amd_kernel_code_t](#amd-kernel-code-object-amd_kernel_code_t)
  * [Compute shader program settings 1 amd_compute_pgm_rsrc1_t](#compute-shader-program-settings-1-amd_compute_pgm_rsrc1_t)
  * [Compute shader program settings 2 amd_compute_pgm_rsrc2_t](#compute-shader-program-settings-2-amd_compute_pgm_rsrc2_t)
  * [AMD Machine Kind amd_machine_kind_t](#amd-machine-kind-amd_machine_kind_t)
  * [Float Round Mode amd_float_round_mode_t](#float-round-mode-amd_float_round_mode_t)
  * [Denorm Mode amd_float_denorm_mode_t](#denorm-mode-amd_float_denorm_mode_t)
* [PCIe Gen3 Atomic Operations](#pcie-gen3-atomic-operations)
* [AMD Queue](#amd-queue)
   * [HSA AQL Queue Object hsa_queue_t](#hsa-aql-queue-object-hsa_queue_t)
   * [AMD AQL Queue Object amd_queue_t](#amd-aql-queue-object-amd_queue_t)
   * [Queue operations](#queue-operations)
* [Signals](#signals)
  * [Signals overview](#signals-overview)
  * [Signal kind amd_signal_kind_t](#signal-kind-amd_signal_kind_t)
  * [Signal object amd_signal_t](#signal-object-amd_signal_t)
  * [Signal kernel machine code](#signal-kernel-machine-code)
* [Debugtrap](#debugtrap)
* [References](#references)

## Introduction

This specification defines the application binary interface (ABI) provided by the AMD implementation of the HSA runtime for AMD GPU architecture agents. The AMD GPU architecture is a family of GPU agents which differ in machine code encoding and functionality.

## Finalizer, Code Object, Executable and Loader

Finalizer, Code Object, Executable and Loader are defined in "HSA Programmer Reference Manual Specification". AMD Code Object uses ELF format. In this document, Finalizer is any compiler producing code object, including kernel machine code.

## Kernel dispatch

The HSA Architected Queuing Language (AQL) defines a user space memory interface, an AQL Queue, to an agent that can be used to control the dispatch of kernels, using AQL Packets, in an agent independent way. All AQL packets are 64 bytes and are defined in "HSA Platform System Architecture Specification". The packet processor of a kernel agent is responsible for detecting and dispatching kernels from the AQL Queues associated with it. For AMD GPUs the packet processor is implemented by the Command Processor (CP).

The AMD HSA runtime allocates the AQL Queue object. It uses the AMD Kernel Fusion Driver (KFD) to initialize and register the AQL Queue with CP. Refer to ["AMD Queue"](#amd-queue) for more information.

A kernel dispatch is initiated with the following sequence defined in "HSA System Architecture Specification" (it may occur on CPU host agent from a host program, or from an HSA kernel agent such as a GPU from another kernel):
  * A pointer to an AQL Queue for the kernel agent on which the kernel is to be executed is obtained.
  * A pointer to the amd_kernel_code_t object of the kernel to execute is obtained. It must be for a kernel that was loaded on the kernel agent with which the AQL Queue is associated.
  * Space is allocated for the kernel arguments using the HSA runtime allocator for a memory region with the kernarg property for the kernel agent that will execute the kernel, and the values of the kernel arguments are assigned. This memory corresponds to the backing memory for the kernarg segment within the kernel being called. Its layout is defined in "HSA Programmer Reference Manual Specification". For AMD the kernel execution directly uses the backing memory for the kernarg segment as the kernarg segment.
  * Queue operations is used to reserve space in the AQL queue for the packet.
  * The packet contents are set up, including information about the actual dispatch, such as grid and work-group size, together with information from the code object about the kernel, such as segment sizes.
  * The packet is assigned to packet processor by changing format field from INVALID to KERNEL_DISPATCH. Atomic memory operation must be used.
  * A doorbell signal for the queue is signaled to notify packet processor.

At some point, CP performs actual kernel execution:
  * CP detects a packet on AQL queue.
  * CP executes micro-code for setting up the GPU and wavefronts for a kernel dispatch.
  * CP ensures that when a wavefront starts executing the kernel machine code, the scalar general purpose registers (SGPR) and vector general purpose registers (VGPR) are set up based on flags in amd_kernel_code_t (see ["Initial kernel register state"](#initial-kernel-register-state)).
  * When a wavefront start executing the kernel machine code, the prolog (see ["Kernel prolog code"](#kernel-prolog-code)) sets up the machine state as necessary.
  * When the kernel dispatch has completed execution, CP signals the completion signal specified in the kernel dispatch packet if not 0.

## Hardware registers setup

SH_MEM_CONFIG register:
  * DEFAULT_MTYPE = 1 (MTYPE_NC)
  * ALIGNMENT_MODE = 3 (SH_MEM_ALIGNMENT_MODE_UNALIGNED)
  * PTR32 = 1 in 32-bit mode and 0 in 64-bit mode

## Initial kernel register state

Prior to start of every wavefront execution, CP/SPI sets up the register state based on enable\_sgpr\_\* and enable\_vgpr\_\* flags in amd_kernel_code_t object:
  * SGPRs before the Work-Group Ids are set by CP using the 16 User Data registers.
  * Work-group Id registers X, Y, Z are set by SPI which supports any combination including none.
  * Scratch Wave Offset is also set by SPI which is why its value cannot be added into the value Flat Scratch Offset (which would avoid the Finalizer generated prolog having to do the add).
  * The VGPRs are set by SPI which only supports specifying either (X), (X, Y) or (X, Y, Z).

SGPR register numbers used for enabled registers are dense starting at SGPR0: the first enabled register is SGPR0, the next enabled register is SGPR1 etc.; disabled registers do not have an SGPR number. Because of hardware constraints, the initial SGPRs comprise up to 16 User SRGPs that are set up by CP and apply to all waves of the grid. It is possible to specify more than 16 User SGPRs using the enable_sgpr_* bit fields, in which case only the first 16 are actually initialized. These are then immediately followed by the System SGPRs that are set up by ADC/SPI and can have different values for each wave of the grid dispatch.

The number of enabled registers must match value in compute\_pgm\_rsrc2.user\_sgpr (the total count of SGPR user data registers enabled). *The enableGridWorkGroupCount\* is currently not implemented in CP and should always be 0.*

The following table defines SGPR registers that can be enabled and their order.

| **SGPR Order** | **Number of Registers** | **Name**  | **Description** |
| --- | --- | --- | --- |
| First | 4 | Private Segment Buffer (enable\_sgpr\_private\_segment\_buffer) | V\# that can be used, together with Scratch Wave Offset as an offset, to access the Private/Spill/Arg segments using a segment address. CP uses the value from amd\_queue\_t.scratch\_resource\_descriptor. |
| then | 2 | Dispatch Ptr (enable\_sgpr\_dispatch\_ptr) | 64 bit address of AQL dispatch packet for kernel actually executing. |
| then | 2 | Queue Ptr (enable\_sgpr\_queue\_ptr) | 64 bit address of amd\_queue\_t object for AQL queue on which the dispatch packet was queued. |
| then | 2 | Kernarg Segment Ptr (enable\_sgpr\_kernarg\_segment\_ptr) | 64 bit address of Kernarg segment. This is directly copied from the kernarg\_address in the kernel dispatch packet. Having CP load it once avoids loading it at the beginning of every wavefront. |
| then | 2 | Dispatch Id (enable\_sgpr\_dispatch\_id) | 64 bit Dispatch ID of the dispatch packet being executed. |
| then | 2 | Flat Scratch Init (enable\_sgpr\_flat\_scratch\_init) | Value used for FLAT_SCRATCH register initialization. Refer to [Flat scratch](#flat-scratch) for more information. |
| then | 1 | Private Segment Size (enable\_sgpr\_private\_segment\_size) | The 32 bit byte size of a single work-items scratch memory allocation. This is the value from the kernel dispatch packet Private Segment Byte Size rounded up by CP to a multiple of DWORD. Having CP load it once avoids loading it at the beginning of every wavefront. Not used for GFX7/GFX8 since it is the same value as the second SGPR of Flat Scratch Init. |
| then | 1 | Grid Work-Group Count X (enable\_sgpr\_grid\_workgroup\_count\_X) | 32 bit count of the number of work-groups in the X dimension for the grid being executed. Computed from the fields in the kernel dispatch packet as ((grid\_size.x + workgroup\_size.x - 1) / workgroup\_size.x).
| then | 1 | Grid Work-Group Count Y (enable\_sgpr\_grid\_workgroup\_count\_Y && less than 16 previous SGPRs) | 32 bit count of the number of work-groups in the Y dimension for the grid being executed. Computed from the fields in the kernel dispatch packet as ((grid\_size.y + workgroup\_size.y - 1) / workgroupSize.y). Only initialized if \<16 previous SGPRs initialized. |
| then | 1 | Grid Work-Group Count Z (enable\_sgpr\_grid\_workgroup\_count\_Z && less than 16 previous SGPRs) | 32 bit count of the number of work-groups in the Z dimension for the grid being executed. Computed from the fields in the kernel dispatch packet as ((grid\_size.z + workgroup\_size.z - 1) / workgroupSize.z). Only initialized if \<16 previous SGPRs initialized. |
| then | 1 | Work-Group Id X (enable\_sgpr\_workgroup\_id\_X) | 32 bit work group id in X dimension of grid for wavefront. Always present. |
| then | 1 | Work-Group Id Y (enable\_sgpr\_workgroup\_id\_Y) | 32 bit work group id in Y dimension of grid for wavefront. |
| then | 1 | Work-Group Id Z (enable\_sgpr\_workgroup\_id\_Z) | 32 bit work group id in Z dimension of grid for wavefront. If present then Work-group Id Y will also be present. |
| then | 1 | Work-Group Info (enable\_sgpr\_workgroup\_info) | {first\_wave, 14b0000, ordered\_append\_term[10:0], threadgroup\_size\_in\_waves[5:0]} |
| then | 1 | Private Segment Wave Byte Offset (enable\_sgpr\_private\_segment\_wave\_byte\_offset) | 32 bit byte offset from base of scratch base of queue executing the kernel dispatch. Must be used as an offset with Private/Spill/Arg segment address when using Scratch Segment Buffer. It must be added to Flat Scratch Offset if setting up FLAT SCRATCH for flat addressing. |

VGPR register numbers used for enabled registers are dense starting at VGPR0: the first enabled register is VGPR0, the next enabled register is VGPR1 etc.; disabled registers do not have a VGPR number.

The following table defines VGPR registers that can be enabled and their order.

| **VGPR Order** | **Number of Registers** | **Name**  | **Description** |
| --- | --- | --- | --- |
| First | 1 | Work-Item Id X (Always initialized) | 32 bit work item id in X dimension of work-group for wavefront lane. |
| then | 1 | Work-Item Id Y (enable\_vgpr\_workitem\_id \> 0) | 32 bit work item id in Y dimension of work-group for wavefront lane. |
| then | 1 | Work-Item Id Z (enable\_vgpr\_workitem\_id \> 1) | 32 bit work item id in Z dimension of work-group for wavefront lane. |

## Kernel prolog code

For certain features, kernel is expected to perform initialization actions, normally done in kernel prologue. This is only needed if kernel uses those features.

## Global/Readonly/Kernarg segments

Global segment can be accessed either using flat or buffer operations. Buffer operations cannot be used for large machine model for GFX7 and later as V# support for 64 bit addressing is not available.

If buffer operations are used then the Global Buffer used to access Global/Readonly/Kernarg (combined) segments using a segment address is not passed into the kernel code by CP since its base address is always 0. The prolog code initializes 4 SGPRs with a V# that has the following properties, and then uses that in the buffer instructions:
  * base address of 0
  * no swizzle
  * ATC: 1 if IOMMU present (such as APU)
  * MTYPE set to support memory coherence specified in amd_kernel_code_t.global_memory_coherence

If buffer operations are used to access Kernarg segment, Kernarg address must be added. It is available in dispatch packet (kernarg_address field) or as Kernarg Segment Ptr SGPR. Alternatively, scalar loads can be used if the kernarg offset is uniform, as the kernarg segment is constant for the duration of the kernel dispatch execution.

## Scratch memory swizzling

Scratch memory may be used for private/spill/stack segment. Hardware will interleave (swizzle) scratch accesses of each lane of a wavefront by interleave (swizzle) element size to ensure each work-item gets a distinct memory location. Interleave size must be 2, 4, 8 or 16. The value used must match the value that the runtime configures the GPU flat scratch (SH\_STATIC\_MEM\_CONFIG.ELEMENT\_SIZE).

For GFX8 and earlier, all load and store operations done to scratch buffer must not exceed this size. For example, if the element size is 4 (32-bits or dword) and a 64-bit value must be loaded, it must be split into two 32-bit loads. This ensures that the interleaving will get the work-item specific dword for both halves of the 64-bit value. If it just did a 64-bit load then it would get one dword which belonged to its own work-item, but the second dword would belong to the adjacent lane work-item since the interleaving is in dwords.

AMD HSA Runtime Finalizer uses value 4.

## Flat addressing

Flat address can be used in FLAT instructions and can access global, private (scratch) and group (lds) memory.

Flat access to scratch requires hardware aperture setup and setup in kernel prologue (see [Flat scratch](#flat-scratch)).

Flat access to lds requires hardware aperture setup and M0 register setup (see [M0 register](#m0-register)).

Address operations for group/private segment may use fields from amd_queue_t, the address of which can be obtained with Queue Ptr SGPR.

To obtain null address value for a segment (nullptr HSAIL instruction),
  * For global, readonly and flat segment use value 0.
  * For group, private and kernarg segments, use value -1 (32-bit).

To convert segment address to flat address (stof HSAIL instruction),
  * For global segment, use the same value.
  * For kernarg segment, add Kernarg Segment Ptr. For small model, this is a 32-bit add. For large model, this is 32-bit add to 64-bit base address.
  * For group segment,
    * for large model, combine group\_segment\_aperture\_base\_hi (upper half) and segment address (lower half),
    * for small model, add group\_segment\_aperture\_base\_hi and segment address.
  * For private/spill/arg segment,
    * for large model, combine private\_segment\_aperture\_base\_hi (upper half) and segment address (lower half),
    * for small model, add private\_segment\_aperture\_base\_hi and segment address.
  * If flat address may be null, kernarg, group and private/spill arg segment machine code must have additional sequence (use V_CMP and V_CNDMASK).

To convert flat address to segment address (ftos HSAIL instruction),
  * For global segment, use the same value.
  * For kernarg segment, subtract Kernarg Segment Ptr. For small model, this is a 32-bit subtract. For large model, this is 32-bit subtract from lower half of the 64-bit flat address.
  * For group segment,
    * for large model, use low half of the flat address,
    * for small model, subtract group\_segment\_aperture\_base\_hi.
  * For private/spill/arg segment,
    * for large model, use low half of the flat address,
    * for small model, subtract private\_segment\_aperture\_base\_hi.
  * If segment address may be null, kernarg, group and private/spill arg segment machine code must have additional sequence (use V_CMP and V_CNDMASK).

To determine if given flat address lies within a segment (segmentp HSAIL instruction),
  * For global segment, check that address does not lie in group/private segments
  * For group segment, check if address lies in group segment aperture
    * for large model, check that upper half of 64-bit address == group\_segment\_aperture\_base\_hi,
    * for small model, check that most significant 16 bits of 32-bit address (address & ~0xFFFF) == group\_segment\_aperture\_base\_hi.
  * For private segment, check if address lies in private segment aperture
    * for large model, check that upper half of 64-bit address == private\_segment\_aperture\_base\_hi,
    * for small model, check that most significant 16 bits of 32-bit address (address & ~0xFFFF) == group\_segment\_aperture\_base\_hi.
  * If flat address may be null, machine code must have additional sequence (use V_CMP).

## Flat scratch

If kernel may use flat operations to access scratch memory, the prolog code must set up FLAT_SCRATCH register pair (FLAT_SCRATCH_LO/FLAT_SCRATCH_HI or SGPRn-4/SGPRn-3).

For GFX7/GFX8, initialization uses Flat Scratch Init and Scratch Wave Offset sgpr registers (see [Initial kernel register state](#initial-kernel-register-state)):
  * The low word of Flat Scratch Init is 32 bit byte offset from SH\_HIDDEN\_PRIVATE\_BASE\_VIMID to base of memory for scratch for the queue executing the kernel dispatch. This is the lower 32 bits of amd\_queue\_t.scratch\_backing\_memory\_location and is the same offset used in computing the Scratch Segment Buffer base address. The prolog must add the value of Scratch Wave Offset to it, shift right by 8 (offset is in 256-byte units) and move to FLAT_SCRATCH_LO for use as the FLAT SCRATCH BASE in flat memory instructions.
  * The second word of Flat Scratch Init is 32 bit byte size of a single work-items scratch memory usage. This is directly loaded from the kernel dispatch packet Private Segment Byte Size and rounded up to a multiple of DWORD. Having CP load it once avoids loading it at the beginning of every wavefront. The prolog must move it to FLAT_SCRATCH_LO for use as FLAT SCRATCH SIZE.

## M0 register

M0 register must be initialized with total LDS size if kernel may access LDS via DS or flat operations. Total LDS size is available in dispatch packet. For M0, it is also possible to use maximum possible value of LDS for given target.

## Dynamic call stack

In certain cases, Finalizer cannot compute the total private segment size at compile time. This can happen if calls are implemented using a call stack and recursion, alloca or calls to indirect functions are present. In this case, workitem\_private\_segment\_byte\_size field in code object only specifies the statically known private segment size. When performing actual kernel dispatch, private_segment_size_bytes field in dispatch packet will contain static private segment size plus additional space for the call stack.

## Memory model

### Memory model overview

A memory model describes the interactions of threads through memory and their shared use of the data. Many modern programming languages implement a memory model. This section describes the mapping of common memory model constructs onto AMD GPU architecture.

Through this section, definitions and constraints from "HSA Platform System Architecture Specification 1.0" are used as reference, although similar notions exist elsewhere (for example, in C99 or C++ 11).

The following memory scopes are defined:
  * Work-item (wi)
  * Wavefront (wave)
  * Work-group (wg)
  * Agent (agent)
  * System (system)

The following memory orders are defined:
  * scacq: sequentially consistent acquire
  * screl: sequentially consistent release
  * scar: sequentially consistent acquire and release
  * rlx: relaxed

The following memory operations are defined:
  * Ordinary Load/Store (non-synchronizing operations)
  * Atomic Load/Atomic Store (synchronizing operations)
  * Atomic RMW (Read-Modify-Write: add, sub, max, min, and, or, xor, wrapinc, wrapdec, exch, cas (synchronizing operations)
  * Memory Fence (synchronizing operation)

Sometimes derived notation is used. For example, agent+ means agent and system scopes, wg- means work-group, wavefront and work-item scopes.

In the following sections, a combination of memory segment, operation, order and scope is assigned a machine code sequence. Note that if s_waitcnt vmcnt(0) is used to enforce a completion of earlier memory operations in same workitem, it can be omitted if it is also enforced using some other mechanism or proven by compiler (for example, if there are no preceding synchronizing memory operations). Similiarily, if s_waitcnt vmcnt(0) is used to enforce completion of this memory operation before the following memory operations, sometimes it can be omitted (for example, if there are no following synchronizing memory operations).

For a flat memory operation, if it may affect either global or group segment, group constraints must be applied to flat operations as well.

### Memory operation constraints for global segment

For global segment, the following machine code instructions may be used (see [Global/Readonly/Kernarg segments](#global/readonly/kernarg-segments)):
  * Ordinary Load/Store: BUFFER_LOAD/BUFFER_STORE or FLAT_LOAD/FLAT_STORE
  * Atomic Load/Store: BUFFER_LOAD/BUFFER_STORE or FLAT_LOAD/FLAT_STORE
  * Atomic RMW: BUFFER_ATOMIC or FLAT_ATOMIC

| **Operation** | **Memory order** | **Memory scope** | **Machine code sequence** |
| --- | --- | --- | --- |
| Ordinary Load | - | - | load with glc=0 |
| Atomic Load | rlx,scacq | wg- | load with glc=0 |
| Atomic Load | rlx | agent+ | load with glc=1 |
| Atomic Load | scacq | agent+ | load with glc=1; s_waitcnt vmcnt(0); buffer_wbinv_vol |
| Ordinary Store | - | - | store with glc=0 |
| Atomic Store | rlx,screl | wg- | store with glc=0 |
| Atomic Store | rlx | agent+ | store with glc=0 |
| Atomic Store | screl | agent+ | s_waitcnt vmcnt(0); store with glc=0; s_waitcnt vmcnt(0) |
| Atomic RMW | rlx,scacq, screl, scar | wg- | atomic |
| Atomic RMW | rlx | agent+ | atomic |
| Atomic RMW | scacq | agent+ | atomic; s_waitcnt vmcnt(0); buffer_wbinv_vol |
| Atomic RMW | screl | agent+ | s_waitcnt vmcnt(0); atomic |
| Atomic RMW | scar | agent+ | s_waitcnt vmcnt(0); atomic; s_waitcnt vmcnt(0); buffer_wbinv_vol |

### Memory operation constraints for group segment

For group segment, the following machine code instructions are used:
  * Ordinary Load/Store: DS_READ/DS_WRITE
  * Atomic Load/Store: DS_READ/DS_WRITE
  * Atomic RMW: DS_ADD, DS_SUB, DS_MAX, DS_MIN, DS_AND, DS_OR, DS_XOR, DS_INC, DS_DEC, DS_WRXCHG, DS_CMPST (and corresponding RTN variants)

AMD LDS hardware is sequentially consistent. This means that it is not necessary to use lgkmcnt to enforce ordering in single work-item for group segment synchronization. s_waitcnt lgkmcnt(0) should still be used to enforce data dependencies, for example, after a load into a register and before use of that register (same applies to non-synchronizing operations).

The current model (and HSA) requires that global and group segments are coherent. This is why synchronizing group segment operations and memfence also use s_waitcnt vmcnt(0).

| **Operation** | **Memory order** | **Memory scope** | **Machine code sequence** |
| --- | --- | --- | --- |
| Ordinary Load | - | - | load |
| Atomic Load | rlx | wg- | load |
| Atomic Load | scacq | wg- | s_waitcnt vmcnt(0); load; buffer_wbinvl1_vol |
| Ordinary Store | - | - | store |
| Atomic Store | rlx | wg- | store |
| Atomic Store | screl | wg- | s_waitcnt vmcnt(0); store |
| Atomic RMW | scacq | wg- | s_waitcnt vmcnt(0); atomic; buffer_wbinvl1_vol |
| Atomic RMW | screl | wg- | s_waitcnt vmcnt(0); atomic |
| Atomic RMW | scacq | wg- | s_waitcnt vmcnt(0); atomic; buffer_wbinvl1_vol |

### Memory fence constraints

Memory fence is currently applied to all segments (cross-segment synchronization). In machine code, memory fence does not have separate instruction and maps to s_waitcnt and buffer_wbinvl1_vol instructions.  In addition, memory fence must not be moved in machine code with respect to other synchronizing operations. In the following table, 'memfence' refers to conceptual memory fence location.

| **Operation** | **Memory order** | **Memory scope** | **Machine code sequence** |
| --- | --- | --- | --- |
| Memory Fence | scacq,screl,scar | wg- | memfence (no additional constraints) |
| Memory Fence | scacq | agent+ | memfence; s_waitcnt 0; buffer_wbinvl1_vol |
| Memory Fence | screl | agent+ | s_waitcnt 0; memfence |
| Memory Fence | scar | agent + | memfence; s_waitcnt 0; buffer_wbinvl1_vol |

## Instruction set architecture

AMDGPU ISA specifies instruction set architecture and capabilities used by machine code. It consists of several fields:
  * Vendor ("AMD")
  * Architecture ("AMDGPU")
  * Major (GFXIP), minor and stepping versions

These fields may be combined to form one defining string, for example, "AMD:AMDGPU:8:0:0".

| **Vendor** | **Architecture** | **Major** | **Minor** | **Stepping** | **Comments** | **Examples** |
| --- | --- | --- | --- | --- | --- | --- |
| AMD | AMDGPU | 7 | 0 | 0 | GFX7, 1/16 double FP | Kaveri, Bonaire |
| AMD | AMDGPU | 7 | 0 | 1 | GFX7, 1/2 double FP | Hawaii |
| AMD | AMDGPU | 8 | 0 | 0 | GFX8, SPI register limitation, -XNACK | Iceland, Tonga |
| AMD | AMDGPU | 8 | 0 | 1 | GFX8, +XNACK | Carrizo |
| AMD | AMDGPU | 8 | 0 | 2 | GFX8, SPI register limitation, -XNACK, PCIe Gen3 atomics | ROCm Tonga |
| AMD | AMDGPU | 8 | 0 | 3 | GFX8, -XNACK, PCIe Gen3 atomics | ROCm Fiji |
| AMD | AMDGPU | 8 | 0 | 4 | GFX8, -XNACK | Fiji |
| AMD | AMDGPU | 8 | 1 | 0 | GFX8, +XNACK | Stoney |

## AMD Kernel Code

AMD Kernel Code object is used by AMD GPU CP to set up the hardware to execute a kernel dispatch and consists of the meta data needed to initiate the execution of a kernel, including the entry point address of the machine code that implements the kernel.

### AMD Kernel Code Object amd_kernel_code_t

| **Bits** | **Size** | **Field Name** | **Description** |
| --- | --- | --- | --- |
| 31:0 | 4 bytes | amd\_code\_version\_major | The AMD major version. Must be the value AMD\_KERNEL\_CODE\_VERSION\_MAJOR. Major versions are not backwards compatible. |
| 63:32 | 4 bytes | amd\_code\_version\_minor | The AMD minor version. Must be the value AMD\_CODE\_VERSION\_MINOR.  Minor versions with the same major version must be backward compatible. |
| 79:64 | 2 bytes | amd\_machine\_kind | [Machine kind](#amd-machine-kind-amd_machine_kind_t). |
| 95:80 | 2 bytes | amd\_machine\_version\_major | [Instruction set architecture](#instruction-set-architecture): major |
| 111:96 | 2 bytes | amd\_machine\_version\_minor | [Instruction set architecture](#instruction-set-architecture): minor |
| 127:112 | 2 bytes | amd\_machine\_version\_stepping | [Instruction set architecture](#instruction-set-architecture): stepping |
| 191:128 | 8 bytes | kernel\_code\_entry\_byte\_offset | Byte offset (possibly negative) from start of amd\_kernel\_code\_t object to kernel's entry point instruction. The actual code for the kernel is required to be 256 byte aligned to match hardware requirements (SQ cache line is 16; entry point config register only holds bits 47:8 of the address). The Finalizer should endeavor to allocate all kernel machine code in contiguous memory pages so that a device pre-fetcher will tend to only pre-fetch Kernel Code objects, improving cache performance. The AMD HA Runtime Finalizer generates position independent code (PIC) to avoid using relocation records and give runtime more flexibility in copying code to discrete GPU device memory. |
| 255:192 | 8 bytes | kernel\_code\_prefetch\_byte\_offset | Range of bytes to consider prefetching expressed as a signed offset and unsigned size. The (possibly negative) offset is from the start of amd\_kernel\_code\_t object. Set both to 0 if no prefetch information is available. |
| 319:256 | 8 bytes | kernel\_code\_prefetch\_byte\_size | |
| 383:320 | 8 bytes | max\_scratch\_backing\_memory\_byte\_size | Number of bytes of scratch backing memory required for full occupancy of target chip. This takes into account the number of bytes of scratch per work-item, the wavefront size, the maximum number of wavefronts per CU, and the number of CUs. This is an upper limit on scratch. If the grid being dispatched is small it may only need less than this. If the kernel uses no scratch, or the Finalizer has not computed this value, it must be 0. |
| 415:384 | 4 bytes | compute\_pgm\_rsrc1 | [Compute Shader (CS) program settings 1 amd_compute_pgm_rsrc1](#compute-shader-program-settings-1-amd_compute_pgm_rsrc1_t) |
| 447:416 | 4 bytes | compute\_pgm\_rsrc2 | [Compute Shader (CS) program settings 2 amd_compute_pgm_rsrc2](#compute-shader-program-settings-2-amd_compute_pgm_rsrc2_t) |
| 448 | 1 bit | enable\_sgpr\_private\_segment\_buffer | Enable the setup of [Private Segment Buffer](#initial-kernel-register-state) |
| 449 | 1 bit | enable\_sgpr\_dispatch\_ptr | Enable the setup of [Dispatch Ptr](#initial-kernel-register-state) |
| 450 | 1 bit | enable\_sgpr\_queue\_ptr | Enable the setup of [Queue Ptr](#initial-kernel-register-state) |
| 451 | 1 bit | enable\_sgpr\_kernarg\_segment\_ptr | Enable the setup of [Kernarg Segment Ptr](#initial-kernel-register-state) |
| 452 | 1 bit | enable\_sgpr\_dispatch\_id | Enable the setup of [Dispatch Id](#initial-kernel-register-state) |
| 453 | 1 bit | enable\_sgpr\_flat\_scratch\_init | Enable the setup of [Flat Scratch Init](#initial-kernel-register-state) |
| 454 | 1 bit | enable\_sgpr\_private\_segment\_size | Enable the setup of [Private Segment Size](#initial-kernel-register-state) |
| 455 | 1 bit | enable\_sgpr\_grid\_workgroup\_count\_X | Enable the setup of [Grid Work-Group Count X](#initial-kernel-register-state) |
| 456 | 1 bit | enable\_sgpr\_grid\_workgroup\_count\_Y | Enable the setup of [Grid Work-Group Count Y](#initial-kernel-register-state) |
| 457 | 1 bit | enable\_sgpr\_grid\_workgroup\_count\_Z | Enable the setup of [Grid Work-Group Count Z](#initial-kernel-register-state) |
| 463:458 | 6 bits | | Reserved. Must be 0. |
| 464 | 1 bit | enable\_ordered\_append\_gds | Control wave ID base counter for GDS ordered-append. Used to set COMPUTE\_DISPATCH\_INITIATOR.ORDERED\_APPEND\_ENBL. |
| 466:465 | 2 bits | private\_element\_size  | [Interleave (swizzle) element size](#scratch-memory-swizzling) in bytes. |
| 467 | 1 bit | is\_ptr64 | 1 if global memory addresses are 64 bits, otherwise 0. Must match SH\_MEM\_CONFIG.PTR32 (GFX7), SH\_MEM\_CONFIG.ADDRESS\_MODE (GFX8+). |
| 468 | 1 bit | is\_dynamic\_call\_stack | Indicates if the generated machine code is using [dynamic call stack](#dynamic-call-stack). |
| 469 | 1 bit | is\_debug\_enabled | Indicates if the generated machine code includes code required by the debugger. |
| 470 | 1 bit | is\_xnack\_enabled | Indicates if the generated machine code uses conservative XNACK register allocation. |
| 479:471 | 9 bits | reserved | Reserved. Must be 0. |
| 511:480 | 4 bytes | workitem\_private\_segment\_byte\_size | The amount of memory required for the static combined private, spill and arg segments for a work-item in bytes. |
| 543:512 | 4 bytes | workgroup\_group\_segment\_byte\_size | The amount of group segment memory required by a work-group in bytes. This does not include any dynamically allocated group segment memory that may be added when the kernel is dispatched. |
| 575:544 | 4 bytes  | gds\_segment\_byte\_size | Number of byte of GDS required by kernel dispatch. Must be 0 if not using GDS. |
| 639:576 | 8 bytes | kernarg\_segment\_byte\_size | The size in bytes of the kernarg segment that holds the values of the arguments to the kernel. This could be used by CP to prefetch the kernarg segment pointed to by the kernel dispatch packet. |
| 671:640 | 4 bytes | workgroup\_fbarrier\_count | Number of fbarrier's used in the kernel and all functions it calls. If the implementation uses group memory to allocate the fbarriers then that amount must already be included in the workgroup\_group\_segment\_byte\_size total. |
| 687:672 | 2 bytes | wavefront\_sgpr\_count | Number of scalar registers used by a wavefront. This includes the special SGPRs for VCC, Flat Scratch (Base, Size) and XNACK (for GFX8 (VI)+). It does not include the 16 SGPR added if a trap handler is enabled. Must match compute\_pgm\_rsrc1.sgprs used to set COMPUTE\_PGM\_RSRC1.SGPRS. |
| 703:688 | 2 bytes | workitem\_vgpr\_count | Number of vector registers used by each work-item. Must match compute\_pgm\_rsrc1.vgprs used to set COMPUTE\_PGM\_RSRC1.VGPRS. |
| 719:704 | 2 bytes | reserved\_vgpr\_first | If reserved\_vgpr\_count is 0 then must be 0. Otherwise, this is the first fixed VGPR number reserved. |
| 735:720 | 2 bytes | reserved\_vgpr\_count | The number of consecutive VGPRs reserved by the client. If is\_debug\_supported then this count includes VGPRs reserved for debugger use. |
| 751:736 | 2 bytes | reserved\_sgpr\_first | If reserved\_sgpr\_count is 0 then must be 0. Otherwise, this is the first fixed SGPR number reserved. |
| 767:752 | 2 bytes | reserved\_sgpr\_count | The number of consecutive SGPRs reserved by the client. If is\_debug\_supported then this count includes SGPRs reserved for debugger use. |
| 783:768 | 2 bytes | debug\_wavefront\_private\_segment\_offset\_sgpr | If is\_debug\_supported is 0 then must be 0. Otherwise, this is the fixed SGPR number used to hold the wave scratch offset for the entire kernel execution, or uint16\_t(-1) if the register is not used or not known. |
| 799:784 | 2 bytes | debug\_private\_segment\_buffer\_sgpr | If is\_debug\_supported is 0 then must be 0. Otherwise, this is the fixed SGPR number of the first of 4 SGPRs used to hold the scratch V\# used for the entire kernel execution, or uint16\_t(-1) if the registers are not used or not known. |
| 807:800 | 1 byte | kernarg\_segment\_alignment | The maximum byte alignment of variables used by the kernel in the specified memory segment. Expressed as a power of two as defined in Table 37. Must be at least HSA\_POWERTWO\_16. |
| 815:808 | 1 byte | group\_segment\_alignment |
| 823:816 | 1 byte | private\_segment\_alignment |
| 831:824 | 1 byte | wavefront\_size | Wavefront size expressed as a power of two. Must be a power of 2 in range 1..256 inclusive. Used to support runtime query that obtains wavefront size, which may be used by application to allocated dynamic group memory and set the dispatch work-group size. |
| 863:832 | 4 bytes | call\_convention | Call convention used to produce the machine code for the kernel. This specifies the function call convention ABI used for indirect functions. If the application specified that the Finalizer should select the call convention, then this value must be the value selected, not the -1 specified to the Finalizer. If the code object does not support indirect functions, then the value must be 0xffffffff. |
| 960:864 | 12 bytes | | Reserved. Must be 0. |
| 1023:960 | 8 bytes | runtime\_loader\_kernel\_symbol | A pointer to the loaded kernel symbol. This field must be 0 when amd\_kernel\_code\_t is created. The HSA Runtime loader initializes this field once the code object is loaded to reference the loader symbol for the kernel. This field is used to allow the debugger to locate the debug information for the kernel. The definition of the loaded kernel symbol is located in hsa/runtime/executable.hpp. |
| 2047:1024 | 128 bytes | control\_directive | Control directives for this kernel used in generating the machine code. The values are intended to reflect the constraints that the code actually requires to correctly execute, not the values that were actually specified at finalize time. If the finalizer chooses to ignore a control directive, and not generate constrained code, then the control directive should not be marked as enabled. |
| 2048 | | | Total size 256 bytes. |

### Compute shader program settings 1 amd_compute_pgm_rsrc1_t

The fields of amd_compute_pgm_rsrc1 are used by CP to set up COMPUTE\_PGM\_RSRC1.

| **Bits** | **Size** | **Field Name** | **Description** |
| --- | --- | --- | --- |
| 5:0 | 6 bits | granulated\_workitem\_vgpr\_count | Number of vector registers used by each work-item, granularity is device specific. |
| 9:6 | 4 bits | granulated\_wavefront\_sgpr\_count | Number of scalar registers used by a wavefront, granularity is device specific. This includes the special SGPRs for VCC, Flat Scratch (Base, and Size) and XNACK (for GFX8 (VI)+). It does not include the 16 SGPR added if a trap handler is enabled. |
| 11:10 | 2 bits | priority | Drives spi\_priority in spi\_sq newWave cmd. |
| 13:12 | 2 bits | float\_mode\_round\_32 | Wavefront initial [float round mode](#float-round-mode-amd_float_round_mode_t) for single precision floats (32 bit). |
| 15:14 | 2 bits | float\_mode\_round\_16\_64 | Wavefront initial [float round mode](#float-round-mode-amd_float_round_mode_t) for double/half precision floats (64/16 bit). |
| 17:16 | 2 bits | float\_mode\_denorm\_32 | Wavefront initial [denorm mode](#denorm-mode-amd_float_denorm_mode_t) for single precision floats (32 bit). |
| 19:18 | 2 bits | float\_mode\_denorm\_16\_64 | Wavefront initial [denorm  mode](#denorm-mode-amd_float_denorm_mode_t) for double/half precision floats (64/16 bit). |
| 20 | 1 bit | priv | Drives priv in spi\_sq newWave cmd. This field is set to 0 by the Finalizer and must be filled in by CP. |
| 21 | 1 bit | enable\_dx10\_clamp | Wavefront starts execution with DX10 clamp mode enabled. Used by the vector ALU to force DX-10 style treatment of NaN's (when set, clamp NaN to zero, otherwise pass NaN through). Used by CP to set up COMPUTE\_PGM\_RSRC1.DX10\_CLAMP. |
| 22 | 1 bit | debug\_mode | Drives debug in spi\_sq newWave cmd. This field is set to 0 by the Finalizer and must be filled in by CP. |
| 23 | 1 bit | enable\_ieee\_mode | Wavefront starts execution with IEEE mode enabled. Floating point opcodes that support exception flag gathering will quiet and propagate signaling-NaN inputs per IEEE 754-2008. Min\_dx10 and max\_dx10 become IEEE 754-2008 compliant due to signaling-NaN propagation and quieting. Used by CP to set up COMPUTE\_PGM\_RSRC1.IEEE\_MODE. |
| 24 | 1 bit | bulky | Only one such work-group is allowed to be active on any given Compute Unit. Only one such work-group is allowed to be active on any given CU. This field is set to 0 by the Finalizer and must be filled in by CP.
| 25 | 1 bit | cdbg\_user | This field is set to 0 by the Finalizer and must be filled in by CP. |
| 31:26 | 6 bits | reserved | Reserved. Must be 0. |
| 32 | | | Total size 4 bytes. |

### Compute shader program settings 2 amd_compute_pgm_rsrc2_t

The fields of amd_compute_pgm_rsrc2 are used by CP to set up COMPUTE\_PGM\_RSRC2.

| **Bits** | **Size** | **Field Name** | **Description** |
| --- | --- | --- | --- |
| 0 | 1 bit | enable\_sgpr\_private\_segment\_wave\_byte\_offset | Enable the setup of the SGPR wave scratch offset system register (see 2.9.8). Used by CP to set up COMPUTE\_PGM\_RSRC2.SCRATCH\_EN. |
| 5:1 | 5 bits | user\_sgpr\_count | The total number of SGPR user data registers requested. This number must match the number of user data registers enabled. |
| 6 | 1 bit | enable\_trap\_handler | Code contains a TRAP instruction which requires a trap hander to be enabled. Used by CP to set up COMPUTE\_PGM\_RSRC2.TRAP\_PRESENT. Note that CP shuld set COMPUTE\_PGM\_RSRC2.TRAP\_PRESENT if either this field is 1 or if amd\_queue.enable\_trap\_handler is 1 for the queue executing the kernel dispatch. |
| 7 | 1 bit | enable\_sgpr\_workgroup\_id\_x | Enable the setup of [Work-Group Id X](#initial-kernel-register-state). Also used by CP to set up COMPUTE\_PGM\_RSRC2.TGID\_X\_EN. |
| 8 | 1 bit | enable\_sgpr\_workgroup\_id\_y | Enable the setup of [Work-Group Id Y](#initial-kernel-register-state). Also used by CP to set up COMPUTE\_PGM\_RSRC2.TGID\_Y\_EN, TGID\_Z\_EN.|
| 9 | 1 bit | enable\_sgpr\_workgroup\_id\_z | Enable the setup of [Work-Group Id Z](#initial-kernel-register-state). Also used by CP to set up COMPUTE\_PGM\_RSRC2. TGID\_Z\_EN. |
| 10 | 1 bit | enable\_sgpr\_workgroup\_info |  Enable the setup of [Work-Group Info](#initial-kernel-register-state). |
| 12:11 | 2 bits | enable\_vgpr\_workitem\_id | Enable the setup of [Work-Item Id X, Y, Z](#initial-kernel-register-state). Also used by CP to set up COMPUTE\_PGM\_RSRC2.TIDIG\_CMP\_CNT. |
| 13 | 1 bit | enable\_exception\_address\_watch | Wavefront starts execution with specified exceptions enabled. Used by CP to set up COMPUTE\_PGM\_RSRC2.EXCP\_EN\_MSB (composed from following bits). Address Watch - TC (L1) has witnessed a thread access an "address of interest". |
| 14 | 1 bit | enable\_exception\_memory\_violation  | Memory Violation - a memory violation has occurred for this wave from L1 or LDS (write-to-read-only-memory, mis-aligned atomic, LDS address out of range, illegal address, etc.). |
| 23:15 | 9 bits | granulated\_lds\_size | Amount of group segment (LDS) to allocate for each work-group. Granularity is device specific. CP should use the rounded value from the dispatch packet, not this value, as the dispatch may contain dynamically allocated group segment memory. This field is set to 0 by the Finalizer and CP will write directly to COMPUTE\_PGM\_RSRC2.LDS\_SIZE. |
| 24 | 1 bit | enable\_exception\_ieee\_754\_fp\_invalid\_operation | Enable IEEE 754 FP Invalid Operation exception at start of wavefront execution. enable_exception flags are used by CP to set up COMPUTE\_PGM\_RSRC2.EXCP\_EN (set from bits 0..6), EXCP\_EN\_MSB (set from bits 7..8). |
| 25 | 1 bit | enable\_exception\_fp\_denormal\_source | Enable FP Denormal exception at start of wavefront execution. |
| 26 | 1 bit | enable\_exception\_ieee\_754\_fp\_division\_by\_zero | Enable IEEE 754 FP Division by Zero exception at start of wavefront execution. |
| 27 | 1 bit | enable\_exception\_ieee\_754\_fp\_overflow | Enable IEEE 754 FP FP Overflow exception at start of wavefront execution. |
| 28 | 1 bit | enable\_exception\_ieee\_754\_fp\_underflow | Enable IEEE 754 FP Underflow exception at start of wavefront execution. |
| 29 | 1 bit | enable\_exception\_ieee\_754\_fp\_inexact | Enable IEEE 754 FP Inexact exception at start of wavefront execution. |
| 30 | 1 bit | enable\_exception\_int\_divide\_by\_zero | Enable Integer Division by Zero (rcp\_iflag\_f32 instruction only) exception at start of wavefront execution. |
| 31 | 1 bit | | Reserved. Must be 0. |
| 32 | | | Total size 4 bytes. |

### AMD Machine Kind amd_machine_kind_t

| **Enumeration Name** | **Value** | **Description** |
| --- | --- | --- |
| AMD\_MACHINE\_KIND\_UNDEFINED | 0 | Machine kind is undefined. |
| AMD\_MACHINE\_KIND\_AMDGPU | 1 | Machine kind is AMD GPU. Corresponds to AMD GPU ISA architecture of AMDGPU. |

### Float Round Mode amd_float_round_mode_t

| **Enumeration Name** | **Value** | **Description** |
| --- | --- | ---- |
| AMD\_FLOAT\_ROUND\_MODE\_NEAR\_EVEN | 0 | Round Ties To Even |
| AMD\_FLOAT\_ROUND\_MODE\_PLUS\_INFINITY | 1 | Round Toward +infinity |
| AMD\_FLOAT\_ROUND\_MODE\_MINUS\_INFINITY | 2 | Round Toward -infinity |
| AMD\_FLOAT\_ROUND\_MODE\_ZERO | 3 | Round Toward 0 |

### Denorm Mode amd_float_denorm_mode_t

| **Enumeration Name** | **Value** | **Description** |
| --- | --- | ---- |
| AMD\_FLOAT\_DENORM\_MODE\_FLUSH\_SRC\_DST | 0 | Flush Source and Destination Denorms |
| AMD\_FLOAT\_DENORM\_MODE\_FLUSH\_DST | 1 | Flush Output Denorms |
| AMD\_FLOAT\_DENORM\_MODE\_FLUSH\_SRC | 2 | Flush Source Denorms |
| AMD\_FLOAT\_DENORM\_MODE\_FLUSH\_NONE | 3 | No Flush |

## PCIe Gen3 Atomic Operations

PCI Express Gen3 defines 3 PCIe transactions, each of which carries out a specific Atomic Operation:
  * FetchAdd (Fetch and Add)
  * Swap (Unconditional Swap)
  * CAS (Compare and Swap)

For compute capabilities supporting PCIe Gen3 atomics, system scope atomic operations use the following sequences:
  * Atomic Load/Store: FLAT_LOAD_DWORD* / FLAT_STORE_DWORD* / TLP MRd / MWr
  * Atomic add: FLAT_ATOMIC_ADD / TLP FetchAdd
  * Atomic sub: FLAT_ATOMIC_ADD + negate/ TLP FetchAdd
  * Atomic swap: FLAT_ATOMIC_SWAP / TLP Swap
  * Atomic compare-and-swap: FLAT_ATOMIC_CMPSWAP / TLP CAS
  * Other Atomic RMW operations: (max, min, and, or, xor, wrapinc, wrapdec): CAS loop

PCIe Gen3 atomics are only supported on certain hardware configurations, for example, Haswell system.

## AMD Queue

### HSA AQL Queue Object hsa_queue_t

HSA Queue Object is defined in "HSA Platform System Architecture Specification". AMD HSA Queue handle is a pointer to amd_queue_t.

### AMD AQL Queue Object amd_queue_t

The AMD HSA Runtime implementation uses the AMD Queue object (amd_queue_t) to implement AQL queues. It begins with the HSA Queue object, and then has additional information contiguously afterwards that is AMD device specific. The AMD device specific information is accessible by the AMD HSA Runtime, CP and kernel machine code.

The AMD Queue object must be allocated on 64 byte alignment. This allows CP microcode to fetch fields using cache line addresses. The entire AMD Queue object must not span a 4GiB boundary. This allows CP to save a few instructions when calculating the base address of amd_queue_t from &(amd_queue_t.read_dispatch_id) and amd_queue_t.read_dispatch_id_field_base_offset.

For GFX8 and earlier systems, only HSA Queue type SINGLE is supported.

| **Bits** | **Size** | **Name** | **Description** |
| --- | --- | --- | --- |
| 319:0 | 40 bytes | hsa_queue | HSA Queue object |
| 447:320 | 16 bytes | | Unused. Allows hsa\_queue\_t to expand but still keeps write\_dispatch\_id, which is written by the producer (often the host CPU), in the same cache line. Must be 0. |
| 511:448 | 8 bytes | write\_dispatch\_id | 64-bit index of the next packet to be allocated by application or user-level runtime. Initialized to 0 at queue creation time. |
| 512 | | | Start of cache line for fields accessed by kernel machine code isa. |
| 543:512 | 4 bytes | group\_segment\_aperture\_base\_hi | For HSA64, the most significant 32 bits of the 64 bit group segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:13], 29b0}. For HSA32, the 32 bits of the 32 bit group segment flat address aperture. This is the same value as {SH\_MEM\_BASES:SHARED\_BASE[15:0], 16b0}. |
| 575:544 | 4 bytes | private\_segment\_aperture\_base\_hi | For HSA64, the most significant 32 bits of the 64 bit private segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:13], 28b0, 1b1} For HSA32, the 32 bits of the 32 bit private segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:0], 16b0}. |
| 607:576 | 4 bytes | max\_cu\_id | The number of compute units on the agent to which the queue is associated. |
| 639:608 | 4 bytes | max\_wave\_id | The number of wavefronts that can be executed on a single compute unit of the device to which the queue is associated. |
| 703:640 | 8 bytes | max\_legacy\_doorbell\_dispatch\_id\_plus\_1 | For AMD_SIGNAL_KIND_LEGACY_DOORBELL, maximum value of write_dispatch_id signaled for the queue. This value is always 64-bit and never decreases. |
| 735:704 | 4 bytes | legacy\_doorbell\_lock | For AMD_SIGNAL_KIND_LEGACY_DOORBELL, atomic variable used to protect critical section which updates the doorbell related fields. Initialized to 0, and set to 1 to lock the critical section. |
| 1023:736 | 36 bytes | | Padding to next cache line. Unused and must be 0. |
| 1024 | | | Start of cache line for fields accessed by the packet processor (CP micro code). |
| 1087:1024 | 8 bytes | read_dispatch_id | 64-bit index of the next packet to be consumed by compute unit hardware. Initialized to 0 at queue creation time. |
| 1119:1088 | 4 bytes | read_dispatch_id_field_base_byte_offset | Byte offset from the base of hsa_queue_t to the read_dispatch_id field. The CP microcode uses this and CP_HQD_PQ_RPTR_REPORT_ADDR[_HI] to calculate the base address of hsa_queue_t when amd_kernel_code_t.enable_sgpr_dispatch_ptr is set. This field must immediately follow read_dispatch_id. This allows the layout above the read_dispatch_id field to change, and still be able to get the base of the hsa_queue_t, which is needed to return if amd_kernel_code_t.enable_sgpr_queue_ptr is requested. These fields are defined by HSA Foundation and so could change. CP only uses fields below read_dispatch_id which are defined by AMD. |
| 1536 | | | Start of next cache line for fields not accessed under normal conditions by the packet processor (CP micro code). These are kept in a single cache line to minimize memory accesses performed by CP micro code. |
| 2048 | | | Total size 256 bytes. |

### Queue operations

A queue has an associated set of high-level operations defined in "HSA Runtime Specification" (API functions in host code) and "HSA Programmer Reference Manual Specification" (kernel code).

The following is informal description of AMD implementation of queue operations (all use memory scope system, memory order applies):
  * Load Queue Write Index: Atomic load of read_dispatch_id field
  * Store Queue Write Index: Atomic store of read_dispatch_id field
  * Load Queue Read Index: Atomic load of write_dispatch_id field
  * Store Queue Read Index: Atomic store of read_dispatch_id field
  * Add Queue Write Index: Atomic add of write_dispatch_id field
  * Compare-And-Swap Queue Write Index: Atomic CAS of write_dispatch_id field

## Signals
### Signals overview
Signal handle is 8 bytes. AMD signal handle is a pointer to AMD Signal Object (amd_signal_t).

The following operations are defined on HSA Signals:
  * Signal Load
    * Read the of the current value of the signal
    * Optional acquire semantics on the signal value
  * Signal Wait on a condition
    * Blocks the thread until the requested condition on the signal value is observed
    * Condition: equals, not-equals, greater, greater-equals, lesser, lesser-equals
    * Optional acquire semantics on the signal value
    * Returns the value of the signal that caused it to wake
  * Signal Store
    * Optional release semantics on the signal value
  * Signal Read-Modify-Write Atomics (add, sub, increment, decrement, min, max, and, or, xor, exch, cas)
    * These happen immediately and atomically
    * Optional acquire-release semantics on the signal value

### Signal kind amd_signal_kind_t

| **ID** | **Name** | **Description** |
| --- | --- | --- |
| 0 | AMD_SIGNAL_KIND_INVALID | An invalid signal. |
| 1 | AMD_SIGNAL_KIND_USER | A regular signal |
| -1 | AMD_SIGNAL_KIND_DOORBELL | Doorbell signal with hardware support |
| -2 | AMD_SIGNAL_KIND_LEGACY_DOORBELL | Doorbell signal with hardware support, legacy (GFX7/GFX8) |

### Signal object amd_signal_t

An AMD Signal object must always be 64 byte aligned to ensure it cannot span a page boundary. This is required by CP microcode which optimizes access to the structure by only doing a single SUA (System Uniform Address) translation when accessing signal fields. This optimization is used in GFX8.

| **Bits** | **Size** | **Name** | **Description** |
| --- | --- | --- | --- |
| 63:0 | 8 bytes | kind | [Signal kind](#signal-kind-amd_signal_kind_t) |
| 127:64 | 8 bytes | value | For AMD_SIGNAL_KIND_USER: signal payload value. In small machine model only the lower 32 bits is used, in large machine model all 64 bits are used. |
| 127:64 | 8 bytes | legacy_hardware_doorbell_ptr | For AMD_SIGNAL_KIND_LEGACY_DOORBELL: pointer to the doorbell IOMMU memory (write-only). Used for hardware notification in [Signal Store](#signal-kernel-machine-code). |
| 127:64 | 8 bytes | hardware_doorbell_ptr | For AMD_SIGNAL_KIND_DOORBELL: pointer to the doorbell IOMMU memory(write-only). Used for hardware notification in [Signal Store](#signal-kernel-machine-code). |
| 191:128 | 8 bytes | event_mailbox_ptr | For AMD_SIGNAL_KIND_USER: mailbox address for event notification in [Signal operations](#signal-kernel-machine-code). |
| 223:192 | 4 bytes | event_id | For AMD_SIGNAL_KIND_USER: event id for event notification in [Signal operations](#signal-kernel-machine-code). |
| 255:224 | 4 bytes | | Padding. Must be 0. |
| 319:256 | 8 bytes | start_ts | Start of the AQL packet timestamp, when profiled. |
| 383:320 | 8 bytes | end_ts | End of the AQL packet timestamp, when profiled. |
| 448:384 | 8 bytes | queue\_ptr | For AMD\_SIGNAL\_KIND\_\*DOORBELL: the address of the associated [amd\_queue\_t](#amd-queue), otherwise reserved and must be 0. |
| 511:448 | 8 bytes | | Padding to 64 byte size. Must be 0. |
| 512 | | | Total size 64 bytes |

### Signal kernel machine code

As signal kind is determined by kind field of amd_signal_t, instruction sequence for signal operation must branch on signal kind.

The following is informal description of signal operations:
* For AMD_SIGNAL_KIND_USER kind:
  * Signal Load uses atomic load from value field of corresponding amd_signal_t (memory order applies, memory scope system).
  * Signal Wait
    * Uses poll loop on signal value.
    * s_sleep ISA instruction provides hint to the SQ to not to schedule the wave for a specified time.
    * s_memtime/s_memrealtime instruction is used to measure time (as signal wait is required to time out in reasonable time interval even if condition is not met).
  * Signal Store/Signal Atomic uses the following sequence:
    * Corresponding atomic operation on signal value (memory scope system, memory order applies).
    * Load mailbox address from event_mailbox_ptr field.
    * If mailbox address is not zero:
      * load event id from event_id field.
      * atomic store of event id to mailbox address (memory scope system, memory order release).
      * s_sendmsg with argument equal to lower 8 bits of event_id.
* For AMD_SIGNAL_KIND_LEGACY_DOORBELL:
  * Signal Store uses the following sequence:
    * Load queue address from queue_ptr field
    * Acquire spinlock protecting the legacy doorbell of the queue.
      * Load address of the spinlock from legacy_doorbell_lock field of amd_queue_t.
      * Compare-and-swap atomic loop, previous value 0, value to set 1 (memory order acquire, memory scope system).
      * s_sleep ISA instruction provides hint to the SQ to not to schedule the wave for a specified time.
    * Use value+1 as next packet index and initial value for legacy dispatch id. GFX7/GFX8 hardware expects packet index to point beyond the last packet to be processed.
    * Atomic store of next packet index (value+1) to max_legacy_doorbell_dispatch_id_plus_1 field (memory order relaxed, memory scope system).
    * For small machine model:
      * legacy_dispatch_id = min(write_dispatch_id, read_dispatch_id + hsa_queue.size)
    * For GFX7:
      * Load queue size from hsa_queue.size field of amd_queue_t.
      * Wrap packet index to a point within the ring buffer (ring buffer size is twice the size of the HSA queue).
      * Convert legacy_dispatch_id to DWORD count by multiplying by 64/4 = 16.
      * legacy_dispatch_id = (legacy_dispatch_id & ((hsa_queue.size << 1)-1)) << 4;
    * Store legacy dispatch id to the hardware MMIO doorbell.
      * Address of the doorbell is in legacy_hardware_doorbell_ptr field of amd_signal_t.
    * Release spinlock protecting the legacy doorbell of the queue. Atomic store of value 0.
  * Signal Load/Signal Wait/Signal Read-Modify-Write Atomics are not supported. Instruction sequence for these operations and this signal kind is empty.


## Debugtrap

Debugtrap halts execution of the wavefront and generates debug exception. For more information, refer to "HSA Programmer Reference Manual Specification". debugtrap accepts 32-bit unsigned value as an argument.

The following is a description of debugtrap sequence:
  * v0 contains 32-bit argument of debugtrap
  * s[0:1] contains Queue Ptr for the dispatch
  * s_trap 0x1 


## References

  * [HSA Standards and Specifications](http://www.hsafoundation.com/standards/)
   * [HSA Platform System Architecture Specification 1.0](http://www.hsafoundation.com/?ddownload=4944)
   * [HSA Programmer Reference Manual Specification 1.01](http://www.hsafoundation.com/?ddownload=4945)
   * [HSA Runtime Specification 1.0](http://www.hsafoundation.com/?ddownload=4946)
  * [AMD ISA Documents](http://developer.amd.com/resources/documentation-articles/developer-guides-manuals/)
    * [AMD GCN3 Instruction Set Architecture (2015)](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/07/AMD_GCN3_Instruction_Set_Architecture.pdf)
    * [AMD_Southern_Islands_Instruction_Set_Architecture](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/07/AMD_Southern_Islands_Instruction_Set_Architecture1.pdf)
  * [ROCR Runtime sources](https://github.com/RadeonOpenCompute/ROCR-Runtime)
    * [amd_hsa_kernel_code.h](https://github.com/RadeonOpenCompute/ROCR-Runtime/blob/master/src/inc/amd_hsa_kernel_code.h)
    * [amd_hsa_queue.h](https://github.com/RadeonOpenCompute/ROCR-Runtime/blob/master/src/inc/amd_hsa_queue.h)
    * [amd_hsa_signal.h](https://github.com/RadeonOpenCompute/ROCR-Runtime/blob/master/src/inc/amd_hsa_signal.h)
    * [amd_hsa_common.h](https://github.com/RadeonOpenCompute/ROCR-Runtime/blob/master/src/inc/amd_hsa_common.h)
  *  [PCI Express Atomic Operations](https://www.pcisig.com/specifications/pciexpress/specifications/ECN_Atomic_Ops_080417.pdf)
