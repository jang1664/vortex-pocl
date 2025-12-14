# Vortex PoCL Implementation Overview

This document provides a technical overview of the Vortex backend implementation in PoCL, covering both the host-side driver and the device-side kernel library.

## 1. Host Driver & Compilation (`lib/CL/devices/vortex`)

This directory contains the PoCL driver that runs on the host CPU. It manages the Vortex device, handles memory allocation, compiles OpenCL kernels, and schedules execution.

### Key Components

*   **`pocl-vortex.c`**: The core driver implementation.
    *   **`pocl_vortex_run`**: The critical function for kernel execution.
        1.  **Argument Packing**: Calculates the total size required for kernel arguments and local memory. It packs all arguments into a single contiguous buffer. The buffer starts with a `kernel_args_t` header (defined in `kernel_args.h`), followed by the actual argument values.
        2.  **Memory Management**: Uses `vx_mem_alloc` to allocate device memory for arguments and `vx_copy_to_dev` to upload them.
        3.  **Binary Upload**: Uploads the compiled kernel binary (`.vxbin`) to the device using `vx_upload_kernel_file`.
        4.  **Execution**: Triggers execution via `vx_start` and waits for completion with `vx_ready_wait`.
    *   **`pocl_vortex_alloc_mem_obj`**: Implements `clCreateBuffer`. It allocates memory on the Vortex device using `vx_mem_alloc` and maps it to a `vortex_buffer_data_t` structure.

*   **`vortex_utils.cc`**: Handles the LLVM IR transformation and binary generation pipeline.
    *   **`processKernels`**: A pass that transforms standard OpenCL kernels into Vortex-compatible functions.
        *   **Signature Change**: Changes the function signature to accept a single `ArgBuffer` pointer (`i8*`).
        *   **Argument Unpacking**: Injects code to load original arguments from the `ArgBuffer` based on calculated offsets.
        *   **Local Memory**: Handles `__local` arguments by calling `vx_local_alloc` dynamically.
    *   **`addKernelSelect`**: Generates a helper function `__vx_get_kernel_callback(int kernel_id)` that acts as a switch-case dispatcher, returning the function pointer for a given kernel ID.
    *   **`compile_vortex_program`**: Orchestrates the compilation flow:
        1.  **LLVM IR**: The transformed module is saved as a `.bc` file.
        2.  **Linking**: Clang links the `.bc` file with `kernel_main.c` (device entry point) to produce an ELF binary.
        3.  **Binary Conversion**: `vxbintool` converts the ELF to a `.vxbin` file executable by Vortex.

*   **`kernel_main.c`**: The entry point code that runs on the Vortex device (linked into every kernel binary).
    *   **`main()`**:
        1.  Reads the `kernel_args_t` structure pointer from the `VX_CSR_MSCRATCH` CSR.
        2.  Initializes global variables like `g_work_dim` and `g_global_offset`.
        3.  Resolves the target kernel function using `__vx_get_kernel_callback`.
        4.  Launches the kernel across the GPU using `vx_spawn_threads`.

*   **`kernel_args.h`**: Defines the data structure shared between the host driver and the device runtime.
    ```c
    typedef struct {
      uint32_t work_dim;
      uint32_t num_groups[3];
      uint32_t local_size[3];
      uint32_t global_offset[3];
      uint32_t kernel_id;
    } kernel_args_t;
    ```

## 2. Device Kernel Library (`lib/kernel/vortex`)

This directory contains the implementation of OpenCL built-in functions for the Vortex architecture. These files are compiled and linked with the user's kernel.

### Key Implementations

*   **`workitems.c`**: Implements OpenCL work-item query functions by mapping them to Vortex hardware intrinsics.
    *   `get_global_id`, `get_local_id`, `get_group_id`: Mapped to `blockIdx`, `threadIdx`, `blockDim`, `gridDim` (CUDA-like terminology used in Vortex runtime).
    *   `get_work_dim`: Returns the global `g_work_dim` set by `kernel_main.c`.

*   **`barrier.c`**: Implements synchronization primitives.
    *   **`barrier()`**: Implemented using `vx_barrier()` for workgroup synchronization.
    *   **Memory Fences**: Uses `vx_fence()` when `CLK_GLOBAL_MEM_FENCE` is specified.

*   **`printf.c`**: Implements device-side printing.
    *   Redirects `printf` calls to the Vortex runtime's `vx_vprintf`.

## Summary of Execution Flow

1.  **Host**: `pocl_vortex_run` packs args into a buffer.
2.  **Host**: Uploads args and `.vxbin` to device.
3.  **Host**: Calls `vx_start`.
4.  **Device**: `kernel_main` starts.
5.  **Device**: Reads args pointer from CSR.
6.  **Device**: `vx_spawn_threads` distributes work.
7.  **Device**: Individual threads execute the transformed kernel (unpacking args from the buffer).
8.  **Device**: Threads use `lib/kernel/vortex` functions for ID queries and synchronization.
