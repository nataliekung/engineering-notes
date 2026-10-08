# RFC: fail fast when a GRPO run has no learning signal

verl RFC #7405, opened 2026-08-14, open with no maintainer response as of 2026-10-07.

## The failure

GRPO normalizes each prompt group's rewards by the group's standard deviation:

```python
id2std[idx] = torch.std(scores_tensor)
scores[i] = (scores[i] - id2mean[index[i]]) / (id2std[index[i]] + epsilon)
```

If every response in a group gets the same reward, because the reward function returned a constant or because every sample collapsed to the same score, the standard deviation is zero and every advantage in that group is `(score - mean) / epsilon = 0`. If that holds for every group for the whole run, the policy gradient is zero, `actor/grad_norm` is about zero, and nothing is learned. The run still exits 0 and checkpoints a model identical to its initialization. The only place the problem is visible is the metric charts, after the GPU hours are spent.

Two existing issues show users hitting this: #5211 reports `reward=0, loss=0, grad_norm=0` in a GRPO example and asks where the error is; #637 asks for NaN checks on loss, gradients and weights, a sibling failure mode.

## The proposal

An opt-in, off-by-default monitor in the fit loop:

- Warmup: ignore the first `warmup_steps`, because early steps can legitimately have near-zero gradients while rollouts diversify.
- Signal: after warmup, mark a step dead when `actor/grad_norm <= grad_eps`, optionally corroborated by zero in-group advantage variance for the batch.
- Trip: when dead steps form a consecutive window (or a whole-run fraction) beyond a threshold, terminate with an error that names the cause and the two things to check: a constant reward function and zero in-group reward variance.

```yaml
trainer:
  no_learning_signal_check:
    enable: false
    warmup_steps: 20
    grad_eps: 1e-9
    max_consecutive_dead_steps: 50
```

When disabled the loop is unchanged. The monitor reads a metric the loop already computes, so there is no per-step cost on the common path.

## How it differs from what exists

- #7069 (closed) added `early_stop_patience`, which stops when reward has plateaued at a converged value and treats that as success. This detector does the opposite: it fails loudly when there is no signal at all.
- #4786 discusses why GRPO variance collapses with continuous rewards. It explains the mechanism but does not detect it.

The RFC asks whether the direction and config surface are acceptable before sending the detector, config plumbing and tests as a PR.
