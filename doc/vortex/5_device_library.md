# 5. Device Library Implementation

The `lib/kernel/vortex` directory contains the implementation of OpenCL built-in functions optimized for the Vortex architecture. These files are compiled and linked into the final kernel binary.

## Work-Item Functions (`workitems.c`)

This file implements the functions that allow a kernel to query its position in the execution grid. It maps OpenCL terminology to Vortex (CUDA-like) hardware intrinsics.

### Mapping Table

| OpenCL Function | Vortex Intrinsic / Variable | Description |
| :--- | :--- | :--- |
| `get_work_dim()` | `g_work_dim` | Global variable set by `kernel_main.c`. |
| `get_global_offset(dim)` | `g_global_offset` | Global variable set by `kernel_main.c`. |
| `get_group_id(dim)` | `blockIdx.x/y/z` | Hardware register for Workgroup ID. |
| `get_local_id(dim)` | `threadIdx.x/y/z` | Hardware register for Local Thread ID. |
| `get_num_groups(dim)` | `gridDim.x/y/z` | Hardware register for Grid Size. |
| `get_local_size(dim)` | `blockDim.x/y/z` | Hardware register for Block Size. |

### Derived Functions

Some functions are calculated from the basic intrinsics:

*   **`get_global_size(dim)`**: `blockDim * gridDim`
*   **`get_global_id(dim)`**: `(blockIdx * blockDim) + threadIdx + global_offset`

## Synchronization (`barrier.c`)

Implements synchronization primitives.

### `barrier(flags)`

```c
void _Z7barrierj(int flags) {
  if (flags & CLK_GLOBAL_MEM_FENCE) {
    vx_fence();
  }
  vx_barrier(__local_group_id, __warps_per_group);
}
```

*   **`vx_fence()`**: Ensures all memory operations are visible to other threads (flush caches/buffers). Used when `CLK_GLOBAL_MEM_FENCE` is set.
*   **`vx_barrier(...)`**: A hardware barrier that stalls execution until all threads in the workgroup have reached this point.
    *   `__local_group_id`: The ID of the current workgroup (cluster/core context).
    *   `__warps_per_group`: The number of warps participating in the barrier.

## Printing (`printf.c`)

Implements device-side `printf`.

```c
int printf (const char *restrict fmt, ...) {
  // ... va_list handling ...
  ret = vx_vprintf(fmt, va);
  // ...
}
```

*   **`vx_vprintf`**: A Vortex runtime function that tunnels the formatted string and arguments back to the host (usually via a special IO device or memory buffer) to be printed on the host's console.
