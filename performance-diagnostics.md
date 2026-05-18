# Performance Diagnostics

Your model is running but something feels off. Start from what you're experiencing, not from what you think the fix might be.

## I want instant responses

Users hit enter and wait. This wait, or thinking time, is called latency to first token. This thinking time needs to be minimized for faster responses.

**Time to First Token (TTFT)** measures the gap between sending a request and seeing the first token come back. Look at P95. P95 is your worst-case user experience: the slowest request out of every 20.

**Some requests are fast, others randomly slow?** Break large prompts into smaller pieces so short requests can interleave. Add to your vLLM args:

```
--enable-chunked-prefill
```

Works on both vLLM deployments and llm-d deployments. [How chunked prefills work](https://docs.vllm.ai). <!-- CONFIRM: link to specific vLLM chunked prefill page -->

**All your requests start with the same long preamble?** Route to pods that already have that prompt loaded in GPU memory. The first user pays the full processing cost, every user after that gets it nearly free. Requires llm-d. [Configure prefix-aware routing](https://llm-d.ai/docs/guides/optimized-baseline). <!-- CONFIRM: link to specific prefix routing page -->

**First response gets slower as traffic increases?** Separate prompt processing from token generation onto different GPUs. Dedicated prompt-processing GPUs handle the heavy compute without slowing down generation. Requires llm-d. [Configure prefill/decode disaggregation](https://llm-d.ai/docs/guides/pd-disaggregation).

**Running batch and interactive traffic on the same endpoint?** Batch requests consume the same prefill resources as interactive ones. Without priority separation, a burst of batch work pushes interactive TTFT higher. Flow control lets you tag requests by priority so interactive users always go first. Requires llm-d. _Coming soon._ <!-- CONFIRM: flow control availability and config -->

**Baseline your numbers:** Run [GuideLLM](https://github.com/neuralmagic/guidellm) with a short prompt (under 512 tokens) at low concurrency. That's your best-case TTFT. Then test with your actual prompt lengths and concurrency to see where it degrades.

**Why is TTFT slow?** The model has to process your entire input prompt before generating the first output token. Longer prompts take longer. If a short request lands behind a long one, the short one waits. This is called head-of-line blocking, and it's what chunked prefills solve.

## I want faster streaming of responses

Users see the model type its response word by word. This typing speed, how fast each word arrives after the last, is called Inter-Token Latency (ITL). For a natural reading experience, each word needs to arrive in under 50ms, roughly 20 tokens per second.

Look at ITL P95, not the average. The average hides stuttering: a stream that's mostly fast but freezes every few seconds feels broken even if the average looks fine. Benchmarking tools often report TPOT (Time Per Output Token), which is the average version of this metric across the full response.

**Output feels slow on a single GPU?** Split the model across multiple GPUs on the same node so each GPU handles a portion of the math. This multiplies your effective memory bandwidth. Two GPUs means roughly twice the bandwidth, four means four times. NVLink between GPUs is required. Add to your vLLM args:

```
--tensor-parallel-size 4
```

Works on both vLLM deployments and llm-d deployments. [How tensor parallelism works](https://docs.vllm.ai). <!-- CONFIRM: link to specific vLLM TP page -->

**Already using multiple GPUs and still slow?** Use a small, fast "draft" model to predict several tokens ahead, then verify them against the full model in one batch. When the guesses are right (and they usually are for predictable text), you generate 3-4 tokens in the time it normally takes to generate one. Add to your vLLM args:

```
--speculative-model <draft-model-name>
--num-speculative-tokens 5
```

Works on both vLLM deployments and llm-d deployments. [How speculative decoding works](https://docs.vllm.ai). <!-- CONFIRM: link to specific vLLM speculative decoding page, confirm args -->

**Want a quick win without changing your setup?** Use a lower-precision quantized variant of your model. Smaller weights mean less data to read from GPU memory per token, which directly speeds up generation. Check your model's [recipe](recipes/) for available variants (FP8, NVFP4). No config change needed if you switch to a pre-quantized model.

**Baseline your numbers:** Run GuideLLM with a single concurrent user generating at least 256 tokens. That isolates ITL from queuing effects. Check both the average and P95 to see if your stream is smooth or stuttery.

**Why is ITL high?** Each token requires a full pass through the entire model. The bottleneck is usually memory bandwidth (reading the model weights from GPU memory), not compute. Bigger models mean more weights to read per token. ITL spikes happen when the scheduler interrupts your request to handle other work, or when memory pressure forces the system to swap cached data in and out.

## I want to handle more users

How many users your system serves at the same time, its concurrency, is what you're expanding here.

**Throughput**, measured in tokens per second across the whole system. Also watch GPU memory utilization: when KV cache memory fills up, new requests queue or get rejected.

**Running out of GPU memory as users increase?** Compress the KV cache entries from 16-bit to 8-bit (FP8). Each conversation uses half the GPU memory, so you fit roughly twice as many concurrent users on the same hardware. Add to your vLLM args:

```
--kv-cache-dtype fp8
```

Works on both vLLM deployments and llm-d deployments. [How KV cache quantization works](https://docs.vllm.ai). <!-- CONFIRM: link to specific vLLM KV cache quantization page -->

**Adding more requests doesn't increase total output?** The CPU-side scheduling loop may be the bottleneck, not the GPU. This happens when scheduling overhead can't keep up with GPU speed. Enable async scheduling so CPU work overlaps with GPU execution. Add to your vLLM args:

```
--disable-frontend-multiprocessing=false
```

Works on both vLLM deployments and llm-d deployments. <!-- CONFIRM: correct vLLM arg for async scheduling -->

**Throughput is fine per pod but you need more total capacity?** Add replicas. On standalone, scale the deployment. On llm-d, the gateway distributes traffic automatically across replicas. [Set up autoscaling](https://llm-d.ai/docs/guides/workload-autoscaling) to scale replicas based on queue depth or latency targets. Requires llm-d for intelligent scaling.

**Processing a queue where response time doesn't matter?** Batch workloads optimize for total throughput, not per-request latency. Increase maximum batch sizes to pack more requests per GPU cycle, and use quantized model variants (FP8, NVFP4) to fit more concurrent work in the same GPU memory. If you're also serving interactive users on the same infrastructure, see the [flow control note](#i-want-instant-responses) in the TTFT section to keep batch traffic from degrading real-time responses. <!-- CONFIRM: specific vLLM args for batch optimization (max batch size, scheduling delay) -->

**Baseline your numbers:** Run GuideLLM at increasing concurrency levels (1, 10, 50, 100 users). Plot throughput vs. concurrency. The point where throughput flattens is your capacity limit on current hardware.

**Why does it degrade under load?** Every active request holds KV cache memory on the GPU. The KV cache stores the model's "working memory" of each conversation. More concurrent users means more cache, and GPU memory is finite. When it fills up, new requests wait.

## I want to serve massive documents

The maximum prompt size your deployment can handle, its context limit, is what you're pushing here.

**Maximum context length** you can serve without OOM errors. Also watch [TTFT](#i-want-instant-responses) at long context lengths, as it grows with prompt size.

**Multiple requests referencing the same long document?** Route them to the pod that already has the context cached. Only the first request pays the full processing cost. Requires llm-d. [Configure prefix-aware routing](https://llm-d.ai/docs/guides/optimized-baseline). <!-- CONFIRM: link to specific prefix routing page -->

**OOM on a single long prompt?** Spill KV cache from GPU memory to CPU RAM. CPU RAM is cheaper and larger. The tradeoff is latency: tokens that hit spilled cache generate slightly slower. Add to your vLLM args:

```
--cpu-offload-gb 32
```

Works on both vLLM deployments and llm-d deployments. [How CPU offloading works](https://docs.vllm.ai). <!-- CONFIRM: link to specific vLLM CPU offloading page -->

**Still running out of memory with CPU offloading?** Compress KV cache entries to FP8. Reduces cache memory by roughly 50%, effectively doubling the maximum context length on the same hardware. Add to your vLLM args:

```
--kv-cache-dtype fp8
```

Works on both vLLM deployments and llm-d deployments. Combines with CPU offloading. If you already enabled `--kv-cache-dtype fp8` for [concurrency](#i-want-to-handle-more-users), you're already getting this benefit.

**Context too large for a single GPU even with offloading?** Split the prompt itself across GPUs. Context parallelism (CP) divides the tokens across GPUs (tokens 1-64K on GPU A, tokens 64K-128K on GPU B). Requires multi-GPU setup and NVLink. <!-- CONFIRM: is CP available in current vLLM? what's the arg? -->

**Baseline your numbers:** Test with progressively longer prompts (4K, 16K, 64K, 128K tokens) at single-user concurrency. Find the prompt length where you first see OOM or unacceptable [TTFT](#i-want-instant-responses). That's your current ceiling.

**Why does it fail with long prompts?** A 128K-token prompt generates a KV cache that can consume tens of gigabytes of GPU memory. At some point, the cache plus the model weights exceed available GPU memory, and the request either fails or evicts other users' caches.

---

_vLLM args shown here are based on current upstream defaults. Confirm exact flags and values against your vLLM version. Upstream doc links to be pointed to specific pages._
