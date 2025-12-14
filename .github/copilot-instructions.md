# Vortex PoCL AI Coding Instructions

## Project Overview
This is the **Vortex GPGPU backend** for **PoCL (Portable Computing Language)**. It enables OpenCL support on Vortex RISC-V GPGPU hardware and simulators.
- **Core PoCL**: `lib/CL` (OpenCL API implementation).
- **Vortex Backend**: `lib/CL/devices/vortex`.
- **Build System**: CMake.

## Architecture & Data Flow

### 1. Device Driver (`lib/CL/devices/vortex`)
The Vortex driver interfaces PoCL with the Vortex runtime (`libvortex`).
- **`pocl-vortex.c`**: Main driver implementation.
  - `pocl_vortex_run`: Handles kernel execution. It packs arguments, uploads them and the kernel binary to the device, and triggers execution.
  - `pocl_vortex_alloc_mem_obj`: Allocates device memory using `vx_mem_alloc`.
  - `vortex_command_scheduler`: Simple FIFO scheduler for command execution.
- **`vortex_device_data_t`**: Holds device state (`vx_device_h`), command lists, and locks.

### 2. Compilation Pipeline (`lib/CL/devices/vortex/vortex_utils.cc`)
OpenCL kernels are compiled to Vortex binaries (`.vxbin`) through a multi-step process:
1.  **LLVM IR Transformation**: `processKernels` modifies kernels to accept a single `ArgBuffer` pointer instead of individual arguments.
2.  **Kernel Selection**: `addKernelSelect` generates `__vx_get_kernel_callback` for device-side kernel dispatch.
3.  **Linking**: The modified LLVM IR is linked with `kernel_main.c` (device-side runtime) using Clang.
4.  **Binary Generation**: The resulting ELF is converted to `.vxbin` using `vxbintool`.

### 3. Device-Side Runtime
- **`kernel_main.c`**: The entry point running on the Vortex device. It uses the `ArgBuffer` to unpack arguments and call the selected kernel.
- **`kernel_args.h`**: Defines the `kernel_args_t` structure used for argument passing between host and device.

## Critical Workflows

### Building
Use CMake with Vortex-specific flags:
```bash
cmake -DENABLE_VORTEX=ON \
      -DVORTEX_PREFIX=<path_to_vortex> \
      -DWITH_LLVM_CONFIG=<path_to_llvm_config> \
      ..
```

### Runtime Environment
The driver relies on specific environment variables during execution (often set by `vortex-pocl` setup scripts):
- `POCL_VORTEX_CFLAGS`: Compiler flags for the Vortex target (RISC-V).
- `POCL_VORTEX_LDFLAGS`: Linker flags.
- `POCL_VORTEX_BINTOOL`: Path to the tool that converts ELF to `.vxbin`.

### Debugging
- **Logging**: Use `POCL_MSG_PRINT_LLVM`, `POCL_MSG_ERR`, `POCL_MSG_WARN` macros.
- **Debug Flags**: Set `POCL_DEBUG=1` or specific categories (e.g., `POCL_DEBUG=vortex`) to enable runtime logging.

## Coding Conventions
- **Memory Management**: Always use `vx_mem_alloc` / `vx_mem_free` for device memory.
- **Argument Packing**: When modifying kernel execution, ensure `kernel_args_t` in `kernel_args.h` matches the packing logic in `pocl_vortex_run`.
- **LLVM Version**: The codebase supports multiple LLVM versions. Use `#if LLVM_MAJOR >= X` for version-specific API usage.
- **Error Handling**: Check `vx_err` return codes from Vortex runtime calls and use `POCL_ABORT` or return `CL_*` error codes as appropriate.

## Key Files
- `lib/CL/devices/vortex/pocl-vortex.c`: Host-side driver logic.
- `lib/CL/devices/vortex/vortex_utils.cc`: LLVM IR manipulation and binary generation.
- `lib/CL/devices/vortex/kernel_main.c`: Device-side kernel entry point.
- `lib/CL/devices/vortex/pocl-vortex.h`: Driver function prototypes.
