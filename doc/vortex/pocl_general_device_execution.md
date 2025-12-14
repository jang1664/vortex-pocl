# PoCL Device Entry Points: General vs. Vortex

이 문서는 PoCL(Portable Computing Language)에서 일반적인 디바이스들이 커널 실행(Entry Point)을 어떻게 처리하는지 분석하고, Vortex 백엔드의 방식과 비교합니다.

## 1. 일반적인 PoCL 디바이스의 실행 모델

PoCL은 다양한 디바이스를 지원하기 위해 `pocl_device_ops` 구조체를 통해 추상화된 인터페이스를 제공합니다. 커널 실행과 관련된 주요 콜백은 `run` 또는 `submit`입니다.

### 1.1. Software Devices (예: `basic`, `cpu`)

`basic` 디바이스는 가장 단순한 형태의 레퍼런스 구현체로, 호스트 CPU에서 싱글 스레드로 커널을 실행합니다.

*   **실행 흐름**:
    1.  `pocl_basic_run` 함수가 호출됩니다.
    2.  **인자 준비**: 커널 인자들을 `void*` 배열(`arguments`)로 준비합니다. 각 요소는 실제 데이터나 버퍼를 가리키는 포인터입니다.
    3.  **Workgroup 반복**: 호스트 코드에서 3중 루프(x, y, z)를 돌며 각 워크그룹을 순차적으로 실행합니다.
    4.  **함수 호출**: 컴파일된 커널 함수(`pocl_workgroup_func`)를 직접 호출합니다. 이때 인자 배열(`arguments`)과 현재 워크그룹 ID 등을 넘깁니다.

    ```c
    // lib/CL/devices/basic/basic.c
    ((pocl_workgroup_func) cmd->command.run.wg)
      ((uint8_t *)arguments, (uint8_t *)pc, x, y, z);
    ```

*   **특징**:
    *   호스트가 직접 워크그룹 스케줄링을 담당합니다.
    *   인자는 포인터 배열 형태로 전달됩니다.
    *   별도의 "커널 바이너리 업로드" 과정 없이, 호스트 메모리에 로드된 함수 포인터를 실행합니다.

### 1.2. Hardware Accelerators (예: `cuda`)

`cuda` 디바이스는 별도의 하드웨어(GPU)에서 커널을 실행하므로, 드라이버 API를 통해 명령을 전달합니다.

*   **실행 흐름**:
    1.  `pocl_cuda_submit` -> `pocl_cuda_submit_kernel` 순으로 호출됩니다.
    2.  **인자 준비**: 커널 인자들을 `void*` 배열(`params`)로 준비합니다. 이때 디바이스 메모리 포인터(CUdeviceptr)나 값들이 담깁니다.
    3.  **커널 런칭**: CUDA Driver API인 `cuLaunchKernel`을 호출하여 GPU에 실행 명령을 내립니다.

    ```c
    // lib/CL/devices/cuda/pocl-cuda.c
    result = cuLaunchKernel (function, ..., stream, params, NULL);
    ```

*   **특징**:
    *   하드웨어 드라이버(CUDA Driver)가 인자 패킹 및 전달 방식을 추상화하여 처리합니다.
    *   PoCL 레벨에서는 여전히 "인자들의 포인터 배열"을 API에 넘겨주는 방식입니다.
    *   워크그룹 스케줄링은 하드웨어/드라이버가 담당합니다.

## 2. Vortex 디바이스의 실행 모델 (Specialized)

Vortex는 RISC-V 기반의 GPGPU로, PoCL 드라이버가 하드웨어에 더 밀접하게 관여하며, 인자 전달 방식이 독특합니다.

*   **실행 흐름**:
    1.  `pocl_vortex_run` 함수가 호출됩니다.
    2.  **인자 패킹 (Argument Packing)**:
        *   일반적인 "포인터 배열" 전달 방식이 아닙니다.
        *   모든 커널 인자(스칼라 값, 버퍼 주소, 로컬 메모리 크기 등)를 **하나의 연속된 메모리 버퍼(Argument Buffer)**에 직렬화(Packing)합니다.
        *   이 구조는 `kernel_args_t` 구조체(device-side)와 일치해야 합니다.
    3.  **버퍼 업로드**: 패킹된 Argument Buffer를 디바이스 메모리에 할당하고 복사합니다.
    4.  **커널 실행**: Vortex 런타임(`vx_start`)을 통해 실행을 요청합니다. 이때 커널에게는 **Argument Buffer의 주소 하나만** 전달됩니다.

    ```c
    // lib/CL/devices/vortex/pocl-vortex.c
    // 인자들을 하나의 버퍼에 패킹
    vx_mem_alloc(..., &vx_kargs_buffer);
    vx_copy_to_dev(..., host_kargs_base_ptr, ...);
    // 커널 실행 시 인자 버퍼 주소 전달
    vx_start(..., dev_kargs_base_addr);
    ```

*   **Device-side 처리**:
    *   Vortex 디바이스에서 실행되는 `kernel_main.c`는 전달받은 하나의 포인터(Argument Buffer)를 통해 인자들을 언패킹(Unpacking)하여 실제 커널 함수를 호출합니다.

## 3. 요약 및 비교

| 특징 | Basic (Software) | CUDA (Accelerator) | Vortex (Specialized) |
| :--- | :--- | :--- | :--- |
| **실행 주체** | 호스트 CPU (Loop) | GPU (Driver API) | Vortex GPU (Runtime) |
| **인자 전달** | `void*` 배열 (Function Call) | `void*` 배열 (Driver API) | **Packed Buffer (Single Pointer)** |
| **스케줄링** | 호스트가 직접 수행 | 하드웨어/드라이버 위임 | 하드웨어 위임 |
| **Entry Point** | 함수 포인터 직접 호출 | `cuLaunchKernel` | `vx_start` (Arg Buffer 주소 전달) |

**결론**:
Vortex가 "Specialized"되었다고 느끼는 이유는, 일반적인 가속기(CUDA 등)가 드라이버 API 레벨에서 인자 전달을 추상화해주는 반면, Vortex 백엔드는 **드라이버가 직접 인자 패킹(Serialization)을 수행하고, 디바이스 런타임(kernel_main.c)이 이를 언패킹하는 구조**를 명시적으로 구현하고 있기 때문입니다. 이는 하드웨어 인터페이스가 매우 Low-level(메모리 주소 전달 방식)이기 때문에 발생하는 차이점입니다.
