## AMDGPU Compute Application Binary Interface

Version 0.40 (March 2016)

### Introduction

This specification defines the application binary interface (ABI) provided by the AMD implementation of the HSA runtime for AMD GPU architecture agents. The AMD GPU architecture is a family of GPU agents which differ in machine code encoding and functionality.


### Entity definitions

#### Instruction set architecture (ISA)

AMDGPU ISA specifies instruction set architecture and capabilities used by machine code. It consists of several fields:
  * Vendor ("AMD")
  * Architecture ("AMDGPU")
  * Major (GFXIP), minor and stepping versions

These fields may be combined to form one defining string, for example, "AMD:AMDGPU:8:0:0".

| **Vendor** | **Architecture** | **Major** | **Minor** | **Stepping** | **Comments** | **Examples** |
| --- | --- | --- | --- | --- | --- | --- |
| AMD | AMDGPU | 7 | 0 | 0 | GFX7, 116 double FP | Kaveri, Bonaire |
| AMD | AMDGPU | 7 | 0 | 1 | GFX7, 1/2 double FP | Hawaii |
| AMD | AMDGPU | 8 | 0 | 0 | GFX8, SPI register limitation, -XNACK | Iceland, Tonga |
| AMD | AMDGPU | 8 | 0 | 1 | GFX8, +XNACK | Carrizo |
| AMD | AMDGPU | 8 | 0 | 2 | GFX8, SPI register limitation, -XNACK, PCIe Gen3 atomics | Boltzmann Tonga |
| AMD | AMDGPU | 8 | 0 | 3 | GFX8, -XNACK, PCIe Gen3 atomics | Boltzmann Fiji |
| AMD | AMDGPU | 8 | 0 | 4 | GFX8, -XNACK | Fiji |
| AMD | AMDGPU | 8 | 1 | 0 | GFX8, +XNACK | Stoney |

#### Queues

##### HSA AQL Queue Object (hsa_queue_t)

HSA Queue Object is defined in __HSA Platform System Architecture Specification__

##### AMD AQL Queue Object (amd_queue_t)

The AMD HSA Runtime implementation uses the AMD Queue object (amd_queue_t) to implement AQL queues. It begins with the HSA Queue object, and then has additional information contiguously afterwards that is AMD device specific. The AMD device specific information is accessible by the AMD HSA Runtime, CP and kernel machine code.

The AMD Queue object must be allocated on 64 byte alignment. Allows CP microcode to fetch fields using cache line addresses. The entire AMD Queue object must not span a 4GiB boundary. This allows CP to save a few instructions when calculating the base address of amd_queue_t from &(amd_queue_t.read_dispatch_id) and amd_queue_t.read_dispatch_id_field_base_offset.

For GFX8 and earlier systems, only HSA Queue type SINGLE is supported.

| **Bits** | **Size** | **Name** | **Description** |
| --- | --- | --- | --- |
| 319:0 | 40 bytes | hsa_queue | HSA Queue object |
| 447:320 | 16 bytes | | Unused. Allows hsa\_queue\_t to expand but still keeps write\_dispatch\_id, which is written by the producer (often the host CPU), in the same cache line. Must be 0. |
| 511:448 | 8 bytes | write\_dispatch\_id | A 64-bit unsigned integer. On GFX8, specifies the Dispatch ID of the next AQL packet to be allocated by the application or user-level runtime. On GFX7, hsa\_queue.base\_address + ((write\_dispatch\_id % hsa\_queue.size) \* 64 /\* AQL packet size \*/) is the virtual addressfor the next AQL packet allocated. Initialized to 0 at queue creation time. Note: On pre-GFX8 (eg Kaveri), the write\_dispatch\_id is specified in dwords and must be divided by (AQL packet\_size of 64 / dword size of 4)=16 to obtain the HSA "packet-granularity" write offset. Therefore, hsa\_queue.base\_address + (((write\_dispatch\_id/16) % hsa\_queue.size) \* 64 /\* AQL packet size \*/) is the virtual address for the next AQL packet allocated. Finalizer and Runtime software is responsible for converting dword offsets to packet granularity. |
| 512 | | | Start of cache line for fields accessed by kernel machine code isa |
| 543:512 | 4 bytes | group\_segment\_aperture\_base\_hi | For HSA64, the most significant 32 bits of the 64 bit group segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:13], 29b0}. For HSA32, the 32 bits of the 32 bit group segment flat address aperture. This is the same value as {SH\_MEM\_BASES:SHARED\_BASE[15:0], 16b0}. Used in kernel machine code to implement HSAIL stof\_group. |
| 575:544 | 4 bytes | private\_segment\_aperture\_base\_hi | For HSA64, the most significant 32 bits of the 64 bit private segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:13], 28b0, 1b1} For HSA32, the 32 bits of the 32 bit private segment flat address aperture base. This is the same value as {SH\_MEM\_BASES:PRIVATE\_BASE[15:0], 16b0}. Used in kernel machine code to implement HSAIL stof\_private instruction.
| 607:576 | 4 bytes | max\_cu\_id | The number of compute units 1 on the agent to which the queue is associated. Used in kernel machine code to implement HSAIL maxcuid instruction. |
| 639:608 | 4 bytes | max\_wave\_id | The number of wavefronts 1 that can be executed on a single compute unit of the device to which the queue is associated. Used in kernel machine code to implement HSAIL maxwaveid instruction. |
| 703:640 | 8 bytes | max\_legacy\_doorbell\_dispatch\_id\_plus\_1 |Must be initialized to 0 at queue creation time. For queues attached to GFX8 and earlier hardware, it is necessary to prevent backwards doorbells in the software signal operation. This field is used to hold the maximum doorbell dispatch id [plus 1] signaled for the queue. The hardware will monitor this field, and not write\_dispatch\_id, on queue connect. The value written to this field is required to be made visible before writing to the queue's doorbell signal (referenced by hsa\_queue.doorbell\_signal) hardware location (referenced by hardware\_doorbell\_ptr or legacy\_hardware\_doorbell\_ptr). The value is always 64 bit, even in small machine model. |
| 735:704 | 4 bytes | legacy\_doorbell\_lock | Must be initialized to 0 at queue creation time. For queues attached to GFX8 and earlier hardware, it is necessary to use a critical section to update the doorbell related fields of amd\_queue\_s.max\_legacy\_doorbell\_dispatch\_id\_plus\_1 and amd\_signal\_s.legacy\_hardware\_doorbell\_ptr. This field is initialized to 0, and set to 1 to lock the critical section. |
| 1023:736 | 36 bytes | | Unused. Must be 0. If additional space is required for the fields accessed by the kernel isa, then this reserved space can be used, and space for an additional cache line(s) can also be added and not break backwards compatibility for CP micro code as the read\_dispatch\_id\_field\_base\_offset field below can be initialized with the correct offset to get to the read\_dispatch\_id field below. If the CP micro code fields need to be expanded that is possible as there are no fields after them. This allows both the kernel machine code fields and CP micro code accessed fields to expand in a backwards compatible way. |
| 1024 | | | Start of cache line for fields accessed by the packet processor (CP micro code). These are kept in a single cache line to minimize memory accesses performed by CP micro code. |
| 1536 | | | Start of next cache line for fields not accessed under normal conditions by the packet processor (CP micro code). These are kept in a single cache line to minimize memory accesses performed by CP micro code. |
| 2048 | | | Total size 256 bytes. |

#### Signals

Signal handle is 8 bytes. AMD signal handle is a pointer to AMD Signal Object (amd_signal_t).

The following operations are defined on HSA Signals:
  * Signal Load
    * Read the of the current value of the signal
    * Acquire semantics on the signal value
  * Signal Wait on a condition
    * Blocks the thread until the requested condition on the signal value is observed
    * Condition: equals, not-equals, greater, greater-equals, lesser, lesser-equals
    * Acquire semantics on the signal value
    * Returns the value of the signal that caused it to wake
  * Signal Store
    * Release semantics on the signal value
  * Signal Read-Modify-Write Atomics (add, sub, increment, decrement, min, max, and, or, xor, exch, cas)
    * These happen immediately and atomically
    * Acquire-Release semantics on the signal value

##### Signal operations in kernel

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
    * Use value+1 as packet index and initial value for legacy dispatch id.
    * Atomic store of packet index (value+1) to max_legacy_doorbell_dispatch_id_plus_1 field (memory order relaxed, memory scope system). AMD hardware expects packet index to point beyond the last packet to be processed.
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

Notes:
  * Although CZ provides us context-switching support, it is at a dispatch level granularity (and is also not selective from the ISA).

<a name="abcd"></a>
##### AMD Signal Kind (amd_signal_kind_t)

| **ID** | **Name** | **Description** |
| --- | --- | --- |
| 0 | AMD_SIGNAL_KIND_INVALID | An invalid signal. |
| 1 | AMD_SIGNAL_KIND_USER | A regular signal |
| -1 | AMD_SIGNAL_KIND_DOORBELL | Doorbell signal with hardware support | 
| -2 | AMD_SIGNAL_KIND_LEGACY_DOORBELL | Doorbell signal with hardware support, legacy (GFX8) |

##### AMD Signal Object (amd_signal_t)

An AMD Signal object must always be 64 byte aligned to ensure it cannot span a page boundary. This is required by CP microcode which optimizes access to the structure by only doing a single SUA (System Uniform Address) translation when accessing signal fields. This optimization is used in GFX8.

| **Bits** | **Size** | **Name** | **Description** |
| --- | --- | --- | --- |
| 63:0 | 8 bytes | kind | Signal kind. Refer to [this](#abcd) |
| 127:64 | 8 bytes | value | For AMD_SIGNAL_KIND_USER kind, the value is the signal payload value. In small machine model only the lower 32 bits is used, in large machine model all 64 bits are used. |
| 127:64 | 8 bytes | legacy_hardware_doorbell_ptr | For AMD_SIGNAL_LEGACY_DOORBELL kind, the value is a pointer to the doorbell memory. This is IOMMU memory and is only writable and cannot be read. |
| 127:64 | 8 bytes | hardware_doorbell_ptr | For AMD_SIGNAL_DOORBELL kind, the value is a pointer to the doorbell memory. This is IOMMU memory and is only writable and cannot be read. |
| 191:128 | 8 bytes | event_mailbox_ptr | For event based notification, forwarded by KFD. |
| 223:192 | 4 bytes | event_id | For event based notification, forwarded by KFD. | 
| 255:224 | 4 bytes | | Padding. Must be 0. |
| 319:256 | 8 bytes | start_ts | Start of the AQL packet timestamp, when profiled. |
| 383:320 | 8 bytes | end_ts | End of the AQL packet timestamp, when profiled. |
| 448:384 | 8 bytes | queue\_ptr | For AMD\_SIGNAL\_KIND\_\*DOORBELL, the address of the associated amd\_queue\_t. If the kind is not AMD_SIGNAL_KIND\_*DOORBELL then reserved and must be 0. This ensures union is 64 bits even for 32 bit applications that use 32-bit pointers. | 
| 511:448 | 8 bytes | | Padding to 64 byte size. Must be 0. |
| 512 | 64 bytes | | Total size |


### Terms and definitions

### References

  * [HSA Standards and Specifications](http://www.hsafoundation.com/standards/)
   * [HSA Platform System Architecture Specification 1.0](http://www.hsafoundation.com/?ddownload=4944)
   * [HSA Programmer Reference Manual Specification 1.01](http://www.hsafoundation.com/?ddownload=4945)
   * [HSA Runtime Specification 1.0](http://www.hsafoundation.com/?ddownload=4946)
  * [AMD ISA Documents](http://developer.amd.com/resources/documentation-articles/developer-guides-manuals/)
    * [AMD GCN3 Instruction Set Architecture (2015)](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/07/AMD_GCN3_Instruction_Set_Architecture.pdf)
    * [AMD_Southern_Islands_Instruction_Set_Architecture](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/07/AMD_Southern_Islands_Instruction_Set_Architecture1.pdf)