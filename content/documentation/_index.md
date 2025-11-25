---
title: Deckhouse Delivery Kit Documentation
linkTitle: Documentation
description: Documentation for the project, including guides, API references, and tutorials.
weight: 10
params:
  no_list: true
cascade:
  params:
    simple_list: true
---

{{< alert level="warning" >}}
The functionality of the Deckhouse Delivery Kit module is only available if you have a license for any commercial version of the Deckhouse Kubernetes Platform.
{{< /alert >}}

Deckhouse Delivery Kit is software designed for efficient delivery of arbitrary applications into Kubernetes.

Deckhouse Delivery Kit enables the organization of:

* Building images
* Distributing images to container registries
* Cleaning up container registries of built images
* Distributing Helm charts
* Distributing bundles — Helm charts and associated images as a single entity
* Deploying Helm charts and bundles into Kubernetes clusters
