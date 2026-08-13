---
title: "Slurm Architecture: How slurmctld and slurmd Actually Work"
date: 2026-08-13
draft: false
tags: ["slurm", "hpc", "architecture", "cluster"]
categories: ["Engineering"]
description: "A tour of Slurm's internals: slurmctld, slurmd, the Node/Partition/Job managers, MUNGE, and how they cooperate to schedule and run a job."
showToc: true
cover:
  image: "images/Slurm_Architecture_Cover_Image.png"
  alt: "Slurm control flow: client commands, slurmctld primary/backup, and slurmd compute daemons"
  relative: true
---

Every genomics pipeline, molecular dynamics run, or docking sweep that lands on a shared HPC cluster eventually goes through the same gatekeeper: a scheduler deciding which node gets the job and when. Slurm is the scheduler behind most of that traffic, and its design is simpler than it looks once you separate the two daemons doing the actual work from the handful of commands you type at a terminal.

<!--more-->

This post is the conceptual companion to [Slurm Cluster Setup on Ubuntu 24.04]({{< ref "slurm-cluster-setup-ubuntu-24.04" >}}): that post walks through *installing* a cluster, while this one explains *what you just installed* and why each piece exists.

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [slurmctld: the Controller](#slurmctld-the-controller)
3. [Inside slurmctld: Three Managers](#inside-slurmctld-three-managers)
4. [slurmd: the Worker](#slurmd-the-worker)
5. [MUNGE: Trust Between Nodes](#munge-trust-between-nodes)
6. [slurmdbd and Accounting](#slurmdbd-and-accounting)
7. [The Command-Line Surface](#the-command-line-surface)
8. [Metascheduler and Globus (Optional Layer)](#metascheduler-and-globus-optional-layer)
9. [Key Takeaways](#key-takeaways)
10. [Further Reading](#further-reading)

---

## The Big Picture

Strip away every config flag and Slurm reduces to four moving parts:

- **`slurmctld`**: the controller. One process, one brain, running on the head node. It knows every node's state, every partition's rules, and every job's place in the queue.
- **`slurmd`**: the worker. One process per compute node. It launches the job steps `slurmctld` assigns it and reports back.
- **`slurmdbd`**: the accountant. Records what ran, for how long, and on whose account (backed by a MySQL/MariaDB database).
- **MUNGE**: the trust layer. A shared-secret authentication scheme so nodes can verify that a request from "the controller" is actually from the controller.

Everything you type (`sinfo`, `squeue`, `srun`, `scancel`, `scontrol`) is a thin client that opens a connection to `slurmctld`, asks a question or issues a command, and prints the answer. None of these commands talk to compute nodes directly; they always go through the controller first.

![Slurm control flow: client commands, slurmctld primary/backup, and slurmd compute daemons](../../images/slurm-architecture-control-flow.svg)

The diagram also shows the piece most tutorials skip: **failover**. A second `slurmctld` process can run in backup mode on a different host, watching the same state directory. If the primary dies, the backup promotes itself and the cluster keeps scheduling; compute nodes and running jobs are untouched, because `slurmd` never depended on which controller instance was talking to it.

---

## slurmctld: the Controller

`slurmctld` is a single logical scheduler, even though it can run as a primary/backup pair for high availability. Its job is narrow but critical:

- Track the state of every node (`idle`, `allocated`, `down`, `drain`, …).
- Enforce partition rules: which nodes belong to which queue, time limits, access controls.
- Decide which pending job gets which nodes next, based on the configured scheduler (`sched/backfill` by default).
- Persist enough state to `StateSaveLocation` that a restart (or a failover to the backup) doesn't lose the queue.

If `slurmctld` goes down and there's no backup configured, the cluster freezes: no new jobs schedule, though jobs already running on compute nodes keep running, since `slurmd` doesn't need a live controller to finish work it was already told to do.

---

## Inside slurmctld: Three Managers

Internally, `slurmctld` splits its responsibilities across three logical managers. They're not separate processes, just a useful mental model (and roughly how the source code is organized):

- **Node Manager**: owns the live inventory of nodes, tracking which ones exist, their CPU/GPU/memory counts, and their current state. This is what `sinfo` is really reading.
- **Partition Manager**: groups nodes into partitions (queues) and enforces the limits attached to each one, such as `MaxTime`, `Default`, and which accounts or QOS can use it.
- **Job Manager**: owns the job queue itself, including admission, priority, and dispatch. When a job is ready to run, the Job Manager is what tells the right `slurmd` instances to start it.

![Slurm slurmctld and slurmd internal components: Node Manager, Partition Manager, Job Manager, and the matching slurmd modules](../../images/slurm-architecture-components.svg)

On the `slurmd` side, the matching pieces are:

- **Machine Status**: reports the node's real hardware state back to the Node Manager (this is the heartbeat that keeps `sinfo` accurate).
- **Job Status** and **Job Control**: track and manage the jobs currently running on that node, on behalf of the Job Manager.
- **Remote Execution**: forks and execs the job step. This is the component doing the work `srun`/`sbatch` asked for.
- **Stream Copy**: pipes the job's stdout/stderr back to the submitting client, which is why `srun --pty bash` feels like a normal interactive shell even though the process is running on another machine entirely.

---

## slurmd: the Worker

`slurmd` is deliberately dumb by comparison to `slurmctld`; it doesn't make scheduling decisions. It waits for instructions from the controller, then:

1. Sets up the job's environment (cgroups, task affinity, environment variables).
2. Launches the requested task via **Remote Execution**.
3. Monitors it via **Job Status** / **Job Control** until it exits.
4. Streams output back via **Stream Copy**.
5. Reports the node's health continuously via **Machine Status**, independent of whether a job is running.

This split is why a compute node can keep running an already-dispatched job even if it temporarily loses contact with the controller; `slurmd` only needs `slurmctld` to *receive* new work, not to *execute* work it already has.

---

## MUNGE: Trust Between Nodes

Slurm's default authentication (`AuthType=auth/munge`) relies on every node sharing the same secret key at `/etc/munge/munge.key`. When `slurmctld` sends a message to `slurmd`, or a client runs `srun`, MUNGE signs it, and the receiving daemon verifies the signature before acting on it. No shared key, no trust, no cluster. The [setup guide]({{< ref "slurm-cluster-setup-ubuntu-24.04" >}}) covers generating and distributing this key step by step; architecturally, just remember that MUNGE sits underneath every arrow in the diagrams above.

---

## slurmdbd and Accounting

`slurmdbd` isn't in the scheduling path at all: jobs run whether or not it's healthy. Its role is historical record-keeping: `slurmctld` reports job start/end/exit-code events to `slurmdbd`, which writes them into a MySQL/MariaDB database (`slurm_acct_db` in the setup guide). That's what backs `sacct`, fair-share priority calculations, and per-account usage reporting, which becomes useful the moment more than one team shares a cluster, and in a pharma or life-science computing environment that's almost always.

---

## The Command-Line Surface

Every Slurm command is a client to `slurmctld`; the difference is which manager it's really talking to.

| Command | Talks to | Purpose |
|---|---|---|
| `sinfo` | Node Manager | List partitions and node states |
| `squeue` | Job Manager | List pending/running jobs |
| `srun` / `sbatch` | Job Manager → Remote Execution | Submit and launch a job |
| `scancel` | Job Manager | Cancel a running or pending job |
| `scontrol` | Node / Partition / Job Manager | Low-level inspect and modify (nodes, partitions, jobs) |
| `sacct` | slurmdbd | Query historical job accounting data |

---

## Metascheduler and Globus (Optional Layer)

Slurm can sit underneath a metascheduler, such as Globus or a similar grid/federation layer, for sites that span multiple clusters or need to broker jobs across administrative domains. That layer talks to the Partition Manager and Job Manager the same way a human running `scontrol`/`srun` would; it's an optional client, not a required part of Slurm's core loop. Most single-site clusters, including the two-node setup in the companion post, never need it.

---

## Key Takeaways

- **Two daemons do all the work.** `slurmctld` decides, `slurmd` executes. Everything else (accounting, auth, metascheduling) is support infrastructure around that core loop.
- **Compute nodes survive controller outages.** A running job doesn't need a live `slurmctld` to finish; it only needs to have been dispatched in the first place.
- **The three-manager model inside slurmctld maps directly to the three questions a scheduler has to answer**: what nodes exist (Node Manager), what rules govern them (Partition Manager), and what runs where (Job Manager).
- **MUNGE is the unstated dependency.** Every arrow in these diagrams assumes a valid, matching key; if nodes can't authenticate, none of the rest matters.

---

## Further Reading

- [Slurm Cluster Setup on Ubuntu 24.04]({{< ref "slurm-cluster-setup-ubuntu-24.04" >}}): the hands-on install guide this post explains the theory behind.
- [SchedMD Slurm Documentation](https://slurm.schedmd.com/documentation.html): official reference for every daemon and config option mentioned here.
- [Slurm Overview (SchedMD)](https://slurm.schedmd.com/overview.html): the original architecture diagrams this post's diagrams are redrawn from.
