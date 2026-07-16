---
title: "Inkling 975B Just Dropped — Here's How to Run, Fine-Tune, and Deploy It (Just the Commands)"
date: 2026-07-16T08:00:00+08:00
draft: false
categories: ["Artificial Intelligence"]
tags: ["AI tools", "productivity"]
description: "Thinking Machines Lab dropped Inkling 975B yesterday — and within hours, it hit 1,100+ points on Hacker News with 278 comments overflowing with one quest"
---

Thinking Machines Lab dropped Inkling 975B yesterday — and within hours, it hit 1,100+ points on Hacker News with 278 comments overflowing with one question: *how do I actually run this thing?* The official announcement covers benchmarks. TechCrunch covers the business angle. Nobody has published the step-by-step terminal commands to get Inkling running on your own hardware.

That's what this guide fixes.

Inkling is a 975B-parameter Mixture-of-Experts model with 256 experts, 41B active parameters, a 1M-token context window, and Apache-2.0 weights. It scores 97.1% on AIME 2026, 77.6% on SWE-bench Verified, and 87.2% on GPQA Diamond. And you can run it — at your desk, in the cloud, or on rented GPUs. Here are the exact commands to download, quantize, deploy, and fine-tune it.

## What Is Inkling 975B?

Inkling is an open-weights multimodal model (text, image, audio in → text out) built on a novel sparse MoE architecture. The headline numbers are 975B total parameters, but the critical number is **41B active** — only 4.2% of parameters fire per token, routed through 6 of 256 experts. That means inference costs and latency are closer to a 40B-class model while the model retains the knowledge surface of a near-trillion-parameter network.

The architecture includes several innovations that make it interesting for developers:

- **Sigmoid-based router** with auxiliary-loss-free load-balancing bias — no gradient interference from a balancing loss during training, cleaner fine-tuning behavior
- **Interleaved sliding-window + global attention** at a 5:1 ratio with only 8 KV heads — efficient for long-context work
- **Relative positional embeddings** instead of RoPE — different from nearly every other current model, with implications for fine-tuning behavior at very long contexts
- **Controllable thinking effort** — you adjust reasoning depth via system message, and the model learned to allocate tokens variably during RL

It was trained on 45 trillion tokens across text, images, audio, and video using Muon+Adam hybrid optimization on NVIDIA GB300 NVL72 hardware. Post-training used SFT on synthetic data bootstrapped from open-weights models, then large-scale RL with over 30 million rollouts.

The result is a model that's not the strongest on any single benchmark — it's intentionally a *broad foundation model* designed to be customized. As Thinking Machines puts it: "We intentionally did not make Inkling the strongest model. We made it the best model to *start from* — to fine-tune, to adapt, to own."

### What About Inkling-Small?

Inkling-Small (276B total, 12B active) is in preview. It shares the same post-training stack and performs remarkably close to the full model on reasoning and agentic benchmarks — HLE 29.6% vs 29.7%, SWE-bench Verified 77.4% vs 77.6%, MCP-Atlas 74.9% vs 74.1%. The full weights aren't released yet, but it's a strong candidate for latency-sensitive or cost-constrained workloads once they drop.

## Inkling vs Kimi K3 vs Gemma 4 vs Nemotron — Benchmark Comparison

Inkling enters a market crowded with capable open and semi-open models. Here's how it stacks up on the benchmarks that matter for real development work.

| Benchmark | Inkling 975B (41B active) | Kimi K3 (2.8T MoE) | Gemma 4 26B (Dense) | Nemotron 3 Ultra (???B) |
|---|---|---|---|---|
| **HLE (no tools)** | 29.7% | ~28% (est.) | ~12% | ~18% |
| **AIME 2026** | 97.1% | ~94% (est.) | ~42% | ~65% |
| **GPQA Diamond** | 87.2% | 85.1% | 62.4% | 72.8% |
| **SWE-bench Verified** | 77.6% | ~70% (est.) | 55.3% | 61.2% |
| **MMMU-Pro (Vision)** | 73.5% | ~68% (est.) | 66.1% | N/A |
| **Context Window** | 1M tokens | 1M tokens | 256K tokens | 128K tokens |
| **Active Params** | 41B | ~200B (est.) | 26B | ~70B (est.) |
| **License** | Apache-2.0 | Proprietary API | Gemma License | NVIDIA Open Model License |
| **API Price (Input)** | $1.87/M tok | ~$2.50/M tok | Free (self-host) | $1.20/M tok |
| **Open Weights?** | Yes (full) | No (API only) | Yes (full) | Yes (full) |
| **Self-Hostable?** | Yes (8×B300 / 16×H200) | No | Yes (1×GPU) | Yes (4×A100) |

**The takeaway:** Inkling leads on reasoning and coding benchmarks while being the only fully open-weights model in the top tier. Kimi K3 is competitive on raw benchmarks but API-only. Gemma 4 26B is the best option for single-GPU deployment. Nemotron is a solid middle ground with NVIDIA tooling. Inkling is the right choice if you need open weights, top-tier reasoning, and can provision the hardware.

## Step-by-Step: Deploy Inkling on Your Own Hardware

I'll cover three deployment paths: **vLLM** (best for production APIs), **llama.cpp / Unsloth GGUF** (best for single-node experimentation), and **SGLang** (best for long-context and high-throughput).

### Prerequisites

Regardless of path, you need serious GPU memory:

- **BF16 weights (1.76 TB):** 8× NVIDIA B300 GPUs (80GB each) or 16× H200 GPUs (141GB each)
- **NVFP4 weights (551 GB):** 4× NVIDIA B300 GPUs (W4A4) or 8× H200 GPUs (W4A16)
- **GGUF Q8_0 (~937 GB):** 12× A100 80GB or equivalent

If you don't have this hardware, skip to the ["Best For / Worst For"](#best-for-worst-for) section for cloud rental options.

### Path 1: Deploy with vLLM (Production API)

vLLM offers the best throughput for serving Inkling as an OpenAI-compatible API endpoint.

```bash
# Install vLLM (requires CUDA 12.4+)
pip install vllm

# Download the NVFP4 checkpoint (551 GB — faster path)
# This uses HuggingFace's auth; set HF_TOKEN if needed
huggingface-cli download thinkingmachines/Inkling-NVFP4 --local-dir ./inkling-nvfp4

# Start the vLLM server
python -m vllm.entrypoints.openai.api_server \
  --model ./inkling-nvfp4 \
  --tensor-parallel-size 8 \
  --dtype half \
  --max-model-len 131072 \
  --gpu-memory-utilization 0.95 \
  --port 8000
```

This starts an OpenAI-compatible server at `localhost:8000`. Send requests like:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "thinkingmachines/Inkling",
    "messages": [{"role": "user", "content": "Explain MoE routing like I am 5"}],
    "max_tokens": 1024
  }'
```

For best throughput, set `--enable-chunked-prefill true` and adjust `--max-num-batched-tokens` to your workload.

### Path 2: Deploy with Unsloth / llama.cpp GGUF (Single-Node Experimentation)

If you want maximal quantization flexibility or are running on heterogeneous hardware, use the GGUF path via Unsloth.

```bash
# Install Unsloth
pip install unsloth

# Download the Q8_0 GGUF quantization
huggingface-cli download unsloth/inkling-GGUF --local-dir ./inkling-gguf

# Run with llama.cpp
./llama-cli \
  -m ./inkling-gguf/inkling-Q8_0.gguf \
  -ngl 99 \
  -t 32 \
  -c 4096 \
  --prompt "Write a Python function to merge two sorted lists:"
```

The UD-Q2_K_XL and UD-Q3_K_XL quants drop to ~200 GB and ~300 GB respectively — viable on 4× H100 80GB. Expect 3-8 tok/s on Q2_K_XL with 4× H100s.

**Note:** The full BF16 GGUF is 1,724 GB. Strongly prefer the NVFP4 checkpoint or quantized GGUF variants unless you need full precision.

### Path 3: Deploy with SGLang (Long-Context Champion)

SGLang has the best handling of very long contexts (up to 1M tokens) — use this for document analysis, long-form RAG, or research workloads.

```bash
# Install SGLang
pip install sglang[all]

# Start server with Inkling
python -m sglang.launch_server \
  --model thinkingmachines/Inkling \
  --tp 8 \
  --context-length 131072 \
  --port 30000 \
  --host 0.0.0.0
```

SGLang uses RadixAttention for prefix caching, which dramatically reduces TTFT for shared prefixes — especially useful in chat applications and RAG pipelines where many requests share a system prompt.

### Fine-Tuning with Tinker

The easiest fine-tuning path is **Tinker** — Thinking Machines' fine-tuning platform with first-class Inkling support.

```bash
# Install Tinker CLI
pip install tinker-cli

# Authenticate
tinker login

# Create a fine-tuning dataset (JSONL format)
# Each line: {"messages": [{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]}

# Launch a LoRA fine-tuning job
tinker run \
  --base-model thinkingmachines/Inkling \
  --train-file ./training_data.jsonl \
  --method lora \
  --rank 64 \
  --learning-rate 1e-4 \
  --epochs 3 \
  --context-length 65536 \
  --output-dir ./inkling-finetuned
```

Tinker supports context lengths up to 256K tokens on Inkling. The self-fine-tuning demo from the launch shows a full loop — writing an objective, generating synthetic data, launching the training job, evaluating, and self-updating the model's own weights — completing in ~27 minutes for a constraint-tuning task.

For a self-hosted alternative, use **Unsloth** for LoRA/DoRA fine-tuning on Inkling:

```bash
pip install unsloth[colab-new]

# See the Unsloth recipe at https://unsloth.ai/docs/models/inkling
# Typical workflow: load model in 4-bit, add LoRA adapters, train, merge
```

### API Path (No Hardware)

If you don't have the GPUs, use Inkling through partner API providers. Tinker customers get a 50% launch discount for a limited time.

Available providers: [AFFILIATE: Together AI], Fireworks AI, Modal, Databricks, Baseten.

Pricing via Artificial Analysis: $1.87/M input tokens, $4.68/M output tokens, cached input at $0.374/M. At a 7:2:1 in:out:cache blend, that's $1.10/M tokens — competitive with mid-tier proprietary APIs.

```bash
# Example: Inkling via Together AI (OpenAI-compatible)
curl https://api.together.xyz/v1/chat/completions \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "thinkingmachines/Inkling",
    "messages": [{"role": "user", "content": "Explain the CAP theorem in 100 words"}],
    "max_tokens": 256
  }'
```

## Best For / Worst For

**Inkling is best for:**
- **Open-source AI companies** — Apache-2.0 license means you can commercialize derivative models, embed Inkling in products, and fine-tune without royalty
- **Research teams fine-tuning at scale** — the 41B active parameter count means fine-tuning cost is manageable while the 975B knowledge surface is enormous
- **Agentic coding workflows** — 77.6% SWE-bench Verified, 63.8% Terminal Bench 2.1, and Tinker integration make it a strong coding agent backbone
- **Multimodal applications** — native image and audio input with strong benchmarks (73.5% MMMU-Pro, 91.4% VoiceBench)
- **Long-context RAG** — 1M-token context window with SGLang's RadixAttention for efficient prefix caching

**Inkling is worst for:**
- **Single-GPU setups** — even NVFP4 requires 4× B300 or 8× H200. If you have one GPU, use Inkling via API or choose a smaller model (Gemma 4 26B, Llama 4 17B, Qwen 3 32B)
- **Latency-critical real-time apps** — the 256-expert routing adds inference overhead even with only 6 active experts. Expect ~73 tok/s median on high-end hardware
- **Simple chat applications** — the model's strength is in reasoning and agentic tasks. For basic Q&A, a dense model half the size will feel faster and cost less
- **Privacy-sensitive deployment on air-gapped hardware** — meeting the 2TB VRAM requirement in an air-gapped environment requires significant procurement effort. Consider Inkling-Small (when released) at ~550 GB if available

## Pricing

| Deployment Option | Hardware / Service | Cost (Approx.) |
|---|---|---|
| **Self-hosted BF16** | 8× B300 or 16× H200 | $100-200/hr (cloud rental) |
| **Self-hosted NVFP4** | 4× B300 or 8× H200 | $50-120/hr |
| **Together AI API** | Partner API (OpenAI-compatible) | $1.87/M in / $4.68/M out |
| **Fireworks AI API** | Partner API | Similar range |
| **Tinker Fine-Tuning** | Managed LoRA/DoRA (64K ctx) | 50% launch discount active |
| **Vast.ai / RunPod Rental** | 8× H200 141GB | ~$40-60/hr spot |
| **GGUF Q2_K_XL (4× H100)** | Self-hosted, quantized | ~$25-40/hr spot |

For most teams, the **API path** makes sense for evaluation and prototyping. Move to self-hosted once you have a verified use case and need lower per-token costs at scale. [AFFILIATE: Together AI] and [AFFILIATE: Vast.ai] both have referral programs for new users.

## FAQ

### Can I run Inkling on a single GPU?

No — not the full 975B model. The NVFP4 checkpoint still requires 4× B300 or 8× H200 GPUs. For single-GPU, use Inkling via API or wait for Inkling-Small (12B active, estimated ~60 GB quantized).

### How do I control Inkling's thinking effort?

Set the system message. Use "Think step by step before answering" for maximum reasoning (best performance on HLE, AIME, GPQA) or "Answer concisely and directly" for faster responses. The model learned variable token allocation during its RL training phase.

### What's the difference between Inkling and Inkling-Small?

Inkling is 975B total / 41B active, fully released under Apache-2.0. Inkling-Small is 276B total / 12B active, in preview — similar post-training, nearly identical scores on agentic benchmarks, but weights aren't fully released yet. Inkling-Small is ideal for latency-sensitive or cost-constrained deployments.

### Does Inkling support tool calling and function calling?

Yes. The Tinker cookbook includes recipes for tool calling, and the Python package `tml-renderer` (on PyPI) handles reliable sampling of structured outputs including tool calls. The model scored 74.1% on MCP Atlas (agentic tool use benchmark).

### What about safety and moderation?

Inkling is released without guardrails baked in, which is standard for Apache-2.0 open-weights models. The model card recommends defense-in-depth: use Llama Guard or similar for input/output classification, especially for consumer-facing applications. Inkling scores 98.6% on StrongREJECT (refusal of clearly harmful requests) but 78.0% on FORTRESS Adversarial (subtle jailbreak attempts), so moderation layers are essential for production.

## Conclusion

Inkling 975B is the most significant open-weights release of 2026 so far. It brings frontier reasoning scores (97.1% AIME, 77.6% SWE-bench) to an Apache-2.0 model for the first time, with a novel 256-expert MoE architecture that makes inference practical at 41B active parameters.

The decision is straightforward:
- **You already have multi-GPU infrastructure?** Download the NVFP4 weights, deploy with vLLM, and fine-tune via Tinker or Unsloth.
- **You want to evaluate before committing hardware?** Use the Together AI or Fireworks API with the standard OpenAI SDK — zero infrastructure changes.
- **You're building a commercial product?** The Apache-2.0 license means Inkling can be embedded, modified, and redistributed without restriction. This alone makes it the most commercially flexible option among top-tier models.

The content vacuum around this launch is temporary. Within a week, there will be dozens of tutorials, benchmarks, and deployment guides. Get ahead of the curve by running the commands in this guide right now.

*Not financial advice. Hardware pricing and API costs are estimates as of July 2026 and may vary by provider and region.*
