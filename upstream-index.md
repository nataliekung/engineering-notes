# Upstream contributions

Status as of 2026-10-07. "Open" means rebased on the current default branch and mergeable, waiting on maintainer review.

## Ray

| item | type | opened | status |
|---|---|---|---|
| [#66073](https://github.com/ray-project/ray/pull/66073) Do not FATAL when an actor is missing its task spec on GCS init | PR, C++ core (GCS) | 2026-09 | merged 2026-09-23 into master; not in ray-2.59.0 |
| [#65405](https://github.com/ray-project/ray/pull/65405) Return structured 503 when job logs are unavailable | PR, dashboard jobs | 2026-08-12 | open, approved by two reviewers |
| [#65402](https://github.com/ray-project/ray/issues/65402) Logs endpoint returns 500 after a completed job's worker node is removed | issue with two-node repro | 2026-08-11 | open |

## KubeRay

| item | type | opened | status |
|---|---|---|---|
| [#5132](https://github.com/ray-project/kuberay/pull/5132) Reject worker group removal in default installs | PR, CRD CEL rule | 2026-08-12 | open, CI green |
| [#5143](https://github.com/ray-project/kuberay/pull/5143) Surface orphaned worker group Pods via status condition and event | PR, controller | 2026-08-13 | open, CI green |
| [#5144](https://github.com/ray-project/kuberay/pull/5144) Validate that worker group names are valid label values | PR, validation | 2026-08-13 | open, CI green |
| [#5142](https://github.com/ray-project/kuberay/issues/5142) Safe worker group removal via pre-deletion Ray node draining | proposal | 2026-08-13 | open, one comment |

## verl

| item | type | opened | status |
|---|---|---|---|
| [#7330](https://github.com/volcengine/verl/pull/7330) Finalize nsys capture range in continuous profiling | PR | 2026-08-09 | open |
| [#7343](https://github.com/volcengine/verl/pull/7343) Keep loop workers on the job's node group | PR | 2026-08-10 | open |
| [#7366](https://github.com/volcengine/verl/pull/7366) Surface bucketed receiver failures | PR, weight transfer | 2026-08-12 | open |
| [#7382](https://github.com/volcengine/verl/pull/7382) Fail fast when a rollout server actor is unschedulable | PR, rollout | 2026-08-13 | open |
| [#7399](https://github.com/volcengine/verl/pull/7399) Remove invalid `super().__post_init__()` in `RewardManagerConfig` | PR, config | 2026-08-13 | open |
| [#7405](https://github.com/volcengine/verl/issues/7405) Optional fail-fast detector for degenerate "no learning signal" runs | RFC | 2026-08-14 | open |

## vLLM

| item | type | date | status |
|---|---|---|---|
| [RFC #37003](https://github.com/vllm-project/vllm/issues/37003) Context-Aware KV-Cache Retention API | comment offering a block-level retention prototype (`retention_deadline_ns`, `retain()`, `popleft_evictable()`) as a draft PR | 2026-10-07 | waiting on the RFC authors so the work does not duplicate theirs |

## Added 2026-10-29

| item | type | status |
|---|---|---|
| [Ascend/TransferQueue#206](https://github.com/Ascend/TransferQueue/pull/206) `TQ_RAY_NAMESPACE` to keep concurrent jobs' controllers apart | PR, with tests | open |
| [verl#8445](https://github.com/verl-project/verl/pull/8445) Correct the legacy trainer deprecation message | PR | open |
| [verl#8446](https://github.com/verl-project/verl/pull/8446) Install a transformers version vLLM 0.24.0 accepts in `Dockerfile.stable.vllm` | PR | open |
| [verl#8447](https://github.com/verl-project/verl/issues/8447) `setup.py` extras dropped by `pyproject.toml`; docs still use `.[trtllm]` | issue | open |
| [verl#7899](https://github.com/verl-project/verl/issues/7899#issuecomment-6504217618) TransferQueue write sites of the equal-length mRoPE layout | comment with a run of the packing function | open thread |
| [flashinfer#3634](https://github.com/flashinfer-ai/flashinfer/issues/3634#issuecomment-6504298371) Unset `MAX_JOBS` JIT build in a memory-limited container | comment | open thread |
| [flash-attention#2638](https://github.com/Dao-AILab/flash-attention/issues/2638#issuecomment-6497707174) flash-attn 2.8.3 builds against torch 2.11 and CUDA 13.0 for sm90 | comment | open thread |
