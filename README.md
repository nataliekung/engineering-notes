# Engineering notes

Measured results and upstream work on inference runtimes and Ray infrastructure. Every number in these notes comes from a run whose raw output I kept. Where something was not measured, the note says so instead of estimating.

| note | what it covers |
|---|---|
| [ray-gcs-restart-crash-loop.md](ray-gcs-restart-crash-loop.md) | A Ray GCS crash on restart under fault tolerance, traced to a write-ordering bug and fixed upstream (merged) |
| [reranker-bf16-on-a10g.md](reranker-bf16-on-a10g.md) | fp32 to bf16 on a cross-encoder reranker: 1.95x throughput at 20 documents per request, no change at batch size 1, and the kernel-level reason |
| [rollout-generation-vllm-vs-sglang.md](rollout-generation-vllm-vs-sglang.md) | vLLM 0.11 vs SGLang 0.4.9 on RL rollout-style generation: 2.38x on grouped sampling, equal on a long-tailed workload |
| [trt-llm-vs-vllm-fp16-a10g.md](trt-llm-vs-vllm-fp16-a10g.md) | TensorRT-LLM 1.1 compiled engine vs vLLM 0.28 at fp16 on an A10G, and what it took to install |
| [fractional-gpu-memory-isolation.md](fractional-gpu-memory-isolation.md) | What torch's per-process memory fraction protects when two processes share a GPU, where it stops, and MPS as the hard limit |
| [kuberay-worker-group-removal.md](kuberay-worker-group-removal.md) | Four KubeRay changes around removing a worker group: reject by default, surface orphans, validate names, drain before delete |
| [ray-jobs-logs-after-node-removal.md](ray-jobs-logs-after-node-removal.md) | The Ray Jobs log endpoint returned 500 after a worker node left; an issue and a PR for a structured 503 |
| [verl-fixes.md](verl-fixes.md) | Five verl fixes: nsys capture range, loop worker placement, a config constructor crash, weight-transfer receiver failures, unschedulable rollout servers |
| [verl-no-learning-signal-detector.md](verl-no-learning-signal-detector.md) | RFC: fail fast when a GRPO run has zero advantage variance for the whole run |
| [upstream-index.md](upstream-index.md) | Every upstream PR, issue and RFC with its current status |

Related: [inference-runtime-lab](https://github.com/nataliekung/inference-runtime-lab), a small engine with paged KV accounting and continuous batching, with its own benchmark reports.
