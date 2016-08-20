# ROCm - AMDGPU Compute Application Binary Interface


Table of Contents

- Introduction
- Finalizer, Code Object, Executable and Loader
- Kernel dispatch
- Hardware registers setup
- Initial kernel register state
- Kernel prolog code
- Global/Readonly/Kernarg segments
- Scratch memory swizzling
- Flat addressing
- Flat scratch
- M0 Register
- Dynamic call stack
- Memory model
- Memory model overview
- Memory operation constraints for global segment
- Memory operation constraints for group segment
- Memory fence constraints
- Instruction set architecture
- AMD Kernel Code
- AMD Kernel Code Object amd_kernel_code_t
- Compute shader program settings 1 amd_compute_pgm_rsrc1_t
- Compute shader program settings 2 amd_compute_pgm_rsrc2_t
- AMD Machine Kind amd_machine_kind_t
- Float Round Mode amd_float_round_mode_t
- Denorm Mode amd_float_denorm_mode_t
- PCIe Gen3 Atomic Operations
- AMD Queue
- HSA AQL Queue Object hsa_queue_t
- AMD AQL Queue Object amd_queue_t
- Queue operations
- Signals
- Signals overview
- Signal kind amd_signal_kind_t
- Signal object amd_signal_t
- Signal kernel machine code
- Debugtrap
- References
