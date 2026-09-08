# LLM Complexity & T-Shirt Sizing

LLM complexity is a multidimensional constraint defined by memory, compute, bandwidth, and context. From first principles, infrastructure "size" (T-shirt sizing) is determined by the **largest constraint** among these four physical limits.

## Defining Factors

### 1. Total vs. Active Parameters
- **Total Parameters**: Dictate memory capacity (VRAM). You must load every weight to run inference.
- **Active Parameters**: Dictate arithmetic operations (FLOPs) per token. This determines generation speed.
- **Crucial distinction**: In **Dense** models, Total = Active. In **MoE** (Mixture of Experts) models, Total >> Active (e.g., 671B total, 37B active). MoE reduces compute but **does not** reduce memory.

### 2. Model Architecture
- **Dense Transformers** (LLaMA, Gemma): Full activation. Predictable scaling. Primary bottleneck is memory bandwidth.
- **Mixture of Experts (MoE)** (DeepSeek, Mixtral): Sparse activation via routing. Low FLOPs but high VRAM. Adds routing/load-balancing overhead.

### 3. Memory Bandwidth (HBM)
- Autoregressive generation is **bandwidth-bound** at low batch sizes. Latency is determined by how fast weights move from HBM to compute cores (`Active Params × Precision`).
- At high batch sizes, the model becomes **compute-bound** (FLOPs), where Active Params dominate.

### 4. Context Window & KV Cache (The Hidden Memory Tax)
During autoregressive generation, transformers cache the Key (K) and Value (V) tensors for every previous token to avoid recomputation. This is the **KV cache**. Its size is the hidden cost of long conversations and document processing.

> **KV Cache Size = 2 × L × d × Seq × Bytes × Compression**

Here is what each variable means:
- **`L` (Number of Layers)**: Deeper models have more layers, multiplying the cache. (e.g., LLaMA 3.1 70B has ~80 layers).
- **`d` (Hidden Dimension)**: The width of each layer. Larger models have wider hidden states (e.g., 8192 for 70B models).
- **`Seq` (Sequence Length)**: Total tokens in the prompt + generated output. This is the **linear multiplier**. Doubling the context length doubles the cache.
- **`Bytes` (Precision)**: The bit-width of the cache values (2 for BF16, 1 for FP8/INT8).
- **`Compression` (Attention Optimisation)**: Modern models use **GQA** (Grouped-Query Attention) or **MLA** (Multi-head Latent Attention) to reduce the KV heads.
  - `1.0` = Standard Multi-Head Attention (MHA).
  - `~0.125` (1/8) = Typical GQA compression (e.g., LLaMA 3).
  - `~0.0625` (1/16) = Typical MLA compression (e.g., DeepSeek V3).

**Why does this matter?** For short contexts (4K–128K), the cache is negligible (usually < 20 GB). However, for frontier models with **1M–10M token windows** (e.g., LLaMA 4 Maverick, Gemini), the cache *explodes* and **often exceeds the model weights themselves**, shifting the primary bottleneck from "Memory Capacity" to "KV Cache".

### 5. Numerical Precision
- Acts as a scalar multiplier on memory and bandwidth.
- BF16 = 2 bytes, FP8/INT8 = 1 byte, INT4 = 0.5 bytes.
- Lower precision reduces memory footprint but does not change FLOP counts.

## T-Shirt Sizing Framework

To assign a size (XS to XL), calculate the **peak requirement** across **three physical constraints**. The highest category dictates the final size.

### The Three Categories

| Category | Formula | What It Measures |
| :--- | :--- | :--- |
| **1. Memory Capacity (VRAM)** | `(Total Params × Bytes) + KV Cache` | How many GPUs are required to host the model. |
| **2. Compute (FLOPs)** | `≈ 2 × Active Params` | Arithmetic operations per token. Dictates throughput at high batch sizes. |
| **3. Context Overhead (KV Cache)** | `2 × L × d × Seq × Bytes × Compression` | Extra memory for long sequences. Dominates at 1M+ tokens. |


### Sizing Matrix

| Size | Memory Capacity (VRAM) | Active Compute Equivalent | Typical Hardware |
| :--- | :--- | :--- | :--- |
| **XS** | < 20 GB | < 10B | 1x Consumer GPU (24GB) |
| **S** | 20 – 80 GB | 10B – 30B | 1x A100/H100 80GB |
| **M** | 80 – 300 GB | 30B – 70B | 2–4x A100/H100 |
| **L** | 300 – 600 GB | 70B – 150B | 4–8x A100/H100 (NVLink) |
| **XL** | > 600 GB | > 150B | 8+ H100s (Multi-Node) |

## Practical Examples

| Model | Architecture | Total / Active | Precision | Context Window | VRAM (Calculation) | Dominant Bottleneck | Final Size |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **[Gemma 3 12B](https://huggingface.co/yrrhall/gemma-3-12b-it-qat-q4_0-unquantized)** | Dense | 12B / 12B | BF16 | 128K | 12B × 2B = **24 GB** | Bandwidth | **S** |
| **[LLaMA 3.1 70B](https://featherless.ai/models/nvidia/Llama-3.1-70B-Instruct-FP8)** | Dense | 70B / 70B | BF16 / FP8 | 128K | 70B × 2B = **140 GB** (BF16) <br> 70B × 1B = **70 GB** (FP8) | Bandwidth | **M** (S with FP8) |
| **[Mixtral 8×22B](https://docs.mistral.ai/models/model-cards/mixtral-8x22b-0-1-0-3)** | MoE | 141B / 39B | BF16 | 64K | 141B × 2B = **282 GB** | Memory Capacity | **M** |
| **[Qwen3 235B-A22B](https://qwenlm.github.io/blog/qwen3/)** | MoE | 235B / 22B | BF16 / FP8 | 128K (32K native)[reference:0] | 235B × 2B = **470 GB** (BF16) <br> 235B × 1B = **235 GB** (FP8) | Memory Capacity | **L** (M with FP8) |
| **[DeepSeek V3](https://arxiv.org/abs/2412.19437)** | MoE | 671B / 37B | FP8 | 128K | 671B × 1B = **671 GB** | Memory Capacity | **XL** |
| **[DeepSeek V3.2](https://docs.vultr.com/inference-cookbook/cuda/model-guides/deepseek-v32)** | MoE (MLA) | 685B / ~37B | FP8 | 128K | 685B × 1B = **685 GB**[reference:1] | Memory Capacity | **XL** |
| **[DeepSeek V4 Flash](https://deepinfra.com/blog/deepseek-v4-flash-vs-qwen3-6-vs-glm-4-6)** | MoE | 284B / 13B | FP4 | 1M | 284B × 0.5B = **142 GB** | Context / Bandwidth | **M** |
| **[Qwen3.6 35B-A3B](https://deepinfra.com/blog/deepseek-v4-flash-vs-qwen3-6-vs-glm-4-6)** | MoE (256 experts) | 35B / 3B | Unknown | 262K (1M with YaRN)[reference:2] | 35B × 2B = **70 GB** (est.) | Bandwidth | **S** |
| **[GLM-4.6](https://deepinfra.com/blog/deepseek-v4-flash-vs-qwen3-6-vs-glm-4-6)** | MoE | 357B / 32B | Unknown | 200K[reference:3] | 357B × 2B = **714 GB** (est.) | Memory Capacity | **XL** |
| **[GLM-5](https://fireworks.ai/blog/best-open-source-llms)** | MoE | 744B / 40B | Unknown | 200K[reference:4] | 744B × 2B = **1.49 TB** (est.) | Memory Capacity | **XL** |
| **[LLaMA 4 Maverick](https://developer.nvidia.com/blog/nvidia-accelerates-inference-on-meta-llama-4-scout-and-maverick/)** | MoE (128 experts) | ~400B / 17B | FP8 | 10M (1M effective)[reference:5] | 400B × 1B + 300GB Cache = **~700 GB** | KV Cache | **XL** |
| **[LLaMA 4 Scout](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct)** | MoE (16 experts) | ~109B / 17B | FP8 | 10M | 109B × 1B + 300GB Cache = **~409 GB** | KV Cache | **L** |
| **[Kimi K2.5](https://fireworks.ai/blog/best-open-source-llms)** | MoE | ~1T / 32B | Unknown | 256K[reference:6] | ~1T × 2B = **2 TB** (est.) | Memory Capacity | **XL** |

> **Why is 300GB added for Maverick/Scout?**
> The 10M context window drives the KV cache calculation:
> `KV Cache ≈ 2 × 80 (L) × 5120 (d) × 10,000,000 (Seq) × 1 (Bytes) × 0.125 (Compression) ≈ 1,024 GB`.
> With sparse attention and sliding window optimisations, this is practically reduced to ~300 GB in production. At standard 128K contexts, the cache is < 20 GB and does not affect the final size, which is why it is excluded from other models in this table.

## Conclusion

From a first-principles perspective, T-shirt sizing is **memory-first**, architecture-aware, and context-dependent.

- **Dense models**: Size scales with total params and bandwidth.
- **MoE models**: Size scales with total params (memory), while active params determine operational speed.
- **Long-context models**: Must account for KV cache, which can dominate VRAM.

Use the **largest constraint** among Memory, Compute (FLOPs), and Context to determine your final hardware requirement.

## References

1. Liew, S.P., et al. "Towards Principled Design of Mixture-of-Experts." *arXiv:2601.08215*.
2. Zhou, Z., et al. "A Survey on Efficient Inference for LLMs." *arXiv:2404.14294*.
3. "Mixture of Experts (MoEs) in Transformers." *Hugging Face Blog*, 2026.
4. DeepSeek V3 Technical Report. *arXiv:2412.19437*.
5. DeepSeek V3.2 Documentation. *Vultr Docs*, 2026. [reference:7]
6. DeepSeek V4 Flash vs Qwen3.6 vs GLM-4.6. *DeepInfra Blog*, 2026. [reference:8]
7. Best Open Source LLMs in 2026. *Fireworks AI Blog*, 2026. [reference:9]
8. Qwen3 Technical Blog. *QwenLM*, 2025. [reference:10]
9. LLaMA 4 Model Card. *Meta / Hugging Face*, 2025. [reference:11]