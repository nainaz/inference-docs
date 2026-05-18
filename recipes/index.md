# Model Recipes

The recipes on this site are platform-agnostic: the same model configuration, vLLM args, and hardware requirements apply whether you're running on OpenShift, CoreWeave, Azure, or bare Linux. The only thing that changes per environment is the initial setup. Each recipe links to the [platform prep](../platform-prep.md) steps your model needs. <!-- TODO: add platform prep links to each recipe page once platform-prep.md is built -->

Each recipe gives you: hardware requirements, deployment YAML (standalone with vLLM and distributed inference with llm-d), vLLM configuration, what healthy looks like, and common pitfalls.

| Model | Type | GPUs | Good for |
|-------|------|------|----------|
| [Llama-3.1-8B-Instruct-FP8](llama-3.1-8b-fp8.md) | Dense, 8B | 1 | Chat, RAG, tool calling. Start here. |
| [Nemotron-3-Super-120B-A12B](nemotron-3-super-120b.md) | Mixture of Experts, 120B params, 12B active per token | 8x H200/B200 | Chat, RAG, long context, code, tool calling. High quality at lower compute cost. |
| Llama-Guard-4-12B _(coming soon)_ | Dense, 12B | 1 | Screens prompts and responses for harmful content. Runs alongside your generation model. |
