# PoCL Vortex Backend - Core Functions

## 1. Initialization

### pocl_vortex_init
**Description**: Initialize Vortex device and configure device capabilities
```c
cl_int pocl_vortex_init(unsigned j, cl_device_id dev, const char* parameters)
```

### pocl_vortex_init_context
**Description**: Initialize OpenCL context and increment reference count
```c
int pocl_vortex_init_context(cl_device_id dev, cl_context context)
```

---

## 2. Memory Management

### pocl_vortex_alloc_mem_obj
**Description**: Allocate device memory buffer and copy initial data
```c
cl_int pocl_vortex_alloc_mem_obj(cl_device_id dev, cl_mem mem_obj, void *host_ptr)
```

### pocl_vortex_write
**Description**: Transfer data from host memory to device memory
```c
void pocl_vortex_write(void *data, const void *__restrict__ host_ptr,
                       pocl_mem_identifier *dst_mem_id, cl_mem dst_buf,
                       size_t offset, size_t size)
```

### pocl_vortex_read
**Description**: Transfer data from device memory to host memory
```c
void pocl_vortex_read(void *data, void *__restrict__ host_ptr,
                      pocl_mem_identifier *src_mem_id, cl_mem src_buf,
                      size_t offset, size_t size)
```

---

## 3. Program Build

### pocl_vortex_post_build_program
**Description**: Compile LLVM IR to Vortex binary (.vxbin) and generate kernel list
```c
int pocl_vortex_post_build_program(cl_program program, cl_uint device_i)
```

---

## 4. Kernel Launch

### pocl_vortex_run
**Description**: Pack kernel arguments and execute kernel on Vortex device
```c
void pocl_vortex_run(void *data, _cl_command_node *cmd)
```

**Key Operations**:
- Allocate and pack kernel argument buffer
- Upload kernel binary
- Start kernel execution via vx_start()
- Wait for completion via vx_ready_wait()

---

## Vortex Runtime API Integration

| PoCL Function | Vortex Runtime API |
|---------------|-------------------|
| `pocl_vortex_init` | `vx_dev_open()`, `vx_dev_caps()` |
| `pocl_vortex_alloc_mem_obj` | `vx_mem_alloc()` |
| `pocl_vortex_write` | `vx_copy_to_dev()` |
| `pocl_vortex_read` | `vx_copy_from_dev()` |
| `pocl_vortex_run` | `vx_upload_kernel_file()`, `vx_start()`, `vx_ready_wait()` |
