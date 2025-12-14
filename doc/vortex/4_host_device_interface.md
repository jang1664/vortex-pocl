# 4. Host-Device Interface (`kernel_args.h`)

The `lib/CL/devices/vortex/kernel_args.h` file defines the contract between the host driver (`pocl-vortex.c`) and the device runtime (`kernel_main.c`). It ensures both sides agree on the layout of the metadata passed during kernel execution.

## Structure Definition

```c
#include <stdint.h>

typedef struct {
  uint32_t work_dim;        // Number of dimensions (1, 2, or 3)
  uint32_t num_groups[3];   // Number of workgroups in each dimension (Grid Size)
  uint32_t local_size[3];   // Number of work-items per workgroup (Block Size)
  uint32_t global_offset[3];// Global offset for work-item IDs
  uint32_t kernel_id;       // ID of the kernel to execute (for multi-kernel programs)
} kernel_args_t;

// Helper macro for alignment
#define ALIGN_OFFSET(offset, alignment) (((offset) + (alignment) - 1) & ~((alignment) - 1))
```

## Memory Layout

When `pocl_vortex_run` prepares the argument buffer, it follows this layout:

| Offset | Content | Description |
| :--- | :--- | :--- |
| `0` | `kernel_args_t` | The struct defined above. Contains execution configuration. |
| `Aligned Size` | **Arguments** | The packed arguments for the kernel. |

### Argument Packing Rules

The **Arguments** section is packed as follows:

1.  **Scalar Arguments**: Copied directly (e.g., `int`, `float`).
2.  **Global Pointers**: The 32-bit or 64-bit device address (`buf_address`) is written.
3.  **Local Memory**:
    *   First, a `uint32_t` **size** is written (bytes to allocate).
    *   Second, a `uint32_t` **offset** is written (offset within the allocated block).
    *   *Note*: The actual allocation happens on the device via `vx_local_alloc`.

## Usage

*   **Host (`pocl-vortex.c`)**: Writes to this structure to configure the execution grid and select the kernel.
*   **Device (`kernel_main.c`)**: Reads this structure to pass dimensions to `vx_spawn_threads` and to resolve the kernel function pointer.
