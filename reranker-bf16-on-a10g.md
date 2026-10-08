# fp32 to bf16 on a cross-encoder reranker: where the 2x shows up and where it does not

Two measurements of the same model on the same GPU. The first runs the model the way a reranking service does, 20 documents per query in one forward pass. The second runs a single (query, document) pair and captures the kernel timeline. Together they explain why a 2x kernel speedup turns into a 2x service speedup in one setting and into nothing in the other.

## Setup

- Model: `jinaai/jina-reranker-v2-base-multilingual`, `max_length=512`.
- GPU: one NVIDIA A10G (22 GiB). torch 2.7.0 with CUDA 12.6, transformers 4.49, native attention (flash-attn not installed, identical for both precisions).
- Single process, closed loop, CUDA-synchronized timing, warmup excluded. Peak memory from `torch.cuda.max_memory_*`, GPU utilization sampled at 5 Hz.
- Workload for the service-shaped run: 4 queries, 20 documents each, 80 pairs per round. Document lengths range from one line to multi-page text truncated at 512 tokens.
- Workload for the kernel run: one pair, 60 tokens, the same pair for both precisions. Each iteration is `forward -> sigmoid -> .float().cpu()`, so it includes the device-to-host sync a service would do.

## Service-shaped run: 20 documents per request

| metric | fp32 | bf16 | change |
|---|---|---|---|
| throughput | 340.0 pairs/s (17.0 req/s) | 663.6 pairs/s (33.2 req/s) | 1.95x |
| latency p50 | 25.5 ms | 15.7 ms | 1.6x lower |
| latency p95 | 171.0 ms | 75.9 ms | 2.25x lower |
| latency p99 | 172.1 ms | 77.7 ms | 2.22x lower |
| peak activation memory | 1760 MiB | 894 MiB | 1.97x less |
| GPU memory in use (nvidia-smi) | 2487 MiB | 1373 MiB | 1.81x less |
| model weights | 1062 MiB | 546 MiB | 2.0x less |

Ranking did not change. Over the 80 pairs, top-1 match 100%, top-3 and top-5 overlap 100%, Spearman 0.9997, maximum score drift 0.0056, mean drift 0.0009.

### Cross-request batching made it worse

I also packed several requests' pairs into one forward, the way a `@serve.batch` decorator would.

| group size | throughput | p99 | GPU memory in use |
|---|---|---|---|
| bf16, per request only | 663.6 pairs/s | 77.7 ms | 1373 MiB |
| 4 requests | 300.1 pairs/s | 271.5 ms | 2993 MiB |
| 8 requests | 302.9 pairs/s | 534.1 ms | 4915 MiB |
| 16 requests | 304.1 pairs/s | 1062.9 ms | 11637 MiB |
| 32 requests | 304.9 pairs/s | 2118.5 ms | 19317 MiB |

Throughput fell to about 300 pairs/s regardless of group size, p99 grew linearly with the group, and at 32 requests the batch nearly filled the card. Two reasons, both visible in the data:

1. One 20-document request already drives the A10G to about 90% utilization at fp32. There is no idle time for a second request to fill.
2. Padding. A cross-request batch pads every pair to the longest sequence in the group, so short documents pay for 512 tokens of compute. Cost per pair went from about 1.5 ms to about 3.4 ms.

These closed-loop numbers leave out the time a real batcher spends waiting to collect a batch, so a deployed version would have a worse tail than the table shows.

## Kernel run: one pair, batch size 1

| per iteration | fp32 | bf16 | change |
|---|---|---|---|
| GPU kernel time | 2.244 ms | 1.141 ms | 1.97x less |
| of which GEMM | 1.871 ms | 0.777 ms | 2.41x less |
| tensor-core GEMM share of kernel time | 0.0% | 64.6% | |
| kernel launches | 241 | 240 | unchanged |
| device syncs | 2.03 | 2.03 | unchanged |
| GPU active window | 7.46 ms | 7.45 ms | unchanged |
| end-to-end latency p50 | 7.49 ms | 7.46 ms | unchanged |
| kernel time / GPU window | 29.7% | 15.3% | bf16 lower |

The top fp32 kernels are SIMT FP32 GEMMs: `ampere_sgemm_128x32_sliced1x4_tn`, `ampere_sgemm_32x32_sliced1x4_tn`, `ampere_sgemm_128x128_tn`. The top bf16 kernels are tensor-core GEMMs: `ampere_bf16_s16816gemm_bf16_64x64`, `cutlass_80_tensorop_bf16_s16816gemm`, `cutlass_80_wmma_tensorop_bf16_s161616gemm`. That is the whole mechanism behind the 2x: bf16 moves the matrix multiplies onto tensor cores, and GEMM time drops 2.4x.

At batch size 1 none of that reaches the wall clock. A forward issues about 240 kernel launches but only 1 to 2 ms of GPU work inside a 7.5 ms window. The GPU is idle 70% of the time at fp32 and 85% at bf16, waiting for the CPU to launch the next kernel. Shrinking the kernels while keeping the launch count makes the GPU idle more, which is why occupancy goes down and latency stays put.

The two runs agree. One pair occupies the GPU 15 to 30% of the time; a 20-pair request fills it, and only then does faster arithmetic turn into throughput.

### How the kernel numbers were taken

`nsys` was not available inside the container (non-root, not pip-installable), so I used `torch.profiler` with the CUPTI backend, which produces a full kernel timeline as a Chrome trace. The profiler adds large per-op CPU overhead, so occupancy comes from a separate unprofiled loop of 300 iterations timed with CUDA events, and the kernel breakdown comes from a 40-iteration profiled trace parsed directly. `key_averages()` was not used for totals because it double-counts an op and its kernel.

## What I would do with this

- Switch the service to bf16. The win is kernel-proven, memory halves, ranking is unchanged on this sample.
- Do not add naive cross-request batching to a reranker whose requests are already 20 documents wide. Revisit only if real traffic is dominated by one-document requests, and then only with length-bucketed padding.
- The memory halving matters more than the throughput when the GPU is lightly loaded: 1.4 GiB in use on a 22 GiB card is an argument for packing more than one model per GPU, not for a faster card.

## Caveats

- One A10G, one process, one 60-token pair for the kernel run, one 80-pair sample for the service run.
- bf16 correctness was checked on 4 queries and 80 pairs. A wider evaluation set is needed before claiming it for every input.
- No flash-attn in either arm. Absolute throughput would rise with it, and the fp32 to bf16 ratio would likely change.
- The 8.3x figure you get by comparing bf16 batched against fp32 at batch size 1 is not a fair "before and after", because the service already batched per request. 1.95x is the honest number.
