# torch.nested picks the jagged dimension from the data, and equal lengths break the guess

Multi-component position ids, as used for mRoPE in vision-language models, are stored per sample as a `(C, L)` tensor: `C` components (4 for Qwen-style models) over `L` tokens. A batch of such samples with different `L` is a natural jagged tensor whose ragged axis is the token axis. A batch where every `L` is the same is not, because `torch.nested.as_nested_tensor(..., layout=torch.jagged)` decides which axis is ragged by looking at the shapes, and equal shapes leave nothing to look at.

## What torch does

On torch 2.13.0, CPU, with `C = 4`:

```python
import torch
C = 4
for lens in ([5, 7, 9], [7, 7, 7], [7]):
    nt = torch.nested.as_nested_tensor(
        [torch.arange(C * l).reshape(C, l) for l in lens], layout=torch.jagged
    )
    print(lens, tuple(nt.shape), nt._ragged_idx, tuple(nt.values().shape), nt.offsets().tolist())
```

```
[5, 7, 9]  (3, 4, j1)  ragged_idx=2  values=(4, 21)  offsets=[0, 5, 12, 21]
[7, 7, 7]  (3, j2, 7)  ragged_idx=1  values=(12, 7)  offsets=[0, 4, 8, 12]
[7]        (1, j3, 7)  ragged_idx=1  values=(4, 7)   offsets=[0, 4]
```

With unequal lengths the token axis is ragged and the values are `(C, sum(L))`. With equal lengths, and with a single sample, the component axis is ragged: the values are `(B * C, L)` and the offsets step by `C`. Both are valid nested tensors. They are different layouts for the same logical data, and nothing in the call says which one the caller wanted.

## Where it bites

Code that assumes the token axis is ragged and relabels the tensor to match breaks on the second layout. Setting `_ragged_idx = 2` on the equal-length tensor above changes the metadata without changing `values` or `offsets`, and the next `unbind(0)` fails (`RuntimeError: Expected cond to be True, but got False` here; the `split_with_sizes` error in the upstream reports is the same inconsistency surfacing in another torch/tensordict combination).

In verl's v1 trainer the relabel is `maybe_fix_3d_position_ids` in `verl/utils/tensordict_utils.py`, which sets `_ragged_idx = 2` on any 3D nested `position_ids`. The equal-length layout reaches it from more than one place:

- verl's own `left_right_2_no_padding`, fixed on main by [verl#8413](https://github.com/verl-project/verl/pull/8413) (merged 2026-10-29).
- TransferQueue, which rebuilds each batch it hands back by calling `as_nested_tensor(values, layout=torch.jagged)` on the per-sample values: `AsyncSimpleStorageManager._pack_field_values` ([simple_storage_manager.py#L799](https://github.com/Ascend/TransferQueue/blob/5c15a2c/transfer_queue/storage/managers/simple_storage_manager.py#L799)) and `_merge_tensors_to_tensordict` ([base.py#L744](https://github.com/Ascend/TransferQueue/blob/5c15a2c/transfer_queue/storage/managers/base.py#L744)). I ran `_pack_field_values` on `(4, L)` samples and got the same three layouts as above. TransferQueue cannot tell which axis of a `(C, L)` sample is the sequence axis, so the normalization belongs on the consumer side.

The trigger in practice is any batch where every sequence has the same length: a batch of one, or a run where generation hits the same length cap for every sample.

## Where the fix is being discussed

[verl#7899](https://github.com/verl-project/verl/issues/7899) has the analysis and a CPU reproduction, and [verl#7901](https://github.com/verl-project/verl/pull/7901) rebuilds the layout with tests; both are open as of 2026-10-29. I added the TransferQueue write sites to the issue ([comment](https://github.com/verl-project/verl/issues/7899#issuecomment-6504217618)). One approach in the thread avoids guessing from `position_ids` alone, which is ambiguous when `values` is square: compare its offsets with those of `input_ids`, whose offsets are the true sequence lengths.

## What I did not test

Only the CPU behavior above and TransferQueue's packing function were run for this note. I did not test other torch versions, so the exact error text may differ, and I did not run any model.
