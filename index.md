> **DRAFT** — This site is a work in progress. Content is under review and not yet validated by engineering. CONFIRM tags mark items that need verification.

# AI Inference on Red Hat

Welcome to the knowledge base for high-performance AI inference. Whether you're deploying your first model or tuning a massive deployment for milliseconds of latency, start with what you want to achieve.

## I want to...

| Goal | Where to go |
|------|-------------|
| **Deploy a model** | [Pick a tested deployment](recipes/index.md) with hardware, YAML, and configuration ready to go. Each recipe links to [platform prep](platform-prep.md) for your environment (OpenShift, CoreWeave, Azure, any Kubernetes*, any Linux*). <!-- TODO: add platform prep links to each recipe page once platform-prep.md is built --> |
| **Make my deployment faster** | [Start from the symptom](performance-diagnostics.md) to find the right fix |
| **Fix something that's broken** | [Troubleshoot](troubleshooting-faq.md) common failures |
| **Serve multiple models through one endpoint** | Route each request to the most effective GPU for it using [routing and gateway](https://llm-d.ai/docs/guides/optimized-baseline) |
| **Handle more concurrent users** | Scale replicas automatically based on demand using [autoscaling](https://llm-d.ai/docs/guides/workload-autoscaling) |
| **Optimize how GPUs divide the work** | Split prompt processing and token generation across dedicated GPU pools using [disaggregation](https://llm-d.ai/docs/guides/pd-disaggregation). Requires high-performance networking between nodes. |
| **Process large batches efficiently** | Maximize throughput for offline workloads with relaxed latency SLAs. See [batch tuning](performance-diagnostics.md#i-want-to-handle-more-users) in performance diagnostics |
| **Monitor my deployment** | [Metrics and monitoring](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_and_monitoring_models/) for TTFT, ITL, throughput, and KV cache utilization. Distributed tracing: _coming soon_. <!-- CONFIRM: link to specific chapter; RHAII monitoring docs TBD --> |
| **Reduce inference costs** | _Coming soon_ |

_\* Tested on specific distributions; expected to work on any compatible Kubernetes or Linux environment._

## Upstream Projects

For topics beyond deployment and tuning, this site links to the upstream projects that cover them in depth.

| Project | What it covers | Docs |
|---------|---------------|------|
| llm-d | Distributed inference: scheduling, disaggregation, KV cache, gateway, RDMA | [llm-d.ai/docs](https://llm-d.ai/docs) |
| vLLM | Model serving engine: quantization, batching, speculative decoding | [docs.vllm.ai](https://docs.vllm.ai) |
| KServe | Model serving CRDs: LLMInferenceService, autoscaling, multi-node | [kserve.github.io](https://kserve.github.io/website) |
| Gateway API Inference Extension | Request routing: EPP, flow control, prefix-aware routing, observability | [gateway-api-inference-extension.sigs.k8s.io](https://gateway-api-inference-extension.sigs.k8s.io) |

## Deploy, tune, and troubleshoot LLM inference

- **Tested recipes** for deploying models on your hardware
- **Practical diagnostics** for fixing performance problems
- **Links to product docs** for platform installation
- **Links to upstream projects** for architecture deep dives

