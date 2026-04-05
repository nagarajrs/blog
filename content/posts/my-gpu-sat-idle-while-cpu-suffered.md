---
title: "My GPU sat Idle While the CPU Suffered"
date: 2026-03-29
draft: false
tags: ["machine-learning", "pytorch", "cuda", "local-ai", "deep-learning"]
categories: ["Engineering"]
description: "Lessons from loading my first transformer model — what went wrong, why it went wrong, and how I fixed it."
showToc: true
---

*A practical guide to getting your AI model off the CPU and onto your GPU.*

This Saturday, I decided to build an AI-powered image editor using Qwen-Image-Edit, a large transformer-based model. What followed was a debugging session that taught me more about how PyTorch and GPU memory management work than any tutorial I had read. Here is what went wrong, why it went wrong, and how I fixed it.

<!--more-->

## The Setup

My machine is a Lenovo Legion 7 with an NVIDIA RTX GPU (8GB VRAM), 32GB RAM, and a fresh Python virtual environment. I had installed PyTorch, diffusers, and all the usual suspects. My first attempt at loading the pipeline following the instructions from Hugging Face looked like this:

```python
pipeline = QwenImageEditPipeline.from_pretrained("Qwen/Qwen-Image-Edit")
pipeline.to(torch.bfloat16)
pipeline.to("cuda")
```

If you are anything like me, you probably assumed the example code would just work. That is the thing with AI systems — even when something runs without errors, you cannot assume it is running well. Getting a model to load is one milestone. Getting it to load efficiently, on the right hardware, at the right precision — that is an entirely different problem.

## Problem #1: PyTorch Said CUDA Was Unavailable

The first thing I noticed was that `torch.cuda.is_available()` was returning `False` — even though my `nvidia-smi` clearly showed CUDA 13.1 support. This confused me at first.

The reason: there are two completely separate things called "CUDA" here. The version shown in `nvidia-smi` is the maximum CUDA toolkit your driver supports. But PyTorch ships as separate builds — a CPU-only build and CUDA-enabled builds. If you ran a simple `pip install torch` without specifying a source URL, you almost certainly got the CPU-only build.

The fix was to reinstall PyTorch explicitly with the CUDA 13.0 build:

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
```

> **Note:** Install PyTorch based on your system specs using the official [Get Started](https://pytorch.org/get-started/locally/) guide.

## Problem #2: CUDA Was Available, But the GPU Still Sat Idle

After fixing the CUDA issue, I ran the script again. CUDA was now available, but something still felt wrong. The model was taking over 7 minutes to load, GPU utilization was under 3%, and VRAM was barely touched. My monitoring tool showed RAM climbing steadily.

The root cause was in my loading pattern. Calling `.from_pretrained()` without specifying a device loads all model weights into CPU RAM first. Then calling `.to("cuda")` afterwards copies the entire model from RAM to VRAM. For a 9-shard model like Qwen-Image-Edit, this means the model exists twice in memory simultaneously during the transfer — once in RAM, once in VRAM.

> **Key insight:** The model loads shards sequentially into CPU RAM, then the entire thing is copied to VRAM. With 8GB of VRAM, a model that is close to that limit will fail, or cause significant memory pressure on both RAM and VRAM.

## The Correct Way to Load a Large Model

The solution is to tell diffusers to load each shard directly onto the GPU as it is read from disk, and to cast to a lower precision during the load itself rather than after. This requires the `accelerate` library:

```bash
pip install accelerate
```

```python
pipeline = QwenImageEditPipeline.from_pretrained(
    "Qwen/Qwen-Image-Edit",
    torch_dtype=torch.bfloat16,   # Cast during load, not after
    device_map="cuda",             # Load directly onto GPU
)
```

Two things are happening here. First, `torch_dtype=torch.bfloat16` halves the memory footprint of the model during the load itself rather than converting after the fact. Second, `device_map="cuda"` (powered by `accelerate`) causes each shard to be moved to the GPU immediately after being read from disk — there is never a full in-memory CPU copy.

If your model is too large for VRAM (which is a real concern at 8GB), swap `device_map="cuda"` for `device_map="auto"`. This tells `accelerate` to fill VRAM first, then spill any overflow into RAM, while still using the GPU for compute. Slower than pure VRAM, but far faster than pure CPU.

---

## Three Things Worth Remembering

- Your driver's CUDA version and PyTorch's CUDA support are completely independent — always verify your torch build with `torch.__version__`
- Never load a model to CPU and then move it to GPU — use `device_map` at load time
- Install `accelerate` before you need it — it unlocks proper memory management across the entire diffusers ecosystem
