# vLLM vs SGLang on rollout-style generation, one A10G

RL post-training spends most of its wall clock in the rollout step: sample many completions from a fixed policy, often several per prompt. This benchmark isolates that step and runs it through each engine's OpenAI-compatible server. It is not a training loop. There is no weight sync, no actor update, and no KV reuse across steps.

## Setup

- Model: `Qwen/Qwen2.5-1.5B-Instruct`.
- vLLM 0.11.0 and SGLang 0.4.9.post2, each launched as its own server with the same settings: GPU memory utilization 0.6, max model length 4096, tensor parallel 1, seed 1234.
- Sampling: temperature 1.0, top_p 0.95, max_tokens 512 unless the scenario says otherwise.
- GPU: one NVIDIA A10G (24 GB).
- Client: Python standard library, a thread pool for concurrency. Prompts are generated with varied input lengths.
- Reported KV capacity: vLLM 366,624 tokens, SGLang 375,066 tokens.

## Scenarios

| name | requests | concurrency | samples per prompt | max_tokens |
|---|---|---|---|---|
| independent | 512 | 64 | 1 | 512 |
| grouped (GRPO-style) | 64 | 32 | 8 | 512 |
| variable-length tail | 256 | 64 | 1 | per request, drawn from a long-tailed distribution |

The grouped scenario is the shape GRPO produces: n completions per prompt sharing one prefix. The tail scenario caps each request at a different length so a few requests run much longer than the rest.

## Results

| scenario | engine | output tok/s | p50 latency | p99 latency | p99 / p50 | GPU memory (MiB) |
|---|---|---|---|---|---|---|
| independent | vLLM | 2962 | 10.94 s | 11.06 s | 1.01 | 14637 |
| independent | SGLang | 3675 | 8.72 s | 8.89 s | 1.02 | 14839 |
| grouped | vLLM | 3950 | 33.17 s | 33.24 s | 1.00 | 16241 |
| grouped | SGLang | 9417 | 13.54 s | 13.88 s | 1.03 | 15701 |
| tail | vLLM | 2314 | 2.61 s | 10.97 s | 4.21 | 16241 |
| tail | SGLang | 2271 | 2.57 s | 10.86 s | 4.23 | 15703 |

SGLang over vLLM: 1.24x on independent sampling, 2.38x on grouped sampling, 0.98x on the tail scenario.

## Reading it

- The grouped case is the one that matters for GRPO, and it is where the gap is largest. Eight samples per prompt share a prefix, and SGLang's radix cache reuses it. vLLM here ran with default flags, so prefix caching was whatever the default was in 0.11.0 for this launch path, and I did not tune either engine.
- On the long-tail scenario the engines are equal, and the tail ratio is about 4.2 for both. The p99 is set by the longest requests in the workload, not by the scheduler. Changing engines does not fix a tail that comes from the length distribution.
- Memory is a wash at this model size.

## What this does not tell you

- Nothing about weight sync, which is the other large cost in an RL step and depends on the training framework, not the engine.
- Nothing about larger models or tensor parallel. One 1.5B model on one card.
- Nothing about tuned configurations. Both servers ran with defaults apart from the shared settings above. A tuned vLLM with prefix caching confirmed on would be the fair next comparison for the grouped case.

Raw results are per-scenario JSON with latency percentiles, completion-token statistics and failure counts; no requests failed in any scenario.
