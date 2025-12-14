# 1. Host Driver Core (`pocl-vortex.c`)

The `lib/CL/devices/vortex/pocl-vortex.c` file implements the PoCL device driver interface for the Vortex GPGPU. It acts as the bridge between the OpenCL runtime (running on the host CPU) and the Vortex hardware/simulator.

## Key Responsibilities

1.  **Device Initialization & Discovery**: Probing the Vortex device and querying its capabilities.
2.  **Memory Management**: Allocating and managing memory on the device.
3.  **Program Compilation**: Orchestrating the compilation of OpenCL kernels to Vortex binaries.
4.  **Command Execution**: Scheduling and executing commands (kernel runs, memory transfers).

## Detailed Analysis

### 1. Device Initialization (`pocl_vortex_init`)

This function initializes the Vortex device driver instance.

*   **Configuration**: Reads environment variables like `POCL_VORTEX_XLEN` to determine if the target is 32-bit or 64-bit RISC-V.
*   **Device Probing**: Calls `vx_dev_open` to connect to the Vortex runtime.
*   **Capability Query**: Uses `vx_dev_caps` to fetch hardware parameters:
    *   `VX_CAPS_NUM_CORES`: Number of compute units.
    *   `VX_CAPS_GLOBAL_MEM_SIZE`: Total global memory.
    *   `VX_CAPS_LOCAL_MEM_SIZE`: Local memory per workgroup.
    *   `VX_CAPS_NUM_WARPS` & `VX_CAPS_NUM_THREADS`: Used to calculate `max_work_group_size`.
*   **PoCL Device Struct**: Populates `cl_device_id` fields (e.g., `llvm_target_triplet`, `address_bits`) to inform the upper PoCL layers about the device properties.

### 2. Memory Management

*   **`pocl_vortex_alloc_mem_obj`**:
    *   Called when `clCreateBuffer` is used.
    *   Allocates device memory using `vx_mem_alloc`.
    *   Stores the device pointer and handle in a `vortex_buffer_data_t` structure attached to the `cl_mem` object.
    *   Handles `CL_MEM_COPY_HOST_PTR` by immediately copying data using `vx_copy_to_dev`.
*   **`pocl_vortex_free`**: Frees device memory using `vx_mem_free`.
*   **`pocl_vortex_write` / `pocl_vortex_read`**: Implements `clEnqueueWriteBuffer` and `clEnqueueReadBuffer` using `vx_copy_to_dev` and `vx_copy_from_dev`.

### 3. Program Compilation (`pocl_vortex_post_build_program`)

This function is triggered after the generic LLVM IR compilation.

*   **Pass Execution**: Runs standard PoCL LLVM passes (`pocl_llvm_run_passes_on_program`).
*   **Vortex Compilation**: Calls `compile_vortex_program` (defined in `vortex_utils.cc`) to convert the LLVM IR into a `.vxbin` executable.
*   **Metadata**: Stores kernel names and counts in `vortex_program_data_t` for later lookup.

### 4. Kernel Execution (`pocl_vortex_run`)

This is the most critical function, handling the actual execution of a kernel.

1.  **Argument Analysis**: Iterates through kernel arguments to calculate the total size required for the argument buffer.
    *   Includes space for `kernel_args_t` header.
    *   Includes space for scalar arguments, pointers, and local memory allocations.
    *   Aligns offsets based on architecture (32-bit or 64-bit).
2.  **Occupancy Check**: Verifies if the requested local memory usage fits within the device limits (`vx_check_occupancy`).
3.  **Argument Packing**:
    *   Allocates a host buffer (`host_kargs_base_ptr`).
    *   Writes the `kernel_args_t` header (work dimensions, group counts, etc.).
    *   Iterates through arguments again, writing values to the buffer.
        *   **Pointers**: Writes the device address (`buf_address`) obtained from `vortex_buffer_data_t`.
        *   **Local Memory**: Writes the size and offset for dynamic local memory allocation.
        *   **Scalars**: Writes the raw value.
4.  **Upload**:
    *   Allocates a device buffer for arguments (`vx_kargs_buffer`).
    *   Copies the packed argument buffer to the device.
    *   Uploads the kernel binary (`.vxbin`) if not already loaded (`vx_upload_kernel_file`).
5.  **Execution**:
    *   Calls `vx_start` to begin execution on the device.
    *   Calls `vx_ready_wait` to block until execution completes.
6.  **Cleanup**: Frees the argument buffer.

### 5. Command Scheduling

*   **`pocl_vortex_submit`**: Adds a command node to the ready list.
*   **`vortex_command_scheduler`**: A simple FIFO scheduler that picks commands from the ready list and executes them. Currently, it executes commands synchronously (blocking).

## Data Structures

*   **`vortex_device_data_t`**:
    *   `vx_device_h vx_device`: Handle to the Vortex device.
    *   `vx_buffer_h vx_kernel_buffer`: Handle to the currently loaded kernel binary.
*   **`vortex_buffer_data_t`**:
    *   `uint64_t buf_address`: Physical/Virtual address on the device.
    *   `vx_buffer_h vx_buffer`: Handle for memory management.
