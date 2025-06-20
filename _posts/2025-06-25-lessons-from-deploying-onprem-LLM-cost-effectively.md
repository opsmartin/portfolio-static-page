---
layout: post
title: "Lessons from deploying on-premises LLM cost effectively"
date: 2025-06-25
categories:
  - AI
  - LLM
  - On-Premises
  - Infrastructure
  - CD
  - Cost-Optimization
---

Developing infrastructure for [M. Zilinec](https://linkedin.com/in/matus-zilinec)'s [on-prem LLM powered data extraction pipeline](https://pactus.ai/),
one of the first of its kind to process the Czech language.

## Problem at hand

- We needed to create infrastructure to support an on-premises LLM document extraction pipeline
- We didn't have any reliable bare metal GPU machine at hand
- We needed to maintain low costs as much as possible, as cloud GPU virtual machines can get *EXPENSIVE*

## Devising an architecture

- The app backend/frontend + all scheduling components run in a single VM.
- It sends inquires to GPU worker instances, which only run the GPU workloads.
- We need to save cost by autoscaling GPU instances, as the cheapest GPU instances with 16GB memory are 400 EUR per month (330 GBP)

This makes deploying the stack inside Kubernetes an obvious choice.

## Challenges

- GPU resource management in Kubernetes is not optimal
- CUDA python images are BIG (6GB).
- Machine learning models are EVEN BIGGER (20+ GB).
- GPU instance boot-up time is LONG (5+ minutes).

### GPUs in k8s

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-worker-llm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-app
  template:
    metadata:
      labels:
        app: gpu-app
    spec:
      containers:
      - name: cuda-container
        image: nvidia/cuda:12.9.0-cudnn-devel-ubuntu24.04 # 6GB just the image! +20GB ML models mounted
        command: ["nvidia-smi"]
        resources:
          limits:
            nvidia.com/gpu: 1  # Request 1 GPU
      restartPolicy: Always
    tolerations:
      # This taint or similar is added on GPU nodes
      # by the cloud provider to select out GPU-required workloads
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "present"
        effect: "NoSchedule"
```

The main takeaways here are:

- The GPU k8s resource is **discrete**
- The CUDA image required to run our workload is also **very big**

The bottleneck for workloads in these cases will most likely be **GPU VRAM limitations**. The VM we used had 16GB GPU VRAM. However, as you can see, the **allocation can not be controlled**. In case you need two pods to share a GPU, or control which pods get which GPU instances, **you must create the logic by which they schedule to meet their chip and VRAM requirements.**

The cloud provider usually pre-installs a GPU driver on the GPU node image. An AMI with pre-installed drivers speeds up the node spin up, although the pre-installed driver might not always be the latest. So For latest GPU features, consider installing it with the [NVIDIA gpu-operator](https://github.com/NVIDIA/gpu-operator) instead.

> 📝 **Note:**
> High-end GPUs support [multi-instance GPU](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/), but this is not available for the lower grades. There is also interesting work undergoing to make this simpler, such as [Project-HAMi](https://github.com/Project-HAMi/HAMi) or [training GPU schedulers with Reinforcement learning](https://www.youtube.com/watch?v=fzT6Ot_PTQ0).

### Autoscaling trigger

I had to implement the apps queue size as a custom metric to trigger GPU autoscaling.
This means designing a custom scaling event based on a custom metric, which represents the number of user requests in queue. The [KEDA operator](https://github.com/kedacore/keda) perfectly covered this use case:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-worker-llm
  namespace: gpu-worker
spec:
  scaleTargetRef:
    name: my-worker-llm  # The name of deployment to scale in this namespace
  maxReplicaCount: 5
  minReplicaCount: 0
  triggers:
    - type: prometheus  # Trigger scaling based on prometheus query
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc.cluster.local:9090  # Prometheus address
        metricName: "my_queue_size_total" # Custom queue metric
        threshold: "10"  # Scales up 1 pod per 10 of my_queue_size_total
        query: sum(my_queue_size_total)  # The Prometheus query to check for the metric
        activationThreshold: "1"  # Scale up when at least 1 in queue
```

I do not recommend using [prometheus-adapter](https://github.com/kubernetes-sigs/prometheus-adapter).
`prometheus-adapter` fired up first in search, but it has complicated syntax. I got stuck with the syntax for hours, whereas I achieved the same with KEDA in minutes.

### Spot vs Regular

We needed to save costs where possible, so at first we thought even Spot instances could be enough.
Sometimes the spot instances will not spin up for 15+ minutes.  It is definitely **only** meant for delayable batch workloads.

We quickly switched to Regular, which boots up in 1 minute.

### Images are TOO BIG

The images, even when stored 'locally' in an OCI registry, in the same region as the k8s cluster,
need to be downloaded on each GPU node.

Same region data transfer fortunately does not incur cost, but storing the images in a registry does.

| Tier     | Minimum Download Bandwidth  |
|----------|-----------------------------|
| Basic    | 30 Mbps                     |
| Standard | 60 Mbps                     |
| Premium  | 100 Mbps                    |

Switching to 'Premium' tier made the registry a more costly, but did not effectively speed up the download.
The specs are **minimum** bandwidth and we already had the max advertised speed even with *Basic*. The specs are therefore not for our usecase but just for meeting SLAs and such, or for a larger number of nodes.

### Volume mount hacks

[Loki's Wager](https://lokiwager.github.io/about/) posted a perfect blog tackling a similar problem by [mounting all big files as a read-only volumes in k8s](https://lokiwager.github.io/posts/reduce-image-size), then only boot up a minimal image such as Alpine.
This is nice and definitely works, but requires effort digging through parts of the image and re-mounting it correctly via Kubernetes `PersistentVolumes`.
You also have to version each `PersistentVolume` if the files differ for each image.

It was not easy to keep track of the changing ML models being used as well as the current CUDA image. We therefore felt this was too hackey and searched for another solution.

Kubernetes 1.31 introduces a [mounting images as volumes feature](https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/). The images still must be downloaded from the Container registry and this not does speed up the node boot duration. However, this can make the transition to the aforementioned volume mount hack simpler.

### Disk speed

The disk speed in Azure VMs is determined by the size of the VM itself. Same VM series can have a lot slower/faster disk IO depending on vCPUs/RAM size.

It is not entirely transparent what disk IO speeds you get. While cloud providers disclose the IOPS value, we still had to conduct measurements ourselves to get an accurate idea.

The image pull gets noticably upon switching to the `Premium_LRS` disk tier. It was however not fast enough for us to make a difference in the product user experience.

### Deallocate vs Delete

Finally, we settled on detaching the VM hard drives and binding them back upon node launch.

This doesn't work with the `Premium_LRS` disk tier. The initial launch of the nodes is therefore a lot slower. Fortunately, that didn't matter for our product experience, because re-attaching the disk with all images already downloaded, is very fast.

The re-mounted disk also contains all models, which were previously streamed from a bucket.

The cost is higher than in the *volume mount hack* method - Azure bills each detached node hard drive separately.

In k8s, the *detached* nodes never disappear from the nodes list, but are instead marked as `NotReady`.

This got us down to **1 minute service ready speed** - from the initial user query to starting processing the workload - without incurring cloud costs for costly GPU virtual machines while the service is idle.
