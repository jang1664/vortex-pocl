# POCL-Vortex OpenCL Kernel Compilation 분석

## 개요

이 문서는 POCL에서 Vortex 디바이스를 사용할 때 OpenCL 커널이 어떻게 컴파일되는지 분석합니다.

---

## 1. 전체 컴파일 Flow

```
OpenCL Kernel Source (.cl)
        │
        ▼
┌─────────────────────────────────────┐
│ Clang/LLVM Frontend                 │
│ (OpenCL C → LLVM IR)                │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│ pocl_llvm_run_passes_on_program()   │
│ ├── Stage 1 POCL Passes (11개)      │
│ │   └── automatic-locals (핵심)     │
│ └── Stage 2 POCL Passes (3개, SPMD) │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│ pocl_vortex_post_build_program()    │
│ └── compile_vortex_program()        │
│     ├── processKernels()            │
│     │   └── createArgumentsBuffer() │
│     ├── addKernelSelect()           │
│     └── WriteBitcodeToFile()        │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│ Clang Compilation                   │
│ (LLVM IR + kernel_main.c → ELF)     │
└─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│ vxbintool                           │
│ (ELF → .vxbin)                      │
└─────────────────────────────────────┘
        │
        ▼
    Runtime Execution
```

---

## 2. 주요 함수 및 파일 위치

### 2.1 Device Driver (pocl-vortex.c)

| 함수 | 위치 | 설명 |
|------|------|------|
| `pocl_vortex_init_device_ops()` | [pocl-vortex.c:79-123](lib/CL/devices/vortex/pocl-vortex.c#L79-L123) | 디바이스 operation 등록 |
| `pocl_vortex_init()` | [pocl-vortex.c:142-265](lib/CL/devices/vortex/pocl-vortex.c#L142-L265) | 디바이스 초기화, 핵심 설정 적용 |
| `pocl_vortex_post_build_program()` | [pocl-vortex.c:306-343](lib/CL/devices/vortex/pocl-vortex.c#L306-L343) | 컴파일 파이프라인 진입점 |
| `pocl_vortex_run()` | [pocl-vortex.c:415-615](lib/CL/devices/vortex/pocl-vortex.c#L415-L615) | 커널 실행 |

### 2.2 Vortex Utils (vortex_utils.cc)

| 함수 | 위치 | 설명 |
|------|------|------|
| `createArgumentsBuffer()` | [vortex_utils.cc:99-181](lib/CL/devices/vortex/vortex_utils.cc#L99-L181) | 커널 argument 변환 (핵심!) |
| `processKernels()` | [vortex_utils.cc:183-194](lib/CL/devices/vortex/vortex_utils.cc#L183-L194) | 커널 처리 |
| `addKernelSelect()` | [vortex_utils.cc:196-239](lib/CL/devices/vortex/vortex_utils.cc#L196-L239) | 커널 선택 dispatcher 생성 |
| `compile_vortex_program()` | [vortex_utils.cc:241-350](lib/CL/devices/vortex/vortex_utils.cc#L241-L350) | 최종 바이너리 생성 |

### 2.3 POCL LLVM Passes

| 함수 | 위치 | 설명 |
|------|------|------|
| `pocl_llvm_run_passes_on_program()` | [pocl_llvm_wg.cc:1212-1228](lib/CL/pocl_llvm_wg.cc#L1212-L1228) | LLVM pass 실행 |
| `addStage1PassesToPipeline()` | [pocl_llvm_wg.cc:436-504](lib/CL/pocl_llvm_wg.cc#L436-L504) | Stage 1 passes |
| `addStage2PassesToPipeline()` | [pocl_llvm_wg.cc:508-585](lib/CL/pocl_llvm_wg.cc#L508-L585) | Stage 2 passes |

### 2.4 AutomaticLocals Pass

| 함수 | 위치 | 설명 |
|------|------|------|
| `processAutomaticLocals()` | [AutomaticLocals.cc:58-136](lib/llvmopencl/AutomaticLocals.cc#L58-L136) | __local 변수를 argument로 변환 |

---

## 3. __local 메모리 처리

### 3.1 디바이스 설정

```c
// pocl-vortex.c:175-176
dev->autolocals_to_args = POCL_AUTOLOCALS_TO_ARGS_ALWAYS;  // __local → argument 변환 강제
dev->device_alloca_locals = CL_FALSE;                       // 스택 할당 비활성화
```

이 설정으로 인해:
- 커널 내 automatic __local 변수들이 커널 argument로 변환됨
- 디바이스가 스택에 local 메모리를 할당하지 않음

### 3.2 처리 단계

#### Step 1: AutomaticLocals Pass (POCL)

[AutomaticLocals.cc:58-136](lib/llvmopencl/AutomaticLocals.cc#L58-L136)

```cpp
// __local address space (3)의 글로벌 변수를 찾아서
for (Module::global_iterator i = M->global_begin(), e = M->global_end(); i != e; ++i) {
    if (isAutomaticLocal(F, *i)) {
        Locals.push_back(&*i);
        // 함수 파라미터 끝에 추가
        Parameters.push_back(i->getType());
    }
}
// 새 커널 함수 생성하고 메타데이터 업데이트
```

**변환 전:**
```c
__kernel void foo(__global int* data) {
    __local int temp[256];  // automatic local
    // ...
}
```

**변환 후:**
```c
__kernel void foo(__global int* data, __local int* _local0) {
    // temp → _local0 대체
}
```

#### Step 2: Vortex-specific 변환 (createArgumentsBuffer)

[vortex_utils.cc:137-153](lib/CL/devices/vortex/vortex_utils.cc#L137-L153)

```cpp
if (pocl::isLocalMemFunctionArg(function, arg_idx)) {
    if (allocated_local_mem == nullptr) {
        // 1. __local_size 로드
        auto local_size = Builder.CreateLoad(I32Ty, local_size_ptr, "__local_size");

        // 2. vx_local_alloc() 호출하여 local 메모리 할당
        auto vx_local_alloc_func = module->getOrInsertFunction("vx_local_alloc", function_type);
        allocated_local_mem = Builder.CreateCall(vx_local_alloc_func, {local_size}, "__local_mem");
    }
    // 3. offset 적용하여 각 __local argument 포인터 계산
    auto offset = Builder.CreateLoad(I32Ty, offset_ptr, OldArg.getName() + "_offset");
    Arg = Builder.CreateGEP(I8PtrTy, allocated_local_mem, offset, OldArg.getName() + "_byte_ptr");
}
```

#### Step 3: 런타임 (kernel_main.c)

[kernel_main.c:8-10](lib/CL/devices/vortex/kernel_main.c#L8-L10)

```c
void* vx_local_alloc(uint32_t size) {
    return __local_mem(size);  // Vortex runtime 함수
}
```

### 3.3 Argument Buffer 구조

런타임에서 커널 arguments는 단일 버퍼로 패킹됨:

```
┌────────────────────────────────────────────┐
│ kernel_args_t (16 bytes aligned)           │
│ ├── work_dim                               │
│ ├── num_groups[3]                          │
│ ├── local_size[3]                          │
│ ├── global_offset[3]                       │
│ └── kernel_id                              │
├────────────────────────────────────────────┤
│ __local_size (4 bytes) - 첫 __local만     │
├────────────────────────────────────────────┤
│ arg0_offset (4 bytes) - __local arg        │
├────────────────────────────────────────────┤
│ arg1 (pointer-sized) - regular pointer     │
├────────────────────────────────────────────┤
│ arg2 (N bytes) - scalar value              │
├────────────────────────────────────────────┤
│ ...                                        │
└────────────────────────────────────────────┘
```

---

## 4. Vortex-specific LLVM Passes

Vortex는 LLVM을 확장하여 SIMT(Single Instruction Multiple Threads) 실행을 위한 branch divergence를 처리합니다.

### 4.1 Pass 목록

| Pass | 파일 | 설명 |
|------|------|------|
| `VortexDivergenceAnalysis0` | [VortexBranchDivergence.cpp:345-643](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L345-L643) | Divergence 분석 전처리 |
| `VortexDivergenceAnalysis1` | [VortexBranchDivergence.cpp:647-666](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L647-L666) | Divergence 분석 후처리 |
| `VortexDivergenceArguments` | [VortexBranchDivergence.cpp:670-895](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L670-L895) | 함수 argument divergence 분석 |
| `VortexBranchDivergence0` | [VortexBranchDivergence.cpp:899-1112](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L899-L1112) | Branch divergence 전처리 |
| `VortexBranchDivergence1` | [VortexBranchDivergence.cpp:1116-1428](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L1116-L1428) | Branch divergence 변환 |
| `VortexBranchDivergence2` | [VortexBranchDivergence.cpp:1432-1556](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L1432-L1556) | Machine-level 후처리 |
| `VortexIntrinsicFuncLowering` | [VortexIntrinsicFunc.cpp:45-246](../vortex-llvm/llvm/lib/Target/RISCV/VortexIntrinsicFunc.cpp#L45-L246) | Vortex intrinsic 함수 lowering |
| `UniformAnnotationPass` | [VortexBranchDivergence.cpp:1567-1641](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L1567-L1641) | Uniform 변수 annotation 처리 |

### 4.2 Branch Divergence 처리

Vortex는 SIMT 실행 모델을 사용하므로, divergent branch를 특별히 처리해야 합니다:

```cpp
// VortexBranchDivergence1::processBranches() - VortexBranchDivergence.cpp:1377-1428
// Divergent branch 앞에 split 명령 삽입
auto stack_ptr = CallInst::Create(split_func_, cond, "", branch);

// IPDOM(immediate post-dominator) 전에 join 명령 삽입
CallInst::Create(join_func_, stack_ptr, "", stub_br);
```

**Vortex Intrinsics 사용:**

| Intrinsic | 용도 |
|-----------|------|
| `riscv_vx_tmask` | Thread mask 조회 |
| `riscv_vx_split` | Divergent branch split |
| `riscv_vx_join` | Branch reconvergence |
| `riscv_vx_pred` | Thread predication |
| `riscv_vx_tmc` | Thread mask control |

### 4.3 Intrinsic Function Lowering

[VortexIntrinsicFunc.cpp:81-210](../vortex-llvm/llvm/lib/Target/RISCV/VortexIntrinsicFunc.cpp#L81-L210)

Vortex API 함수들을 LLVM intrinsics로 변환:

| API 함수 | Intrinsic |
|----------|-----------|
| `vx_thread_id()` | `riscv_vx_tid` |
| `vx_warp_id()` | `riscv_vx_wid` |
| `vx_core_id()` | `riscv_vx_cid` |
| `vx_num_threads()` | `riscv_vx_nt` |
| `vx_num_warps()` | `riscv_vx_nw` |
| `vx_num_cores()` | `riscv_vx_nc` |
| `vx_barrier()` | `riscv_vx_bar` |

### 4.4 DivergenceTracker

[VortexBranchDivergence.cpp:1690-1823](../vortex-llvm/llvm/lib/Target/RISCV/VortexBranchDivergence.cpp#L1690-L1823)

LLVM의 UniformityAnalysis와 통합되어 값의 divergence를 추적:

```cpp
bool DivergenceTracker::isSourceOfDivergence(const Value *V) {
    // Atomics are divergent
    if (isa<AtomicRMWInst>(V) || isa<AtomicCmpXchgInst>(V))
        return true;

    // Function call results are assumed divergent by default
    if (isa<CallBase>(V))
        return true;

    return false;
}

bool DivergenceTracker::isAlwaysUniform(const Value *V) {
    // vortex.uniform annotation 체크
    if (II->getIntrinsicID() == Intrinsic::riscv_vx_uniform)
        return true;

    // Machine CSRs are uniform
    // warp_id, core_id 등 특수 CSR도 uniform
}
```

---

## 5. POCL LLVM Passes 상세

### 5.1 Stage 1 Passes

Vortex는 `dev->spmd = CL_TRUE` 설정으로 SPMD 모델을 사용합니다.

[pocl_llvm_wg.cc:436-504](lib/CL/pocl_llvm_wg.cc#L436-L504)

```cpp
// Stage 1 passes (11개)
1. fix-min-legal-vec-size    // 최소 벡터 크기 수정
2. inline-kernels            // 커널 인라인
3. optimize-wi-func-calls    // work-item 함수 최적화
4. handle-samplers           // 샘플러 처리
5. infer-address-spaces      // 주소 공간 추론
6. mem2reg                   // 메모리 → 레지스터 승격
7. domtree                   // Dominator tree 분석
8. workitem-handler-chooser  // work-item 핸들러 선택
9. flatten-inline-all        // SPMD용: 모든 함수 인라인
10. always-inline            // always_inline 속성 처리
11. automatic-locals         // __local 변수 처리 (핵심!)
12. optimize-wi-gvars        // work-item 전역변수 최적화
```

### 5.2 Stage 2 Passes (SPMD)

SPMD 디바이스(Vortex)는 barrier/loop 관련 pass를 건너뜁니다:

[pocl_llvm_wg.cc:508-585](lib/CL/pocl_llvm_wg.cc#L508-L585)

```cpp
// Stage 2 passes for SPMD (3개만)
1. simplifycfg     // CFG 단순화
2. loop-simplify   // 루프 단순화
3. allocastoentry  // alloca를 entry block으로 이동
```

**건너뛰는 passes (non-SPMD용):**
- implicit-loop-barriers
- workitemloops
- phi-node-elimination
- barrier-analysis

---

## 6. 디바이스 설정 요약

[pocl-vortex.c:167-188](lib/CL/devices/vortex/pocl-vortex.c#L167-L188)

| 설정 | 값 | 의미 |
|------|-----|------|
| `dev->type` | `CL_DEVICE_TYPE_GPU` | GPU 타입 디바이스 |
| `dev->spmd` | `CL_TRUE` | SPMD 실행 모델 |
| `dev->run_workgroup_pass` | `CL_FALSE` | workgroup pass 스킵 |
| `dev->autolocals_to_args` | `ALWAYS` | __local 항상 argument로 변환 |
| `dev->device_alloca_locals` | `CL_FALSE` | 스택 기반 local 비활성화 |
| `dev->llvm_target_triplet` | `riscv32/64-unknown-unknown-elf` | RISC-V 타겟 |
| `dev->llvm_cpu` | `generic-rv32/64` | Generic RISC-V CPU |
| `dev->llvm_abi` | `ilp32f` / `lp64d` | ABI |

---

## 7. 환경 변수

| 변수 | 용도 |
|------|------|
| `POCL_VORTEX_XLEN` | 비트 폭: "32" 또는 "64" (기본: "32") |
| `POCL_VORTEX_CFLAGS` | Clang 컴파일러 플래그 (필수) |
| `POCL_VORTEX_LDFLAGS` | 링커 플래그 (필수) |
| `POCL_VORTEX_BINTOOL` | vxbintool 경로 (필수) |
| `LLVM_PREFIX` | 커스텀 LLVM 설치 경로 |

---

## 8. 커널 실행 과정

[pocl-vortex.c:415-615](lib/CL/devices/vortex/pocl-vortex.c#L415-L615) 및 [kernel_main.c:14-25](lib/CL/devices/vortex/kernel_main.c#L14-L25)

```
1. Host: Argument buffer 준비
   ├── kernel_args_t 헤더 작성
   ├── __local 크기/offset 계산
   └── Arguments 마샬링

2. Host: Device 메모리 할당
   └── vx_mem_alloc()

3. Host: 데이터 전송
   └── vx_copy_to_dev()

4. Host: 커널 로드 및 실행
   ├── vx_upload_kernel()
   ├── vx_start()
   └── vx_ready_wait()

5. Device: kernel_main.c::main()
   ├── CSR에서 kernel_args 로드
   ├── __vx_get_kernel_callback()으로 커널 함수 조회
   └── vx_spawn_threads()로 스레드 스폰

6. Device: 커널 실행
   ├── vx_local_alloc()으로 local 메모리 할당
   └── 커널 코드 실행
```

---

## 9. 요약

### __local 처리 흐름

1. **AutomaticLocals Pass**: 커널 내 `__local` 변수를 커널 argument로 변환
2. **createArgumentsBuffer()**: 모든 `__local` arguments를 단일 `vx_local_alloc()` 호출로 통합
3. **Runtime**: `vx_local_alloc()` → `__local_mem()` Vortex runtime 함수 호출
4. **각 argument**: offset 계산으로 개별 `__local` 포인터 생성

### Vortex LLVM Pass 역할

- **Branch Divergence**: SIMT 실행에서 divergent branch를 split/join으로 처리
- **Intrinsic Lowering**: Vortex API를 LLVM intrinsics로 변환
- **Uniformity Analysis**: 값의 uniform/divergent 속성 추적

### POCL의 역할

- OpenCL C → LLVM IR 변환
- `__local` 변수 처리 (AutomaticLocals pass)
- SPMD 실행 모델에 맞는 pass 선택
