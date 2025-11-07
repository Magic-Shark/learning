| GPU 型号                       | 架构代号                    | 显存类型 / 容量    | 显存带宽（解释：显存与核心的数据通道宽度，越高越快） | Tensor Core / AI 单元（解释：执行矩阵乘法的AI专核）    | FP32 算力（解释：单精度浮点计算性能，越高越能支撑通用计算）      | NVLink / Interconnect（解释：多卡间高速通信带宽） | 功耗 (TDP) | 典型用途                   | 优势                     | 局限性                |
| ---------------------------- | ----------------------- | ------------ | -------------------------- | -------------------------------------- | ------------------------------------- | ----------------------------------- | -------- | ---------------------- | ---------------------- | ------------------ |
| **NVIDIA H100 (80GB HBM3)**  | **Hopper (2022)**       | 80 GB HBM3   | 🔹 3.35 TB/s（极高带宽，缓解内存瓶颈）  | 第4代 Tensor Core（支持 FP8、FP16、BF16、TF32） | 67 TFLOPS (FP32)，989 TFLOPS (FP16)    | NVLink 4 (900 GB/s)，NVSwitch        | 700 W    | 大模型训练、推理、集群部署（GPT/LLM） | 当前最强AI加速卡，支持FP8推理，能效比高 | 极高价格、仅限服务器部署       |
| **NVIDIA A100 (80GB HBM2e)** | **Ampere (2020)**       | 80 GB HBM2e  | 🔹 2.04 TB/s               | 第3代 Tensor Core（FP64/32/16/BF16 支持）    | 19.5 TFLOPS (FP32)，312 TFLOPS (FP16)  | NVLink 3 (600 GB/s)，NVSwitch        | 400 W    | 通用AI训练、HPC、大规模分布式      | 性价比高，广泛部署于企业集群         | 效率略低于 H100，功耗仍高    |
| **NVIDIA L40S (48GB GDDR6)** | **Ada Lovelace (2023)** | 48 GB GDDR6  | 🔹 864 GB/s（中高带宽）          | 第4代 Tensor Core（FP16、FP8 支持）           | 91 TFLOPS (FP32)                      | PCIe Gen4 (双卡互联有限)                  | 350 W    | 推理、视频生成、私有部署           | 高性价比、兼容性强、适合多租户部署      | 无NVLink，训练扩展性有限    |
| **NVIDIA A40 (48GB GDDR6)**  | **Ampere (2021)**       | 48 GB GDDR6  | 🔹 696 GB/s                | 第3代 Tensor Core                        | 37 TFLOPS (FP32)                      | NVLink (600 GB/s)                   | 300 W    | 可视化AI、推理、边缘计算          | 稳定可靠，支持虚拟化（vGPU）       | 性能低于 A100，不适合大模型训练 |
| **AMD Instinct MI300X**      | **CDNA3 (2023)**        | 192 GB HBM3  | 🔹 5.2 TB/s（全行业最高带宽）       | Matrix Core（AI矩阵加速单元）                  | 82 TFLOPS (FP32)，>1,000 TFLOPS (FP16) | Infinity Fabric (896 GB/s)          | 750 W    | 超大模型训练、推理、开源AI（ROCm）   | 显存超大，性能强劲，支持开源生态       | CUDA 生态兼容性差，软件仍在完善 |
| **AMD Instinct MI250X**      | **CDNA2 (2021)**        | 128 GB HBM2e | 🔹 3.2 TB/s                | Matrix Core                            | 47.9 TFLOPS (FP32)                    | Infinity Fabric (800 GB/s)          | 500 W    | HPC、高性能训练              | 成熟稳定，成本低于 MI300X       | 不支持 FP8 等新精度，效率略低  |
| **Intel Gaudi 2**            | **Habana (2022)**       | 96 GB HBM2e  | 🔹 2.45 TB/s               | Tensor Processor Core（专用AI阵列）          | ~45 TFLOPS (FP32)，200 TFLOPS (FP16)   | On-Chip Fabric (200 GB/s)           | 600 W    | AI 推理、分布式训练            | 开源生态（PyTorch 支持）、性能稳定  | 软件生态不如NVIDIA成熟     |
| **Intel Gaudi 3** *(新发布)*    | **Habana (2024)**       | 128 GB HBM3  | 🔹 >3 TB/s                 | Tensor Processor Core（改进版）             | ~183 TFLOPS (FP32)，900 TFLOPS (FP16)  | On-Chip Fabric (>300 GB/s)          | 600 W    | 云端推理、AI训练              | 性能追平A100，成本更低          | 生态仍在建设中            |


# 主要参数解释：
| 参数名称                          | 解释                                            |
| ----------------------------- |-----------------------------------------------|
| **显存容量**                      | 直接决定能放多大模型。例如 GPT-3 175B 参数需多张 80 GB 显存的 GPU。 |
| **显存带宽**                      | 决定数据在“显存 ↔ 计算核心”之间传输速度，是隐藏延迟的关键。              |
| **Tensor Core / Matrix Core** | 执行 AI 中最重要的矩阵乘加（GEMM）操作，是 GPU 相比 CPU 的性能来源。   |
| **FP32 / FP16 / FP8**         | 不同精度代表不同性能/能耗权衡。AI 通常使用 FP16/BF16/FP8 训练。     |
| **NVLink / Infinity Fabric**  | GPU 间直连带宽，用于多卡并行通信，比 PCIe 快数倍。                |
| **TDP（功耗）**                   | 电力与散热设计要求，数据中心通常需机架散热或液冷。                     |

# 适用场景
| 目标方向                           | 推荐 GPU          | 推荐理由                           |
| ------------------------------ | --------------- | ------------------------------ |
| **分布式训练架构学习（MIG/NVLink/NCCL）** | A100 / H100     | 数据中心主力卡，支持多实例 GPU (MIG) 与高带宽互连 |
| **AI 推理服务、边缘部署研究**             | L40S / A40      | 功耗较低，支持企业虚拟化，适合云推理             |
| **开源基础设施研究（ROCm、非NVIDIA生态）**   | MI300X / Gaudi3 | 非 CUDA 架构代表，利于学习多厂商 AI Infra   |
| **HPC 与混合任务编排**                | A100 / MI250X   | 兼顾 HPC + AI，加深对混合计算调度的理解       |
