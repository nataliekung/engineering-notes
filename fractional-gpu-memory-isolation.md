# Sharing one GPU between two processes: what the torch memory fraction protects, and what it does not

If two inference replicas share a physical GPU, one of them can starve the other of memory. `torch.cuda.set_per_process_memory_fraction` is the usual answer. I measured what it actually guarantees on an H200 (139.8 GiB).

## The experiment

Two separate OS processes on one GPU:

- B, the well-behaved neighbor, allocates 21.6 GiB, then grows in 7 GiB steps on a schedule, writing a heartbeat as it goes. It "survives" if it completes its schedule.
- A, the greedy neighbor, allocates as fast as it can with no self-limit until it hits an out-of-memory error, then holds what it has.

Run once with no cap, and once with both processes calling `set_per_process_memory_fraction(0.4)` at startup (a 55.9 GiB cap each).

| | no cap | fraction 0.4 in both |
|---|---|---|
| A peak reserved | 107.8 GiB | 50.3 GiB |
| B outcome | OOM on its second growth step; only 5.4 GiB free on the card | grew to 43.1 GiB and heartbeated to the end |

With the fraction set, the greedy process ran into its own cap and the neighbor finished. Without it, the greedy process took the card.

## Where the fraction stops working

The fraction is a limit inside PyTorch's caching allocator. It does nothing for memory allocated by anything else. With the 0.4 cap set in one process, I tried to allocate 97.86 GiB three ways:

| allocation path | result |
|---|---|
| `torch.empty` | OOM, cap held |
| raw `cudaMalloc` through ctypes | succeeded, past the cap |
| CuPy | succeeded, past the cap |

So a neighbor that uses CuPy, a custom CUDA extension, or any library with its own allocator is not bounded by the torch fraction at all.

## MPS gives a hard limit

With the CUDA MPS control daemon running and `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` set to 40G for the client, the same raw `cudaMalloc` test:

| attempted | result |
|---|---|
| 30 GiB | succeeded |
| 50 GiB | failed |
| 97.86 GiB | failed |

MPS enforces at the driver level, so it bounds every allocation path. It also lets small kernels from two processes overlap instead of time-slicing between two CUDA contexts, which is the other reason to use it for co-located small models. I did not record timing numbers for that part, so I am not claiming a figure.

## What to take from it

- The torch fraction is enough when every co-located process is a PyTorch model that allocates only through torch. It protects well-behaved neighbors from each other.
- It is not an isolation boundary. For a mix of workloads, use MPS with a pinned memory limit, or separate devices, or MIG where the hardware supports it.
- Whichever mechanism you pick, test it the way above: a greedy process and a scheduled one, on the real card, and check that the scheduled one finishes.
