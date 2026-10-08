# Five verl fixes

All five are open pull requests against verl as of 2026-10-07, rebased on current `main` and mergeable. Each came out of running RL training on a shared Ray cluster, where a hang or a silent misplacement costs GPU hours. Each has a CPU-only test so the behavior is checkable without a GPU.

## 1. nsys profiling never finalized its report in continuous mode (#7330)

`RayWorkerGroup` set nsys `capture-range-end` to `repeat-shutdown:{6 * len(profile_steps)}`. That assumes discrete profiling, which emits six `cudaProfilerApi` ranges per profiled step. With `discrete=False`, which is the default, the profiler opens one range for the whole step, so only `len(profile_steps)` ranges are ever produced. nsys keeps waiting for ranges that never come, and the `.nsys-rep` only appears when the worker process exits. If you read it right after the profiled step, there is nothing there.

The fix passes `profiler_discrete` from the trainer config into the worker group and scales `repeat-shutdown` by the number of ranges actually produced per step: one in continuous mode, six in discrete. Discrete mode is unchanged. 12 lines added, 1 removed, no public API change.

## 2. Agent-loop and reward-loop workers landed on the wrong nodes (#7343)

verl's agent-loop and reward-loop managers placed their lightweight workers round-robin over every alive node returned by `ray.nodes()`, with a soft node affinity. On a shared or heterogeneous Ray cluster that list includes nodes that belong to other jobs and do not run this job's runtime image. A worker placed there fails on import with `ModuleNotFoundError`. A soft affinity also falls back to a foreign node whenever the targeted node is momentarily full.

The change adds an optional environment variable, `VERL_LOOP_WORKER_NODE_RESOURCE`, naming a Ray custom resource that only this job's nodes advertise. When set:

- candidate nodes are filtered to those advertising the resource, with a clear error if none do;
- each worker requests a tiny amount (1e-4) of that resource, so Ray refuses to schedule it anywhere else, closing the soft-affinity leak;
- workers are spread toward the nodes with the most available CPU instead of stacking on the first node, falling back to round-robin when Ray does not report per-node availability.

Unset, behavior is unchanged. 431 lines added across 7 files. Tests build three local raylets, two with the resource and one without, and assert that every worker lands on the two. They cover actor creation and placement only, not a full training run.

## 3. `RewardManagerConfig()` raised on construction (#7399)

`RewardManagerConfig.__post_init__` called `super().__post_init__()`, but its base class `BaseConfig` and everything above it in the MRO define no `__post_init__`. Default construction raised `AttributeError: 'super' object has no attribute '__post_init__'`, and default construction is exactly what `RewardConfig` does through `field(default_factory=RewardManagerConfig)`.

The fix removes the call, which matches how the other direct `BaseConfig` subclasses (`ActorConfig`, `CriticConfig`, `EngineConfig`, `PolicyLossConfig`) are written. New tests cover default construction, the `source="importlib"` module-path validation, and the dict-like interface. 55 lines added, 1 removed.

## 4. A failing weight-transfer receiver hung the sender forever (#7366)

The bucketed ZMQ weight-transfer protocol sends weights to rollout workers in buckets and waits for an ACK after each. If the receiver failed during initialization, metadata decoding, or its per-bucket callback, the sender waited for an ACK that never came, with no timeout and no error.

The change bounds every ACK wait and makes the receiver send an explicit error ACK that carries its exception back to the sender, so the failure surfaces where the operator is looking. The timeout is exposed as `rollout.checkpoint_engine.update_weights_ack_timeout_seconds`, default 30 seconds, and must be positive and finite. Tests run the protocol over shared memory and cover a clean transfer, receiver failures at each step including after an intermediate bucket, final-ACK timeout after peer loss, and cleanup of IPC sockets and shared memory. 438 lines added, 33 removed. CUDA IPC and a full vLLM integration were not exercised.

## 5. An unschedulable rollout server hung the whole job (#7382)

A colocated rollout replica creates its per-node server actor with a hard `NodeAffinitySchedulingStrategy(soft=False)` pinned to its training worker's node. If that node has no free slot, the actor sits in `PENDING_CREATION` and the first `.remote()` call on it blocks forever. `LLMServerManager.create()` hangs at first init with no timeout and no diagnostic, and the job holds its GPUs doing nothing until something external kills it.

The change adds `wait_rollout_servers_scheduled`, which polls Ray actor state and returns as soon as every server actor has left `PENDING_CREATION` and `DEPENDENCIES_UNREADY`. An already-scheduled actor passes immediately, so slow engine initialization is not affected; only a real placement failure trips the deadline (300 seconds by default). On timeout it raises a `RuntimeError` naming each stuck actor, its state, its node and its required resources, and suggests reducing per-node requests or using a layout with at least two usable GPUs per node for cross-node hybrid rollout. The guard is wired into the vLLM, SGLang and TensorRT-LLM backends right after the server actors are created. Four CPU tests cover the alive, transient-pending, stuck, and missing-actor cases. 131 lines added, 3 removed. Closes verl issue #7381.
