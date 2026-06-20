---
title: "Karpenter Rollout"
draft: false
date: 2026-06-19T15:10:00+05:30
description: "A short practical summary of migrating from old Karpenter and ASG node groups to Karpenter v1.7.2, covering safe cutover, v1 API changes, disruption budgets, GitLab runner cost issues, and workload-specific node pools."
---

**Goal.** Replace old Karpenter `v0.34` and ASG node groups with Karpenter `v1.7.2` across non-prod and prod — with no CI/CD downtime.

## The Safe Cutover

A Managed Node Group sat under `shared-cluster` as fallback.

Key insight: only pods pinned via `nodeSelector` needed touching, tolerations don’t force scheduling.

Remove the `nodeSelector`s, workloads drift to the Managed Node Group, and old Karpenter can be torn down safely.

The ARM64 Graviton runner was the exception — there was no amd64 fallback.

## The v1 API Papercuts

Predictable breakages appeared during the migration:

- `nodeClassRef` needs full GVK.
- `expireAfter` moved under `template.spec`.
- `WhenUnderutilized` became `WhenEmptyOrUnderutilized`.
- `AL2` moved to `AL2023` AMIs.

We versioned the base as `prod-base-v1.7.2` instead of editing in place, so old `v1beta1` clusters kept working during the rollout.

## The Custom Networking Problem and `RESERVED_ENIS`
With AWS VPC CNI custom networking, pod IPs came from secondary ENIs via ENIConfig, while the primary ENI stayed with the node. Karpenter was still calculating pod density as if all ENIs were available, so it overestimated node capacity. As a result, Kubernetes scheduled more pods than AWS CNI could assign IPs for, causing pod sandbox/IP assignment failures.

The fix was to configure Karpenter with:

```yaml
RESERVED_ENIS: "1"
```

**Before:** Karpenter thought the node had more pod IP capacity than AWS CNI could actually provide. <br/>
**After:** `RESERVED_ENIS=1` made Karpenter subtract one ENI from capacity calculation. <br/>
**Result:** Pods scheduled according to real available ENI/IP capacity, so IP allocation failures stopped.


## The Cost Surprise on Shared Cluster

GitLab runner bursts landed short-lived CI pods on the general `karpenter-nodepool` the same pool used by long-lived apps.

Karpenter launched big `4xlarge` nodes for the burst. When the jobs finished in around 10–30 minutes, those nodes went idle.

But that pool had strict node disruption budgets during business hours to protect production services. Because of that, consolidation was blocked, and `4xlarge` nodes sat idle for days.

## Why This Was New With Karpenter

Cluster Autoscaler only added or removed nodes. It never actively repacked pods.

Karpenter’s `WhenEmptyOrUnderutilized` consolidation actively moves workloads onto fewer nodes. That is useful for cost saving, but it also means disruption budgets become more important.

Those strict budgets were correct for production services, but they were not meant for short-lived GitLab runner pods.

## The Fix

Split workloads instead of clamping one shared pool.

We created a dedicated GitLab runner node pool:

```yaml
karpenter-nodepool-gitlab-runners
````

This pool had:

* Its own taint.
* Max `2xlarge` instance size.
* Spot capacity.
* 40% disruption budget.
* No schedule-based disruption blocks.

Now runner nodes can scale up and down freely in around 30 minutes, while the general pool keeps strict disruption budgets for real services.

## The Lasting Design

Prod got purpose-built node pools:

* **Stateful workloads:** on-demand, `WhenEmpty`, never spot.
* **Long jobs:** protected with `do-not-disrupt`.
* **Batch workloads:** spot, fast reclaim.
* **Apps:** balanced, PDB-protected.

Each pool now has the disruption behavior it actually needs.

## The Lesson

A disruption budget protects stability, but it can also block cost saving.

If it is too strict, you may end up paying for idle nodes.

Do not mix ephemeral workloads and long-lived workloads in one pool. Isolate them and match the consolidation policy to each workload type.

