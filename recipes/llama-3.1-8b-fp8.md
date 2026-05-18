# Llama-3.1-8B-Instruct-FP8

General-purpose dense model. Chat, RAG, tool calling. The starting point for most deployments.

## Hardware

| Requirement | Minimum |
|-------------|---------|
| GPUs | 1 |
| GPU memory | 24GB+ (L4, A100, H100, L40S) |
| CPU | 4 cores |
| RAM | 32GB |

FP8 quantization (8-bit floating point) cuts the memory footprint roughly in half compared to BF16, with minimal quality loss. This model fits comfortably on a single GPU with room for KV cache, meaning more concurrent users before you need to scale out.

Make sure your environment is ready: [Platform prep](../platform-prep.md).

> **Note:** FP8 requires CUDA compute capability 8.9+ (H100, L40S, L4). A100 support: needs engineering confirmation. <!-- CONFIRM: does FP8-dynamic work on A100-80GB? -->

## Model References

| Source | Reference |
|--------|-----------|
| ModelCar | `registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5` |
| HuggingFace | `RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic` |

## Deploy Standalone

Single model, no routing or fleet orchestration. Use this when you have one model serving one application.

```yaml
# <!-- CONFIRM: exact CRD structure with engineering -->
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-3-1-8b-fp8
spec:
  predictor:
    model:
      modelFormat:
        name: vLLM
      runtime: vllm-runtime
      storageUri: "oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5"
      resources:
        requests:
          cpu: "2"
          memory: 16Gi
          nvidia.com/gpu: 1
        limits:
          cpu: "4"
          memory: 32Gi
          nvidia.com/gpu: 1
      ports:
        - containerPort: 8000
          protocol: TCP
```

## Deploy with llm-d

Fleet routing, scaling, prefix-aware caching. Use this when you need intelligent request routing, multiple replicas, or you're serving multiple models through a gateway.

```yaml
# <!-- CONFIRM: exact CRD structure with engineering -->
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceService
metadata:
  name: llama-3-1-8b-fp8
spec:
  model:
    uri: "oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5"
    name: RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic
  replicas: 3
  router:
    scheduler: {}
    gateway: {}
  template:
    containers:
      - name: main
        resources:
          requests:
            cpu: "2"
            memory: 16Gi
            nvidia.com/gpu: 1
          limits:
            cpu: "4"
            memory: 32Gi
            nvidia.com/gpu: 1
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
            scheme: HTTPS
          initialDelaySeconds: 120
          periodSeconds: 30
          failureThreshold: 5
```

With llm-d, the scheduler routes requests to the replica most likely to have relevant KV cache blocks already loaded. For a model like Llama-3.1-8B with a shared system prompt, this means the first user pays the full prefill cost, but subsequent users hitting the same prompt get near-instant responses. Details: [llm-d prefix-aware routing](https://llm-d.ai/docs).

## vLLM Args

None required. Llama-3.1-8B uses standard architecture, standard tokenizer. Defaults work.

## What Good Looks Like

- Pod status: `Running`, health check passing at `/health`
- Logs show model weights fully loaded (no OOM, no restarts)
- Run [GuideLLM](https://github.com/neuralmagic/guidellm) to establish your performance baseline
- Set up [monitoring and metrics](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_and_monitoring_models/) for TTFT, ITL, and throughput <!-- CONFIRM: link to specific monitoring chapter; RHAII version TBD -->
- If your numbers feel off, go to [Performance Diagnostics](../performance-diagnostics.md)

## Gotchas

**HuggingFace token required.** Llama is a gated model. You need a valid HF token with accepted license terms, or the model download fails silently. Create the token secret before deploying:

```bash
kubectl create secret generic hf-token --from-literal=token=<your-token>
```

**OpenShift SCC.** On OpenShift, the pod needs a SecurityContextConstraint that allows GPU device access. If the pod is stuck in `Pending` or `CrashLoopBackOff` with permission errors, check the SCC binding for your namespace.

**FP8 GPU compatibility.** If the model loads but produces garbage output on older GPUs, you may be hitting a compute capability mismatch. Verify your GPU supports FP8 with `nvidia-smi --query-gpu=compute_cap --format=csv`.

---

_YAML in this recipe is based on upstream KServe/llm-d sample patterns (Qwen2.5-7B, gpt-oss-20b). Flagged for engineering confirmation: exact CRD fields, resource limits for this model, FP8 on A100 compatibility._
