# TensorRT-LLM engine backend vs vLLM at fp16 on an A10G

I wanted to know whether a compiled TensorRT-LLM engine was worth its build and dependency cost for a small model at fp16 on Ampere hardware. The answer, with default settings and the offline Python API on both sides, was no.

## Setup

- Model: `Qwen/Qwen2.5-1.5B-Instruct`, forced to float16 on both sides (the model's native dtype is bf16).
- GPU: one NVIDIA A10G, 23 GB, compute capability 8.6. No fp8 support on this part.
- vLLM 0.28.0 and TensorRT-LLM 1.1.0 using the compiled engine backend (`tensorrt_llm._tensorrt_engine.LLM`).
- Same prompts, same 8-request warmup, fixed 128 output tokens with `ignore_eos`, greedy decoding.
- Single stream: 64 requests, one at a time. Throughput: 256 requests submitted as one batch.

## Results

| metric | vLLM 0.28.0 | TensorRT-LLM 1.1.0 engine | ratio |
|---|---|---|---|
| single-stream latency p50 | 1009.9 ms | 2648.2 ms | vLLM 2.6x faster |
| single-stream latency p99 | 1010.5 ms | 2650.7 ms | |
| ms per output token (single stream) | 7.89 | 20.69 | vLLM 2.6x |
| decode tok/s (single stream) | 126.7 | 48.3 | |
| batch throughput, 256 requests | 12,366 tok/s | 7,982 tok/s | vLLM 1.55x |
| batch request rate | 96.6 req/s | 62.4 req/s | |
| GPU memory (nvidia-smi) | 19,783 MiB | 21,580 MiB | both are reserved pools |
| startup | 162 s load including CUDA graph capture | 41.6 s engine build, 58.5 s with executor | |

The memory row is not model footprint. vLLM reserves a KV pool at `gpu_memory_utilization=0.85` and the TensorRT engine reserves its own block. The fp16 weights of a 1.5B model are about 3 GB.

## How to read the gap

The single-stream gap is inflated by call overhead, not just kernels. At 1.5B parameters and fp16, memory-bandwidth-bound decode on this card should land around 5 to 8 ms per token. vLLM's 7.89 ms is close to that, helped by CUDA graph capture. The TensorRT-LLM backend's 20.69 ms is mostly per-call overhead in the high-level Python API driven one request at a time. The batch comparison amortizes that overhead and vLLM still wins by 1.55x.

What was not tested, and would change the picture:

- fp8 or int4. This is where TensorRT-LLM's kernel fusion and halved weight bytes pay off, and it needs Hopper or Ada hardware. The A10G cannot run it.
- A tuned TensorRT-LLM deployment: C++ executor, in-flight batching, CUDA graphs, tuned `max_batch_size`. Those would narrow or flip the result, and they are also the cost you are deciding whether to pay.
- Larger models, long context, sustained concurrent load.

## What it cost to get TensorRT-LLM running

These are the real obstacles, in the order I hit them:

1. `import tensorrt_llm` fails without OpenMPI. `mpi4py` needs `libmpi`, so `libopenmpi-dev` and `openmpi-bin` have to be in the image.
2. `pip install tensorrt_llm==1.1.0` is broken out of the box with torch 2.9. The bindings are built against CUDA 13 (`libcudart.so.13`, `libcublasLt.so.13`), but dependency resolution pulls the cu12 libraries. The fix is to install the CUDA 13 libraries explicitly. On the NVIDIA index the cu13 packages have no suffix (`nvidia-cublas`, not `nvidia-cublas-cu13`; the suffixed name on PyPI is a 1 KB placeholder) and `site-packages/nvidia/cu13/lib` plus `tensorrt_cu13_libs` have to be on `LD_LIBRARY_PATH`.
3. The install is large. `tensorrt_cu13_libs` alone is a 2.7 GB wheel; the full environment is several GB.
4. In 1.x, `from tensorrt_llm import LLM` is the PyTorch backend by default. To benchmark a compiled engine you must import from `tensorrt_llm._tensorrt_engine`. A plan built around static compiled engines is building on a path upstream is de-emphasizing.
5. Every model and most configuration changes require a new engine build (41.6 s here for 1.5B; longer for larger models and longer contexts), and the engine is tied to GPU architecture and TensorRT version.

## Conclusion

For fp16 on Ampere with small models, vLLM was faster in both single-stream latency and batch throughput, needed no per-model compilation, and installed cleanly. TensorRT-LLM may still win on Hopper with fp8, but that is a different experiment on different hardware. Reproduction commands and raw JSON for both runs are kept alongside this note.
