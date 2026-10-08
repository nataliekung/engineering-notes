# Removing a worker group from a RayCluster: four changes to make it safe

KubeRay issue #1739 (2023) describes what happens when you delete an entry from `RayCluster.spec.workerGroupSpecs` on a running cluster: the operator's `reconcilePods` only iterates groups still present in the spec, so the removed group's Pods are never listed again. They keep running, nobody scales or cleans them up, and the head autoscaler crash-loops. The maintainers agreed the short-term fix was to reject the update, and that the real blocker was the lack of a way to drain Ray nodes before deleting their Pods.

As of 2026-10-07 nothing in a default Helm install rejects that transition. I sent three PRs and one proposal that cover the problem from different sides. All four are open.

## 1. Reject the removal at the CRD level (#5132)

Default installs do not enable the admission webhook, so a webhook rule would not protect most users. This change adds a CEL transition rule on `RayCluster.spec` that rejects removing a worker group:

- Adding groups, editing fields in place, and reordering are still allowed.
- Removal through `RayService.spec.rayClusterConfig` is still allowed, because a RayService provisions a replacement cluster instead of editing the existing one.
- Head-only clusters are unaffected; the rule is guarded with `has()`. Removing the last group is still rejected.

Two bounds were needed to make the rule installable: `workerGroupSpecs` gets `MaxItems=100` and `groupName` gets `MaxLength=63`. Without them the estimated CEL cost exceeds the apiserver's per-expression limit and the CRD fails to install. The reasoning is documented inline next to each bound. envtest on Kubernetes 1.29 covers add, modify, reorder, rejected removal on a standalone cluster, and allowed removal through a RayService template. 268 lines added across 16 files, mostly tests and Helm unit tests.

## 2. Surface clusters that are already broken (#5143)

Rejecting new removals does nothing for clusters that already have orphaned Pods. This change makes `calculateStatus` detect Pods whose `ray.io/group` label no longer maps to a group in the spec and report them:

- A status condition `OrphanedWorkerGroup` listing the group names and Pod count.
- One warning event per transition into the orphaned state, so it does not spam every reconcile.

It never deletes or mutates the Pods. The condition clears when the group is restored or the Pods are gone, and terminating Pods are ignored. Gated behind the existing `RayClusterStatusConditions` feature gate. 215 lines, two unit tests.

## 3. Validate group names before they become label values (#5144)

`groupName` is written verbatim into the `ray.io/group` Pod label, and nothing validated it as a label value. A name with a space or a slash is accepted on the RayCluster and fails later, repeatedly, at Pod creation with an apiserver error about `metadata.labels`. The change validates the name with `validation.IsValidLabelValue` inside `ValidateRayClusterSpec`, which runs on every reconcile and is shared by RayJob and RayService validation, so it protects default installs.

It deliberately uses label-value rules rather than the stricter DNS-1123 label rules, because label values allow uppercase and existing clusters with mixed-case group names must keep working. Tests cover a lowercase name, a mixed-case name, and names containing a space and a slash.

## 4. Proposal: drain before delete (#5142)

Rejection is not the end state. Teams do need to retire a worker group. The proposal is an explicit, opt-in removal that drains the group's Ray nodes before deleting their Pods:

- Reuse Ray's existing `DrainNode` GCS RPC (forwarded to the raylet as `DrainRaylet`). It has two modes: `IDLE_TERMINATION`, which is rejected if the node is busy and expects the caller to retry, and `PREEMPTION`, which cancels leases immediately. Autoscaler v2 already uses this path internally.
- Build on #5005, which adds `spec.gracefulTerminationOptions` and a `preStop` hook that polls the idle drain and falls back to preemption on a deadline. The proposal covers what #5005 scoped out: orchestrating deletion for removed and scaled-down groups.
- Unrequested removal stays rejected (coexists with #5132). No Ray core changes. Default off, backward compatible on `ray.io/v1`.

The proposal pins its line references to a specific commit and lists the unconditional delete paths in the controller that would need to route through the drain. The known risk is that `ray drain-node` is explicitly not a public Ray API.

## Status

All three PRs are green on CI and rebased on current `master`; the proposal has one comment. None has had a maintainer review yet.
