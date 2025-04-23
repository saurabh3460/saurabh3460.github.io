---
title: "Kubernetes API Structure Explained"
draft: false
date: 2025-04-23T14:49:00.000Z
description: "A comprehensive guide to understanding Kubernetes API architecture, covering API groups, resources, kinds, and key concepts like GVK and GVR. Perfect for developers working with Kubernetes."
categories:
  - kubernetes
tags:
  - Kubernetes
  - API
---

{{< img
  src="k8s_api.png"
  alt="k8s"
  caption="Kubenretes api structure diagram" >}}

# Core Concepts

## API Groups
- Organize the API into logical sets of related resources.
- **Core group**: No name, accessed via `/api/v1`.
- **Named groups**: Have names, accessed via `/apis/<group name>/<version>`.
- Resource types are always referred to in **plural form** (e.g., `pods`).

## Resources
- API endpoints representing a collection of objects of a certain kind.
- **Examples**:
  - `/api/v1/pods`
  - `/apis/apps/v1/deployments`

## Resource Instance
- A specific occurrence of a resource type with a unique name.
- Unique **within its namespace** (for namespaced resources) or **cluster-wide** (for cluster-scoped resources).
- **Example**: A Pod named `web-server-1234`.

## Kinds
- Schema definitions for API objects.
- Three categories:
  - **Object kinds**: Persistent entities (e.g., `Pod`, `Deployment`).
  - **List kinds**: Collections of resources (e.g., `PodList`, `DeploymentList`).
  - **Special Purpose**: Non-persistent or action-oriented types (e.g., `Scale` for scaling operations, `Binding` for assigning Pods to Nodes).

## Objects
- Runtime instances of a Kind.
- Persistent entities stored in Kubernetes (e.g., in etcd).
- Represented in **YAML/JSON manifests** and **API requests**.

## API Versions
- Track maturity of resources within an API group.
- Progression: **Alpha** (`v1alpha1`) → **Beta** (`v1beta1`) → **Stable** (`v1`).

# Key Concepts

## GVK (Group Version Kind)
- Uniquely identifies a resource type.
- Format: `<group>/<version>, <kind>`.
- **Example**: `apps/v1, Deployment`.

## GVR (Group Version Resource)
- Locates a resource endpoint in the API.
- Format: `<group>/<version>/<resource>`.
- **Example**: `apps/v1/deployments`.

## Scheme
- Runtime mapping of GVKs to Go types and conversion functions.
- Enables **serialization/deserialization** and **API processing**.

# Conclusion
Whether you're defining YAML manifests, debugging API calls, or working on extensions, this knowledge is **foundational for Kubernetes success**.
