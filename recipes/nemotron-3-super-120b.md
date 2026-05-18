# Nemotron-3-Super-120B-A12B

High-quality reasoning model for chat, RAG, long-context RAG (10K+ tokens), code fixing, and tool calling. Gets the quality of a 120B-parameter model at the compute cost of a 12B.

## Hardware

| Requirement | Minimum |
|-------------|---------|
| GPUs | 8x H200 or 8x B200 |
| GPU interconnect | NVLink (required) |
| CPU | 8 cores per pod |
| RAM | 64GB+ per pod |

This is a Mixture-of-Experts (MoE) model: 120B total parameters, but only 12B activate per token. The full 120B of weights must still fit in GPU memory (that's why 8 GPUs), but only 12B worth of compute runs per token. The practical result: quality comparable to a large dense model, at a fraction of the per-token cost.

The expert routing sends activations between GPUs on every token. NVLink provides the bandwidth this requires. Without it, communication falls back to PCIe and latency degrades significantly.

Make sure your environment is ready: [Platform prep](../platform-prep.md). For multi-node deployments with Wide EP, see [high-performance networking](../platform-prep.md#high-performance-networking).

Verify your topology:

```bash
nvidia-smi topo -m
```

> **Note:** 8xH200/B200 is the tested configuration for BF16. Whether FP8/NVFP4 variants fit on 8xA100-80GB is unverified. <!-- CONFIRM: A100-80GB compatibility for FP8/NVFP4 -->

> **Note:** Parallelism strategy (TP=8 vs Expert Parallelism) needs confirmation from engineering. <!-- CONFIRM: what does vLLM default to for this model? -->

## Model References

Three quantization variants, all with shipped ModelCars:

| Variant | Quantization | Tradeoff | ModelCar |
|---------|-------------|----------|----------|
| BF16 | None | Best quality, highest memory | `registry.redhat.io/rhai/modelcar-nvidia-nemotron-3-super-120b-a12b-bf16:3.0` |
| FP8 | 8-bit float | Minimal quality loss, ~50% memory savings | `registry.redhat.io/rhai/modelcar-nvidia-nemotron-3-super-120b-a12b-fp8:3.0` |
| NVFP4 | 4-bit (NVIDIA) | Some quality loss, ~75% memory savings | `registry.redhat.io/rhai/modelcar-nvidia-nemotron-3-super-120b-a12b-nvfp4:3.0` |

FP8 and NVFP4 don't necessarily mean fewer GPUs for this model (the MoE architecture still wants 8 GPUs for expert routing). The win is headroom: less weight memory means more KV cache space, which means more concurrent users on the same hardware.

**HuggingFace:**
- `RedHatAI/NVIDIA-Nemotron-3-Super-120B-A12B-BF16`
- `RedHatAI/NVIDIA-Nemotron-3-Super-120B-A12B-FP8`
- `RedHatAI/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`

## Deploy Standalone

Single model, no routing. Replace the ModelCar reference with your chosen variant.

```yaml
# <!-- CONFIRM: exact CRD structure with engineering -->
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: nemotron-3-super-120b
spec:
  predictor:
    model:
      modelFormat:
        name: vLLM
      runtime: vllm-runtime
      storageUri: "oci://registry.redhat.io/rhai/modelcar-nvidia-nemotron-3-super-120b-a12b-fp8:3.0"
      args:
        - "--tokenizer_mode=mistral"
        - "--config_format=mistral"
        - "--load_format=mistral"
        - "--enable-auto-tool-choice"
        - "--tool-call-parser=mistral"
      resources:
        requests:
          cpu: "4"
          memory: 32Gi
          nvidia.com/gpu: 8
        limits:
          cpu: "8"
          memory: 64Gi
          nvidia.com/gpu: 8
      ports:
        - containerPort: 8000
          protocol: TCP
```

## Deploy with llm-d

Fleet routing with intelligent scheduling. The gateway routes requests to replicas with relevant KV cache already loaded.

```yaml
# <!-- CONFIRM: exact CRD structure with engineering -->
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceService
metadata:
  name: nemotron-3-super-120b
spec:
  model:
    uri: "oci://registry.redhat.io/rhai/modelcar-nvidia-nemotron-3-super-120b-a12b-fp8:3.0"
    name: RedHatAI/NVIDIA-Nemotron-3-Super-120B-A12B-FP8
  replicas: 2
  router:
    scheduler: {}
    gateway: {}
  template:
    containers:
      - name: main
        args:
          - "--tokenizer_mode=mistral"
          - "--config_format=mistral"
          - "--load_format=mistral"
          - "--enable-auto-tool-choice"
          - "--tool-call-parser=mistral"
        resources:
          requests:
            cpu: "4"
            memory: 32Gi
            nvidia.com/gpu: 8
          limits:
            cpu: "8"
            memory: 64Gi
            nvidia.com/gpu: 8
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
            scheme: HTTPS
          initialDelaySeconds: 300
          periodSeconds: 30
          failureThreshold: 5
```

Note the longer `initialDelaySeconds` (300s vs 120s for Llama-8B). Loading 120B parameters across 8 GPUs takes time. If the liveness probe fires too early, Kubernetes restarts the pod before the model finishes loading.

## vLLM Args (Required)

```
--tokenizer_mode mistral
--config_format mistral
--load_format mistral
--enable-auto-tool-choice
--tool-call-parser mistral
```

All five are required. Nemotron-3-Super uses Mistral's architecture internally. Without the first three flags, the tokenizer loads incorrectly and you get garbage output or a crash. The last two enable native function calling / tool use support.

These must be set as container args in the pod spec, not as environment variables.

## What Good Looks Like

- All 8 GPUs showing the model loaded (check pod logs for weight loading completion across all ranks)
- Pod status: `Running`, health check passing at `/health`
- No OOM kills, no restarts during loading
- Run [GuideLLM](https://github.com/neuralmagic/guidellm) to establish your performance baseline
- Set up [monitoring and metrics](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_and_monitoring_models/) for TTFT, ITL, and throughput <!-- CONFIRM: link to specific monitoring chapter; RHAII version TBD -->
- If your numbers feel off, go to [Performance Diagnostics](../performance-diagnostics.md)

## Gotchas

**The Mistral args are the #1 support issue with this model.** People deploy Nemotron, get bad output or crashes, and don't realize it needs Mistral tokenizer configuration. If you're seeing garbled responses, check the args first.

**Don't skimp on the liveness probe delay.** A 120B model loading across 8 GPUs can take 3-5 minutes. The default Kubernetes liveness probe will kill the pod before it's ready. Set `initialDelaySeconds` to at least 300.

**NVLink is not optional.** Run `nvidia-smi topo -m` and verify you see `NV` connections between GPUs, not `PIX` or `PHB` (PCIe). If you're on PCIe, the model will technically run but token generation will be slow enough to be unusable for interactive workloads.

**Choosing a variant.** Start with FP8 unless you have a specific reason to need full precision. BF16 gives the best quality but uses the most memory. NVFP4 maximizes concurrent user capacity at the cost of some output quality. For most production workloads, FP8 is the right default.

---

_YAML in this recipe is based on upstream KServe/llm-d sample patterns, adapted for Nemotron-3-Super with args from the model validation spreadsheet. Flagged for engineering confirmation: exact CRD fields, resource limits, A100 compatibility for FP8/NVFP4, parallelism strategy (TP vs EP)._
