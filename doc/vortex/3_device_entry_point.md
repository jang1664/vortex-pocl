# 3. Device Entry Point (`kernel_main.c`)

The `lib/CL/devices/vortex/kernel_main.c` file is the "main" program that runs on the Vortex device. It is linked with every OpenCL kernel binary.

## Role

Unlike typical CPU programs where `main` is the start, in GPGPU context, this `main` function acts as the **kernel launcher** running on the device's control processor (or the initial thread). It is responsible for setting up the environment and spawning the worker threads.

## Code Analysis

```c
#include <vx_spawn.h>
#include <vx_print.h>
#include "kernel_args.h"

// Global variables accessible by work-item functions
int g_work_dim;
dim3_t g_global_offset;

// Dynamic local memory allocator
void* vx_local_alloc(uint32_t size) {
  return __local_mem(size);
}

// Forward declaration of the generated dispatcher
void* __vx_get_kernel_callback(int kernel_id);

int main(void) {
  // 1. Retrieve Argument Buffer
  // The host driver passes the pointer to the argument buffer via the MSCRATCH CSR.
  kernel_args_t* kargs = (kernel_args_t*)csr_read(VX_CSR_MSCRATCH);

  // 2. Initialize Global State
  // These globals are used by get_work_dim() and get_global_offset()
  g_work_dim = kargs->work_dim;
  for (int i = 0, n = kargs->work_dim; i < 3; i++) {
    g_global_offset.m[i] = (i < n) ? kargs->global_offset[i] : 0;
  }

  // 3. Calculate Argument Pointer
  // Skip the header (kernel_args_t) to point to the actual kernel arguments.
  uint32_t aligned_kernel_args_size = ALIGN_OFFSET(sizeof(kernel_args_t), sizeof(size_t));
  void* arg = (void*)((uint8_t*)kargs + aligned_kernel_args_size);

  // 4. Resolve Kernel Function
  // Call the compiler-generated switch function to get the function pointer.
  vx_kernel_func_cb kernel_func = (vx_kernel_func_cb)__vx_get_kernel_callback(kargs->kernel_id);

  // 5. Spawn Threads
  // Distribute the work across the GPU cores/warps/threads.
  // - kargs->num_groups: Grid size
  // - kargs->local_size: Block size
  // - kernel_func: The transformed kernel function (takes 'arg' as input)
  // - arg: The pointer to the packed argument buffer
  return vx_spawn_threads(kargs->work_dim, kargs->num_groups, kargs->local_size, kernel_func, arg);
}
```

## Key Concepts

*   **CSR (Control and Status Register)**: `VX_CSR_MSCRATCH` is used as a mailbox to pass the initial pointer from the host driver to the device firmware.
*   **`vx_spawn_threads`**: This is a Vortex runtime function (part of `libvortex`) that handles the hardware-specific details of distributing threads. It ensures that `kernel_func(arg)` is called for every work-item specified by the grid and block dimensions.
*   **`__local_mem`**: A Vortex intrinsic used to allocate shared memory within a workgroup.
