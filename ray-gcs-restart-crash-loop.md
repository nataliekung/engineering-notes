# A dangling actor row that crash-loops Ray GCS on restart

Ray's Global Control Store (GCS) runs on the head node and holds cluster-level state: jobs, actors, placement groups, nodes. With GCS fault tolerance enabled, that state is persisted to an external Redis and replayed when the GCS starts. I hit a bug in that replay path that took a shared cluster down on every restart. The fix is merged into Ray master as ray-project/ray#66073.

## Symptom

The head pod went into CrashLoopBackOff. Every start ended with:

```
[C] (gcs_server) container_util.h:163: Key <actor_id_hex> doesn't exist
RuntimeError: Failed to start GCS.
```

A fresh cluster with empty Redis state started fine. Only restarts that restored persisted state died. The only way to recover was to wipe the persisted GCS state, which throws away exactly what fault tolerance is there to keep.

## Where the fatal comes from

`GcsActorManager::Initialize()` iterates the persisted actor table. For each actor it decides to load, it looks up the task spec with `map_find_or_die()`. The key in the fatal log is an actor id, so a row in the actor table had no matching row in the task-spec table, and the lookup aborted the whole `gcs_server`.

That lookup is not the bug. The task spec is always written before the actor row, so the normal write path cannot produce a loadable actor without a spec. Something had to delete the spec afterwards and leave the actor row behind.

## The actual bug

`Initialize()` does that itself. When it decides not to load an actor, for example because its owning job is dead, it appends the actor to a `dead_actors` list and later batch-deletes those actors' task specs. It never updated the actor-table row. The row kept its old non-DEAD state, such as `ALIVE` or `DEPENDENCIES_UNREADY`.

So the sequence is:

1. Restart A: `Initialize()` de-selects an actor, deletes its spec, leaves the row non-DEAD.
2. Persisted state is now inconsistent. Nothing fails yet.
3. Restart B: `Initialize()` reloads that row as non-DEAD, looks up its spec, and `map_find_or_die()` aborts.

The cause and the crash are one restart apart. The stack trace points at the lookup, not at the cleanup that created the inconsistency.

## Fix

Two changes in `gcs_actor_manager.cc`.

`OnInitializeActorShouldLoad()` now uses `find()` and returns false when the spec is missing. A missing spec isolates that one actor instead of aborting the process.

When `Initialize()` de-selects an actor that is not already permanently dead, it marks the actor DEAD, persists the row, publishes the death, and deletes the task spec only from inside the `ActorTable().Put()` completion callback. This mirrors what `DestroyActor()` does during normal teardown. If the process dies between the two writes, the worst case is a DEAD row whose spec is still present, which is harmless and gets cleaned up on the next start. A non-DEAD row without a spec can no longer be produced.

Actors that were already permanently dead keep the existing behavior: their spec is removed by the end-of-loop batch delete.

## Tests

Two gtest cases in `gcs_actor_manager_test.cc`:

- `TestInitializeMarksDeadWhenOwnerJobDeadAndReloads`: an actor whose owning job is dead but whose spec still exists. `Initialize()` marks it DEAD, persists that, deletes the spec. A second `GcsActorManager` built from the persisted state starts, reports the actor DEAD, and does not re-register it.
- `TestInitializeIsolatesActorWithMissingTaskSpec`: a non-DEAD row with no spec does not abort `Initialize()` and ends up DEAD.

## What I took from it

When two persisted tables have to agree, the write order is part of the contract. Write the tombstone first, then delete what it refers to. And when a crash only shows up after a restart, the log line is pointing at the reader, not at the writer that broke the invariant.

The fix is on Ray master and not in a tagged release as of ray-2.59.0.
