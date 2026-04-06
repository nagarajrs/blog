---
title: "torchrun Rendezvous Timeout on SLURM? You're Missing srun"
date: 2026-04-05
draft: false
tags: ["slurm", "pytorch", "distributed-training", "hpc", "fine-tuning"]
categories: ["Engineering"]
description: "Why torchrun rendezvous times out on multi-node SLURM jobs, what srun actually does, and the sbatch script that finally works."
showToc: true
cover:
  image: "images/torchrun-srun-cover.png"
  alt: "torchrun rendezvous timeout on SLURM - srun is essential for launch and resource management"
  relative: true
---

We were fine-tuning [Protenix](https://github.com/bytedance/protenix) across multiple nodes on a SLURM cluster. The sbatch script looked reasonable — nodes, GPUs, torchrun all wired up. We submitted the job and waited. Then the timeout hit:

```
RuntimeError: Timed out waiting for rendezvous to be initialized.
```

No useful stack trace. No indication of which node failed. Just silence, then death. Here is what was actually wrong, why it happens, and the exact fix.

<!--more-->

## The Broken Script

```bash
#!/bin/bash
#SBATCH --job-name=protenix-finetune
#SBATCH --nodes=3
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:8
#SBATCH --partition=gpu

MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n1)
MASTER_PORT=29500

torchrun \
  --nnodes=3 \
  --nproc_per_node=8 \
  --rdzv_backend=c10d \
  --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
  train.py
```

This looks correct. It is not.

## Why the Rendezvous Times Out

### What torchrun actually expects from `--nnodes`

`--nnodes=3` does not instruct torchrun to go and launch processes on 3 nodes. It is a *promise* — torchrun is told: "3 independently launched copies of me will show up and register at the rendezvous endpoint. Wait for all of them before proceeding."

The rendezvous backend (`c10d`) acts as a barrier. Every torchrun process announces itself, and once the expected count (`--nnodes * --nproc_per_node`) is reached, training begins. If fewer processes show up than promised, the barrier never lifts — and you get a timeout.

In the broken script above, only one torchrun process ever starts: on the node that SLURM assigns as the batch host. The other two nodes are allocated but sitting completely idle — no process was ever told to run on them.

### Why SLURM doesn't "just figure it out"

This is the part that trips people up. You have `--nodes=3` and `--ntasks-per-node=1` in the sbatch header — shouldn't SLURM infer that means 3 tasks total and run them across all nodes?

No. And the reason is intentional.

SLURM treats **nodes** and **tasks** as independent dimensions. Consider these legitimate use cases:

- 1 task on 4 nodes (a single MPI process that spans nodes via shared memory)
- 4 tasks on 1 node (pure multi-process, single machine)
- 8 tasks on 4 nodes with 2 tasks per node

SLURM cannot assume which model you want. It needs to be told explicitly. The `#SBATCH` directives only **reserve** resources. They define what is available, not what gets launched. The actual launch happens when `srun` is called inside the script — and `srun` is what dispatches work to every allocated node.

Without `srun`, the shell just runs the command on the batch host. The other nodes are reserved but receive nothing to execute.

## `--ntasks` vs `--ntasks-per-node`

These two directives are related but not interchangeable:

| Directive | What it means |
|---|---|
| `--ntasks=N` | Total number of tasks across the entire job |
| `--ntasks-per-node=N` | How many tasks to place on each node |

If you set `--nodes=3` and `--ntasks-per-node=1`, SLURM knows to run 1 task per node — but only if something actually triggers the dispatch. That something is `srun`.

If you set `--ntasks=3` without `--ntasks-per-node`, SLURM will schedule 3 tasks across your nodes but may pack multiple tasks onto fewer nodes depending on availability.

For multi-node torchrun, the right combination is:

```bash
#SBATCH --nodes=3
#SBATCH --ntasks-per-node=1   # one torchrun process per node
```

And then `srun` inside the script to actually dispatch.

## The Fix

```bash
#!/bin/bash
#SBATCH --job-name=protenix-finetune
#SBATCH --nodes=3
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:8
#SBATCH --partition=gpu

MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n1)
MASTER_PORT=29500

srun torchrun \
  --nnodes=$SLURM_NNODES \
  --nproc_per_node=8 \
  --rdzv_id=$SLURM_JOB_ID \
  --rdzv_backend=c10d \
  --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
  train.py
```

One word — `srun` — is the entire fix. Here is what changes:

- `srun` dispatches the `torchrun` command to **every allocated node** (3 nodes × 1 task per node = 3 torchrun processes)
- Each torchrun process starts up, registers at the rendezvous endpoint, and waits
- Once all 3 have registered, the barrier lifts and training begins across all nodes
- `--rdzv_id=$SLURM_JOB_ID` avoids rendezvous collisions if multiple jobs are running simultaneously

## How to Verify It Is Working

After submitting the fixed job, SSH into each allocated node and run:

```bash
ps -aux | grep torchrun
```

You should see a torchrun process running on every node. Before the fix, only the master node showed the process. Once all nodes show it, and all GPUs show utilization via `nvidia-smi`, the job is genuinely distributed.

## Why This Matters Beyond the Fix

ML engineers, computational biologists, and senior scientists running fine-tuning jobs on cluster infrastructure are not always SLURM experts — and they shouldn't need to be. A job that silently wastes 2 out of 3 allocated nodes for its entire wall time is burning real compute budget and real researcher time.

On p4d or p5 instances on AWS, that idle compute is not cheap. A clear, correct sbatch template and a team that understands the `srun` requirement saves hours of confused troubleshooting and prevents expensive misuse of shared resources.

---

## The Three Things Worth Remembering

- `--nnodes` in torchrun is a promise, not an instruction — it tells torchrun how many processes to *expect*, not how many to *launch*
- `#SBATCH` directives reserve resources; `srun` is what actually dispatches work to those resources
- `--ntasks-per-node=1` shapes placement, but only `srun` triggers execution across all nodes
