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
