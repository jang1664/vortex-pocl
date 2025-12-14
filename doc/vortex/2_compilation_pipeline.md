# 2. Compilation Pipeline (`vortex_utils.cc`)

The `lib/CL/devices/vortex/vortex_utils.cc` file implements the specialized compilation pipeline required to transform standard OpenCL kernels (LLVM IR) into Vortex-compatible binaries (`.vxbin`).

## Overview

The Vortex architecture does not support standard OpenCL kernel calling conventions directly. Instead, it requires a unified entry point where all arguments are passed via a single pointer to a packed buffer. This file implements the LLVM passes and linking logic to achieve this.

## Key Functions

### 1. `processKernels` (LLVM IR Transformation)

This function iterates over all kernels in the module and transforms them.

*   **Signature Modification**:
    *   Original: `void kernel(int* a, int b, local int* c)`
    *   New: `void kernel_vortex(int8* ArgBuffer)`
*   **Argument Unpacking**:
    *   It creates a new function body.
    *   It injects LLVM IR instructions to calculate offsets and load arguments from `ArgBuffer`.
    *   **Scalar/Global Pointers**: Loaded directly from the buffer based on calculated offsets.
    *   **Local Memory**:
        *   Detects `__local` arguments.
        *   Injects a call to `vx_local_alloc(__local_size)` (implemented in `kernel_main.c`) to dynamically allocate shared memory.
        *   Calculates the pointer using the allocated base address and the offset provided in the argument buffer.
*   **Replacement**: The original kernel function is replaced by this new "vortex-style" function.

### 2. `addKernelSelect` (Kernel Dispatcher)

Since multiple kernels can exist in a single program, the device needs a way to call the correct one based on an ID.

*   **Generates `__vx_get_kernel_callback`**:
    *   Creates a new function: `void* __vx_get_kernel_callback(int kernel_id)`.
    *   Builds a `switch` statement using LLVM IR.
    *   Maps integer IDs (0, 1, 2...) to the function pointers of the transformed kernels (`kernel_vortex`).
    *   This function is called by `kernel_main.c` on the device to resolve the function pointer to execute.

### 3. `compile_vortex_program` (Build Orchestration)

This function manages the external tools required to produce the final binary.

1.  **Prepare LLVM Module**: Calls `processKernels` and `addKernelSelect` to modify the in-memory LLVM module.
2.  **Export Bitcode**: Writes the modified module to a temporary `.bc` file.
3.  **Link & Compile (Clang)**:
    *   Invokes the `clang` compiler (RISC-V target).
    *   Links the kernel bitcode with `kernel_main.c` (the device-side runtime).
    *   Uses flags from `POCL_VORTEX_CFLAGS` and `POCL_VORTEX_LDFLAGS`.
    *   Produces a standard RISC-V ELF executable.
4.  **Binary Conversion (vxbintool)**:
    *   Invokes `vxbintool` (path from `POCL_VORTEX_BINTOOL`).
    *   Converts the ELF file into a `.vxbin` file.
    *   The `.vxbin` format is what the Vortex hardware/simulator loads.

## Environment Variables

The compilation process relies on several environment variables:

*   `LLVM_PREFIX`: Path to the LLVM installation.
*   `POCL_VORTEX_CFLAGS`: Compiler flags (e.g., `-march=riscv32...`).
*   `POCL_VORTEX_LDFLAGS`: Linker flags (e.g., linker script location).
*   `POCL_VORTEX_BINTOOL`: Path to the `vxbintool` executable.

## Debugging

*   If `POCL_DEBUGGING_ON` is enabled:
    *   Dumps the intermediate LLVM IR to `program.ll`.
    *   Runs `llvm-objdump` on the generated ELF and saves it to `program.dump`.
    *   Prints the exact command lines used for `clang` and `vxbintool`.
