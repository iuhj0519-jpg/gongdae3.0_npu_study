# Systolic Array NPU for MNIST FC-MLP

MNIST handwritten digit inference를 수행하는 오픈소스 FC-MLP RTL에 자체 설계한 **5×5 systolic array 기반 NPU**를 이식하고,
interface를 최적화해 기존 FC-MLP보다 성능을 끌어올리는 것을 목표로 하는 SystemVerilog 프로젝트이다.

- Reference model 출처: [yeshvanth-m/CNN-Handwritten-Digit-MNIST](https://github.com/yeshvanth-m/CNN-Handwritten-Digit-MNIST/tree/main/Neuron/Source/rtl)
- `zyNet`은 위 오픈소스 reference model에 포함된 top module이다.
- 개발 과정 정리: [블로그 글](https://blog.naver.com/iuhj0519/224387246213)

## 프로젝트 구성

```text
Open-source FC-MLP reference model (zyNet)
        +
5×5 systolic array NPU 이식
        ↓
systolic_project
        +
AXI latency hiding, Unified Buffer 통합
        ↓
dnn_optimization_project
        +
BRAM, DSP mapping 최적화
        ↓
rag_db_project
```

| 경로 | 역할 |
|---|---|
| `systolic_project/` | reference model에 5×5 systolic array NPU를 이식한 prototype |
| `dnn_optimization_project/` | `systolic_project`의 AXI latency hiding과 buffer 통합 최적화 결과 |

BRAM, DSP mapping 등의 추가 최적화는 [rag_db_project](https://github.com/iuhj0519-jpg/rag_db_project)로 이어진다.

## 기준 사양

`rtl/module/npu_parameter_pkg.sv` 기준이다.

| 항목 | 값 |
|---|---|
| Network | 784 → 30 → 20 → 10 FC-MLP |
| Systolic array | 5×5 (`ARRAY_SIZE = 5`) |
| Batch size | 5 (`BATCH_SIZE = 5`) |
| Data width | 8-bit (`DATA_W = 8`) |
| Output class | 10 |
| Testbench sample | 100 (`MaxTestSamples`) |

## `systolic_project`

Reference model의 neuron별 연산 구조를 5×5 systolic array 기반 연산 구조로 대체한 prototype이다.

| 경로 | 내용 |
|---|---|
| `rtl/module/NPU_Top.sv`, `zyNet.sv` | NPU top과 reference model top |
| `rtl/module/axi_bus/` | `axi_lite_wrapper` |
| `rtl/module/buffer/` | `Input_Buffer`, `Global_Buffer` |
| `rtl/module/sram/` | `Weight_SRAM`, `Bias_SRAM`, `Sig_ROM` |
| `rtl/module/systolic_array/` | `mac_pe`, `pe_systolic_cell`, `systolic_array_2d`, `systolic_controller` |
| `rtl/module/vpu/` | `Activation_Unit`, `maxFinder` |
| `rtl/tb/top_sim.sv` | 전체 inference testbench |
| `rtl/tb/tb_checker.sv` | AXI-Lite를 우회해 `NPU_Top` 내부 layer 결과를 확인하는 stand-alone checker |
| `memory/` | weight, bias, sigmoid content `.mif` |
| `project/` | ModelSim project 파일 |

## `dnn_optimization_project`

`systolic_project`를 기반으로 다음 두 가지를 목표로 최적화했다.

1. **AXI latency hiding**: AXI latency를 hiding해 simulation run time을 줄인다.
2. **Unified Buffer**: `Input_Buffer`와 `Global_Buffer`를 `Unified_Buffer` 하나로 통합해 buffer area를 줄인다.

### 결과

AXI 입력 로딩(3,920 cycle/batch)을 연산과 겹쳐 전체 simulation time을 약 41% 줄였고, 정확도는 그대로 유지했다.

| 항목 | `systolic_project` | `dnn_optimization_project` |
|---|---|---|
| Simulation time | 1,844,235 ns (184,424 cycle) | 1,087,155 ns (108,716 cycle) |
| Accuracy | 99.0% (FAIL 1) | 99.0% (FAIL 1) |
| AXI 입력 로딩 | 매 batch 연산 전에 직렬 수행 | 첫 batch 이후 연산과 중첩 |

- 감소한 75,708 cycle은 20개 batch 중 19번의 AXI 로딩을 숨긴 이론값 3,920 × 19 = 74,480 cycle과 거의 일치한다.
- Input Loader FSM을 DNN Scheduler에서 분리해, compute가 시작되면 Loader가 곧바로 반대쪽 영역으로 다음 batch를 받는다.
- `Unified_Buffer`는 A/B 두 영역을 번갈아 쓰며, 각 영역은 입력 784 + 중간 결과 30 = 814 depth다.
- `axi_lite_wrapper`의 `statReg[3:0]`로 Loader 상태를 testbench에 전달하고, `top_sim`은 결과를 기다리기 전에 다음 batch를 보낸다.

`systolic_project` 대비 주요 변경 파일은 다음과 같다.

| 구분 | 파일 |
|---|---|
| 추가 | `rtl/module/buffer/Unified_Buffer.sv`, `memory/test_data/` (test vector 100개) |
| 제거 | `rtl/module/buffer/Input_Buffer.sv`, `rtl/module/buffer/Global_Buffer.sv`, `rtl/tb/tb_checker.sv` |
| 수정 | `NPU_Top.sv`, `zyNet.sv`, `npu_parameter_pkg.sv`, `axi_lite_wrapper.sv`, `maxFinder.sv`, `rtl/tb/top_sim.sv` |

이 폴더의 파일은 `rag_db_project`의 `historical_baselines/optimized_accelerator/`와 동일하다.

## 개발 환경

- Simulation: ModelSim
- Synthesis: Vivado
- FPGA 보드에 직접 올리지 않고 logic 단계까지만 진행했다.
