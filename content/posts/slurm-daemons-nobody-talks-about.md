---
title: "The Slurm Daemons Nobody Talks About"
date: 2026-08-13
draft: false
tags: ["slurm", "hpc", "daemons", "architecture"]
categories: ["Engineering"]
description: "slurmstepd, slurmrestd, sackd, and scrun: the four Slurm components most tutorials skip, and why each one matters."
showToc: true
cover:
  image: "images/Slurm_Daemons_Nobody_Talks_About.png"
  alt: "slurmctld and slurmd at the core, with slurmstepd, sackd, slurmrestd, and scrun as the auxiliary daemons around them"
  relative: true
---

Most Slurm writeups stop at `slurmctld`, `slurmd`, and `slurmdbd`, and call it a day. That's the right starting point, but it skips four components that quietly carry real weight in production clusters: one that does the actual work `slurmd` gets credit for, and three that exist because modern HPC has to talk to portals, cloud instances, and containers, not just a terminal.

<!--more-->

This post is a direct follow-up to [Slurm Architecture: How slurmctld and slurmd Actually Work]({{< ref "slurm-architecture-and-components" >}}). That post simplified one thing worth correcting here: `slurmd` doesn't execute your job step itself, it forks something else to do it.

---

## Table of Contents

1. [slurmstepd: What slurmd Actually Forks](#slurmstepd-what-slurmd-actually-forks)
2. [slurmrestd: Slurm Over HTTP](#slurmrestd-slurm-over-http)
3. [sackd: Authentication Without a Full slurmd](#sackd-authentication-without-a-full-slurmd)
4. [scrun: Running Containers as Slurm Jobs](#scrun-running-containers-as-slurm-jobs)
5. [Where Each One Fits](#where-each-one-fits)
6. [Key Takeaways](#key-takeaways)
7. [Further Reading](#further-reading)

---

## slurmstepd: What slurmd Actually Forks

In the previous post, "Remote Execution" was described as a piece of `slurmd` that "forks and execs the job step." That's a simplification. The component that actually does it is `slurmstepd`, and it isn't a standing daemon at all.

![slurmd forks slurmstepd per job step, which owns stdio streaming, per-task accounting, and signal delivery](../../images/slurm-daemon-slurmstepd.svg)

`slurmd` forks a new `slurmstepd` process for every job step, and that process exits the moment the step ends. While it's alive, `slurmstepd` owns three jobs:

- **Streaming**: piping the step's stdin, stdout, and stderr between the compute node and the submitting client, which is why `srun --pty bash` feels like a normal shell.
- **Accounting**: collecting per-task resource usage (CPU, memory, and so on) for whatever `slurmctld` later reports to `slurmdbd`.
- **Signals**: delivering signals sent to the step (`scancel`, timeouts, `SIGTERM` on preemption) to the actual running processes.

Nobody starts `slurmstepd` manually, and you're not supposed to. It exists purely as the boundary between `slurmd`, which is a trusted, always-running system daemon, and the user's job, which is untrusted code that needs its own process to live and die in. If you've ever wondered why a killed job step doesn't take `slurmd` down with it, this is the reason: `slurmd` was never running your code in the first place.

---

## slurmrestd: Slurm Over HTTP

Every command in the core architecture post (`sinfo`, `squeue`, `srun`, `scancel`, `scontrol`) is a CLI client talking to `slurmctld` over Slurm's own protocol. `slurmrestd` exists because not every client wants to be a CLI.

It runs in one of two modes:

- **Listen mode**: opens a TCP or UNIX socket and serves requests like any normal HTTP daemon.
- **Inetd mode**: reads from and writes to stdin/stdout, which lets it work under systemd socket activation instead of running as a permanent process.

Authentication is one of two schemes: `rest_auth/local`, which rides on `auth/munge` over a UNIX socket and requires the client's UID to match, or `rest_auth/jwt`, which works over TCP or UNIX sockets using `X-SLURM-USER-NAME` and `X-SLURM-USER-TOKEN` headers on every request.

![An HTTP client talks to slurmrestd, which authenticates via rest_auth/local or rest_auth/jwt before forwarding to slurmctld and slurmdbd](../../images/slurm-daemon-slurmrestd.svg)

The practical reason this matters: self-service portals like Open OnDemand, internal dashboards, and any tooling that shouldn't have to shell out to `squeue` and parse text all talk to Slurm through `slurmrestd`. If your scientists submit jobs through a web form instead of a terminal, this is what's answering on the other end.

---

## sackd: Authentication Without a Full slurmd

`sackd`, the Slurm Auth and Cred Kiosk Daemon, solves a narrower problem: what does a login node do if it isn't running `slurmd`?

Login nodes are where users land, write scripts, and submit jobs; they don't execute job steps, so they have no real reason to run the full `slurmd` worker. But they still need to authenticate to the cluster, using the same MUNGE-backed trust described in the architecture post. `sackd` provides that authentication on its own, without pulling in everything else `slurmd` does.

![sackd on a login node and slurmd on a compute node both authenticate to slurmctld the same way, but only slurmd executes jobs](../../images/slurm-daemon-sackd.svg)

This matters most in two setups that are common in cloud-based HPC: configless clusters, where nodes fetch `slurm.conf` from the controller instead of having it deployed statically, and cloud-bursting or ephemeral-node setups, where a login node might exist for hours and compute nodes come and go under it. Both patterns show up constantly in AWS ParallelCluster-style pharma HPC, where the login node is long-lived and the compute fleet is not.

---

## scrun: Running Containers as Slurm Jobs

`scrun` is the odd one out: it isn't a Slurm-specific concept dressed up in a new name, it's a full OCI runtime proxy, a drop-in replacement for `crun` or `runc` from Docker or Podman's point of view.

When `docker` or `podman` calls `scrun` to create, start, check the state of, kill, or delete a container, `scrun` doesn't run any of that locally. It translates each operation into a Slurm job and dispatches it to a compute node, using a `scrun.lua` script on the execution side to stage the container's data out to the node and results back. From the container tool's perspective, nothing changed. From Slurm's perspective, it's just another job.

![docker/podman calls scrun, which submits the container as a normal Slurm job through slurmctld and slurmd, with scrun.lua staging data to and from the compute node](../../images/slurm-daemon-scrun.svg)

For pharma and life-science HPC specifically, this is the piece that lets containerized, reproducible bioinformatics pipelines (the kind you want for anything that touches validated or GxP-adjacent workflows) run on the same scheduler and the same resource accounting as everything else, instead of needing a separate container orchestration layer bolted on the side.

---

## Where Each One Fits

| Daemon | Runs where | Talks to | Purpose |
|---|---|---|---|
| `slurmstepd` | Every compute node, per job step | Forked by `slurmd` | Executes the step; owns stdio, accounting, and signals |
| `slurmrestd` | Anywhere with API access to the cluster | `slurmctld` / `slurmdbd` over REST | Lets HTTP clients (portals, dashboards) act like `srun`/`squeue` |
| `sackd` | Login nodes without a full `slurmd` | `slurmctld`, via MUNGE-backed auth | Authenticates the node without running job steps |
| `scrun` | Wherever `docker`/`podman` is invoked | `slurmctld`, via a normal job submission | Proxies OCI container operations into Slurm jobs |

---

## Key Takeaways

- **`slurmd` doesn't execute your job, `slurmstepd` does.** `slurmd` forks a fresh `slurmstepd` per step; it's the real boundary between the trusted daemon and untrusted user code.
- **Not every Slurm client is a terminal.** `slurmrestd` is what portals and dashboards actually talk to, with JWT or local-socket auth standing in for a shell session.
- **Login nodes don't need the whole stack.** `sackd` gives them authentication without the job-execution machinery, which is exactly what configless and cloud-bursting clusters need.
- **Containers don't need a separate scheduler.** `scrun` makes `docker`/`podman` operations into ordinary Slurm jobs, so accounting and scheduling stay unified.

---

## Further Reading

- [Slurm Architecture: How slurmctld and slurmd Actually Work]({{< ref "slurm-architecture-and-components" >}}): the core daemons this post extends.
- [Slurm Cluster Setup on Ubuntu 24.04]({{< ref "slurm-cluster-setup-ubuntu-24.04" >}}): the hands-on install guide for the core cluster.
- [slurmstepd](https://slurm.schedmd.com/slurmstepd.html): official reference.
- [slurmrestd](https://slurm.schedmd.com/slurmrestd.html): official reference, including both auth schemes.
- [sackd](https://slurm.schedmd.com/sackd.html): official reference.
- [scrun](https://slurm.schedmd.com/scrun.html): official reference, including the OCI operation set.
