# DeepSeek Model Usage and Optimisation

SGLang provides several optimizations specifically designed for the DeepSeek model to boost its inference speed. This document outlines current optimizations for DeepSeek. Additionally, the SGLang team is actively developing enhancements for [DeepSeek-V3](https://github.com/sgl-project/sglang/issues/2591).

## Use DeepSeek_V3 on SGLang
SGLang is recognized as one of the top engines for [DeepSeek model inference](https://github.com/deepseek-ai/DeepSeek-V3/tree/main?tab=readme-ov-file#62-inference-with-sglang-recommended).

### Install and Launch
Please refer to [DeepSeek_V3 install](https://github.com/sgl-project/sglang/blob/7ab84948d87d2c264cccc4ae8c1db339b9efea6a/docs/backend/server_arguments.md) for installation.

### Launch with One node
```bash
# launching one node
python -m sglang.launch_server --model-path deepseek-ai/DeepSeek-V3 --tp 16 --dist-init-addr 10.0.0.1:5000
```

### Multiple nodes
```bash
# launching Multiple node, we use two nodes as an example here
# Node 0
python -m sglang.launch_server --model-path deepseek-ai/DeepSeek-V3 --tp 16 --dist-init-addr 10.0.0.1:5000 --nnodes 2 --node-rank 0

# Node 1
python -m sglang.launch_server --model-path deepseek-ai/DeepSeek-V3 --tp 16 --dist-init-addr 10.0.0.1:5000 --nnodes 2 --node-rank 1
```
For high QPS scenarios, add the `--enable-dp-attention` [argument](https://github.com/sgl-project/sglang/blob/7ab84948d87d2c264cccc4ae8c1db339b9efea6a/docs/backend/server_arguments.md#optimization) to boost throughput.
### Do not use quantization
Deepseek V3 is already in FP8. So we should not run it with  `--quantization fp8 --kv-cache-dtype fp8_e5m2`

## Optimisations
### Multi-head Latent Attention (MLA) Throughput Optimizations

**Description**: [MLA](https://arxiv.org/pdf/2405.04434) is an innovative attention mechanism introduced by the DeepSeek team, aimed at improving inference efficiency. SGLang has implemented specific optimizations for this, including:

- **Weight Absorption**: By applying the associative law of matrix multiplication to reorder computation steps, this method balances computation and memory access and improves efficiency in the decoding phase.
- **Triton Decoding Kernel Optimization**: In the MLA decoding kernel, there is only one KV head. This optimization reduces memory access to the KV cache by processing multiple query heads within one block, accelerating the decoding process.

- **FP8 Quantization**: W8A8 FP8 and KV Cache FP8 quantization enables efficient FP8 inference. Additionally, we have implemented Batched Matrix Multiplication (BMM) operator to facilitate FP8 inference in MLA with weight absorption.

- **CUDA Graph & Torch.compile**: Both MLA and Mixture of Experts (MoE) are compatible with CUDA Graph and Torch.compile, which reduces latency and accelerates decoding speed for small batch sizes.

Overall, with these optimizations, we have achieved up to a 7x acceleration in output throughput compared to the previous version.

<p align="center">
  <img src="https://lmsys.org/images/blog/sglang_v0_3/deepseek_mla.svg" alt="Multi-head Latent Attention for DeepSeek Series Models">
</p>

**Usage**: MLA optimization is enabled by defalut, to disable, use `--disable-mla`.

**Reference**: Check [Blog](https://lmsys.org/blog/2024-09-04-sglang-v0-3/#deepseek-multi-head-latent-attention-mla-throughput-optimizations) and [Slides](https://github.com/sgl-project/sgl-learning-materials/blob/main/slides/lmsys_1st_meetup_deepseek_mla.pdf) for more details.

### Data Parallelism Attention

**Description**: This optimization involves data parallelism (DP) for the MLA attention mechanism of DeepSeek Series Models, which allows for a significant reduction in the KV cache size, enabling larger batch sizes. Each DP worker independently handles different types of batches (prefill, decode, idle), which are then synchronized before and after processing through the Mixture-of-Experts (MoE) layer.

<p align="center">
  <img src="https://lmsys.org/images/blog/sglang_v0_4/dp_attention.svg" alt="Data Parallelism Attention for DeepSeek Series Models">
</p>

**Usage**: This optimization is aimed at improving throughput and should be used for scenarios with high QPS (Queries Per Second). Data Parallelism Attention optimization can be enabeld by `--enable-dp-attention` for DeepSeek Series Models.

<p align="center">
  <img src="https://lmsys.org/images/blog/sglang_v0_4/deepseek_coder_v2.svg" alt="Data Parallelism Attention Performance Comparison">
</p>

**Reference**: Check [Blog](https://lmsys.org/blog/2024-12-04-sglang-v0-4/#data-parallelism-attention-for-deepseek-models).

### Multi Node Tensor Parallelism

**Description**: For users with limited memory on a single node, SGLang supports serving DeepSeek Series Models, including DeepSeek V3, across multiple nodes using tensor parallelism. This approach partitions the model parameters across multiple GPUs or nodes to handle models that are too large for one node's memory.

**Usage**: Check [here](https://github.com/sgl-project/sglang/tree/main/benchmark/deepseek_v3#example-serving-with-2-h208) for usage examples.

### Block-wise FP8

**Description**: SGLang implements block-wise FP8 quantization with two key optimizations:

- **Activation**: E4M3 format using per-token-per-128-channel sub-vector scales with online casting.
- **Weight**: Per-128x128-block quantization for better numerical stability.

**Usage**: turn on by default for DeepSeek V3 models.
