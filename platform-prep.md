# Platform Prep

Your model recipe tells you what hardware you need. This page gets your environment ready to run it.

## Where are you running?

| Environment | Product | Install docs |
|-------------|---------|-------------|
| OpenShift | Red Hat OpenShift AI (RHOAI) | [Installing OpenShift AI Self-Managed](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4) |
| CoreWeave | Red Hat AI Inference <!-- CONFIRM: Server or Stack? docs say "Server", site plan says "Stack" --> | [Deploying llm-d on managed Kubernetes](https://opendatahub-io.github.io/rhaii-on-xks/deploying-llm-d-on-managed-kubernetes/) |
| Azure (AKS) | Red Hat AI Inference | [Deploying llm-d on managed Kubernetes](https://opendatahub-io.github.io/rhaii-on-xks/deploying-llm-d-on-managed-kubernetes/) |
| Other Kubernetes* | Red Hat AI Inference | [Deploying llm-d on managed Kubernetes](https://opendatahub-io.github.io/rhaii-on-xks/deploying-llm-d-on-managed-kubernetes/) |
| Bare Linux* | Red Hat AI Inference | [Getting started](https://docs.redhat.com/en/documentation/red_hat_ai_inference_server/3.4) <!-- CONFIRM: correct link for standalone container --> |

_\* Tested on specific distributions; expected to work on any compatible Kubernetes or Linux environment._

The install docs walk you through operators, dependencies, and platform-specific configuration. Once installed, every model recipe on this site works the same regardless of which environment you chose.

## Preflight checklist

After install, verify your cluster is ready before deploying a model. These checks catch the issues that cause silent failures or stuck pods.

### Cluster access

| Check | How to verify | Pass |
|-------|--------------|------|
| Cluster admin access | `oc whoami` or `kubectl auth can-i '*' '*'` | Returns your user with admin privileges |
| OpenShift version (OCP only) | `oc version` | 4.19+ <!-- CONFIRM: minimum version --> |
| No conflicting Service Mesh (OCP only) | `oc get csv -A \| grep servicemesh` | No pre-existing Service Mesh operator |

### GPU access

| Check | How to verify | Pass |
|-------|--------------|------|
| GPU nodes present | `kubectl get nodes -l nvidia.com/gpu.present=true` | At least one node labeled |
| GPUs allocatable | `kubectl describe node <gpu-node> \| grep nvidia.com/gpu` | `nvidia.com/gpu` shows expected count |
| GPU drivers working | Run `nvidia-smi` in a GPU pod | Shows GPU model, driver version, CUDA version |
| NVLink topology (multi-GPU models) | `nvidia-smi topo -m` | `NV` connections between GPUs, not `PIX` or `PHB` |

### Required CRDs

| Check | How to verify | Pass |
|-------|--------------|------|
| KServe CRDs | `kubectl get crd inferenceservices.serving.kserve.io` | CRD exists |
| llm-d CRDs (distributed only) | `kubectl get crd llminferenceservices.serving.kserve.io` | CRD exists <!-- CONFIRM: exact CRD name --> |
| Gateway CRDs (distributed only) | `kubectl get crd httproutes.gateway.networking.k8s.io` | CRD exists |

### Resource capacity

| Check | How to verify | Pass |
|-------|--------------|------|
| Sufficient CPU | Check your recipe's hardware section | Node has enough allocatable CPU |
| Sufficient RAM | Check your recipe's hardware section | Node has enough allocatable memory |
| GPU memory | `nvidia-smi --query-gpu=memory.total --format=csv` | Meets recipe's GPU memory requirement |

> If any check fails, go back to the [install docs](#where-are-you-running) for your environment. The install process configures operators, drivers, and CRDs that these checks depend on.

## High-performance networking

**When you need this:** Your recipe or feature requires high-speed communication between nodes. Two features drive this requirement:

- **P/D disaggregation:** Separating prompt processing and token generation across different node pools. See [disaggregation docs](https://llm-d.ai/docs/guides/pd-disaggregation).
- **Wide Expert Parallelism (Wide EP):** Distributing MoE model experts across multiple nodes for scale-out. <!-- CONFIRM: link to Wide EP docs -->

If your model runs on a single node (like Llama-3.1-8B on 1 GPU or Nemotron-120B on 8 GPUs within one node), skip this section.

### What's involved

High-performance networking replaces standard TCP with RDMA (Remote Direct Memory Access), which lets GPUs on different nodes communicate directly without going through the CPU. The result: inter-node latency drops from milliseconds to microseconds.

| Component | What it does | OCP guide | xKS guide |
|-----------|-------------|-----------|-----------|
| NVIDIA Network Operator | Detects and configures Mellanox NICs (ConnectX-6/7) | [Infrabric deployer](https://github.com/bbenshab/Infrabric-deployer) | <!-- CONFIRM: xKS RDMA guide --> |
| SR-IOV + NetworkAttachmentDefinitions | Creates virtual network interfaces for GPU pods | [Infrabric deployer](https://github.com/bbenshab/Infrabric-deployer) | <!-- CONFIRM --> |
| RDMA device plugin | Makes RDMA devices allocatable by Kubernetes | [Infrabric deployer](https://github.com/bbenshab/Infrabric-deployer) | <!-- CONFIRM --> |

### Validate networking

After configuring RDMA, verify it works before deploying models that depend on it.

| Check | How to verify | Pass |
|-------|--------------|------|
| NICs detected | `ibv_devices` in a test pod | Shows expected Mellanox NICs |
| Node-to-node connectivity | Ping over secondary interface between GPU nodes | All pairs reachable |
| Bandwidth | `ib_write_bw` between node pairs | >80% of NIC line rate (e.g., >160 Gbps for ConnectX-7 200G) |
| Latency | `ib_write_lat` between node pairs | <5μs (InfiniBand) or <10μs (RoCE) |

> If bandwidth or latency numbers are off, check NIC firmware, switch configuration, and flow control settings. The [Infrabric deployer](https://github.com/bbenshab/Infrabric-deployer) README covers common networking issues.

---

_Product doc links are based on 3.4 EA releases. Confirm exact URLs against GA versions. xKS RDMA automation is TBD per the deployment playbook._
