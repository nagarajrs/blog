---
title: "Slurm Cluster Setup on Ubuntu 24.04"
date: 2026-05-07
draft: false
tags: ["slurm", "hpc", "linux", "ubuntu", "cluster"]
categories: ["Engineering"]
description: "Step-by-step guide to setting up a two-node Slurm cluster (head node + compute node) on Ubuntu 24.04, covering MUNGE, slurmdbd, MariaDB, and job submission."
showToc: true
---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Set Hostnames on Both Nodes](#step-1--set-hostnames-on-both-nodes)
4. [Step 2 — Configure /etc/hosts on Both Nodes](#step-2--configure-etchosts-on-both-nodes)
5. [Step 3 — Update and Upgrade Both Nodes](#step-3--update-and-upgrade-both-nodes)
6. [Step 4 — Install and Configure MUNGE (Both Nodes)](#step-4--install-and-configure-munge-both-nodes)
7. [Step 5 — Copy the MUNGE Key to the Compute Node](#step-5--copy-the-munge-key-to-the-compute-node)
8. [Step 6 — Verify MUNGE is Working](#step-6--verify-munge-is-working)
9. [Step 7 — Install Slurm Controller on Head Node](#step-7--install-slurm-controller-on-head-node)
10. [Step 8 — Install and Configure MariaDB on Head Node](#step-8--install-and-configure-mariadb-on-head-node)
11. [Step 9 — Install and Configure slurmdbd on Head Node](#step-9--install-and-configure-slurmdbd-on-head-node)
12. [Step 10 — Configure slurm.conf on Head Node](#step-10--configure-slurmconf-on-head-node)
13. [Step 11 — Set Correct File Permissions on Head Node](#step-11--set-correct-file-permissions-on-head-node)
14. [Step 12 — Start Slurm Services on Head Node](#step-12--start-slurm-services-on-head-node)
15. [Step 13 — Install and Configure Slurm on Compute Node](#step-13--install-and-configure-slurm-on-compute-node)
16. [Step 14 — Test the Cluster](#step-14--test-the-cluster)
17. [Troubleshooting](#troubleshooting)

---

## Overview

A Slurm cluster consists of:

| Role | Hostname | Services Running |
|---|---|---|
| **Head Node** (controller) | `headnode` | `slurmctld`, `slurmdbd`, `mariadb`, `munge` |
| **Compute Node** (worker) | `computenode` | `slurmd`, `munge` |

- **slurmctld** — the main Slurm controller daemon. Schedules and manages jobs.
- **slurmd** — runs on each compute node. Executes the actual jobs.
- **slurmdbd** — Slurm's database daemon. Records job accounting history.
- **MariaDB** — the database backend for slurmdbd.
- **MUNGE** — a shared secret authentication system. All nodes must have the same MUNGE key to trust each other.

> **Important:** Every command in this guide must be run as `root` unless stated otherwise. You can switch to root with `sudo -i` or prefix commands with `sudo`.

---

## Prerequisites

Before you begin, make sure you have:

- Two machines (physical, virtual, or cloud instances) running **Ubuntu 24.04**.
- Both machines can reach each other over the network (test with `ping`).
- SSH access to both machines. The head node should be able to SSH into the compute node without a password (using an SSH key pair).

### Set Up Passwordless SSH from Head Node to Compute Node

If you have not already set this up, do the following **on the head node**:

```bash
# Generate an SSH key pair if you don't have one yet
ssh-keygen -t rsa -b 4096

# Copy your public key to the compute node so you can SSH without a password
ssh-copy-id ubuntu@computenode
```

If you are using a `.pem` key file (common on AWS/cloud), copy it to the compute node for use:

```bash
scp -i nagaraj.pem nagaraj.pem ubuntu@computenode:~
```

---

## Step 1 — Set Hostnames on Both Nodes

A hostname is the name your machine uses to identify itself on the network. Setting meaningful hostnames makes it much easier to manage a cluster.

### On the Head Node

```bash
hostnamectl set-hostname headnode
```

### On the Compute Node

```bash
hostnamectl set-hostname computenode
```

Verify the hostname was set correctly:

```bash
hostname
```

> You may need to open a new terminal session for the prompt to reflect the new hostname.

---

## Step 2 — Configure /etc/hosts on Both Nodes

The `/etc/hosts` file is a simple lookup table that maps hostnames to IP addresses. We need both nodes to know each other's IP addresses by name, so Slurm can communicate between them.

Run this **on both nodes**:

```bash
vi /etc/hosts
```

Add the following lines at the bottom of the file. Replace the example IP addresses with your actual node IPs:

```
192.168.1.10   headnode
192.168.1.20   computenode
```

> **Tip:** To find a node's IP address, run `ip addr show` or `hostname -I` on that machine.

Save and close the file. Then verify that each node can reach the other by name:

```bash
# From the head node, ping the compute node
ping computenode

# From the compute node, ping the head node
ping headnode
```

If both pings work, your name resolution is configured correctly.

---

## Step 3 — Update and Upgrade Both Nodes

Always start with a fully updated system to avoid dependency issues.

Run this **on both nodes**:

```bash
apt update
apt upgrade -y
```

This may take a few minutes. After upgrading, a reboot is recommended:

```bash
reboot
```

---

## Step 4 — Install and Configure MUNGE (Both Nodes)

MUNGE is the authentication system that Slurm uses. Every node in your cluster must have MUNGE installed and must share the **exact same secret key** (`munge.key`). If the keys don't match, nodes will refuse to communicate with each other.

### Install MUNGE on Both Nodes

Run this **on both nodes**:

```bash
apt install munge -y
```

When MUNGE is installed on the head node, it automatically generates a unique `munge.key` at `/etc/munge/munge.key`. This key must be copied to the compute node in the next step.

### Start and Enable MUNGE on the Head Node

```bash
systemctl enable munge
systemctl start munge
systemctl status munge
```

You should see `active (running)` in the status output.

---

## Step 5 — Copy the MUNGE Key to the Compute Node

The MUNGE key only needs to be generated once (it was auto-generated on the head node). You must copy it from the head node to the compute node so they share the same key.

### On the Head Node — Copy the Key

```bash
scp -i /home/ubuntu/.ssh/id_rsa /etc/munge/munge.key ubuntu@computenode:~/
```

This copies the key to the home directory (`~/`) of the `ubuntu` user on the compute node.

### Verify the Key Was Copied Correctly

Check the MD5 checksum of the key on the **head node**:

```bash
md5sum /etc/munge/munge.key
```

Note the checksum output. Then on the **compute node**:

```bash
md5sum ~/munge.key
```

Both checksums must be **identical**. If they differ, copy the key again.

### On the Compute Node — Install the Key

Now move the key into the correct location and set the proper ownership:

```bash
# Set the correct owner before moving
chown munge:munge ~/munge.key

# Copy the key into the MUNGE directory
cp ~/munge.key /etc/munge/

# Verify the file is in place
ls -l /etc/munge/
```

The key file should be owned by `munge:munge`. If it is not, fix it:

```bash
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
```

### Start and Enable MUNGE on the Compute Node

```bash
systemctl enable munge
systemctl restart munge
systemctl status munge
```

---

## Step 6 — Verify MUNGE is Working

From the **head node**, test that MUNGE authentication works between both nodes:

```bash
munge -n -t 10 | ssh computenode unmunge
```

A successful response looks like:

```
STATUS:          Success (0)
ENCODE_HOST:     headnode
...
```

If you see `Success`, MUNGE is correctly set up and the two nodes trust each other.

---

## Step 7 — Install Slurm Controller on Head Node

The Slurm controller daemon (`slurmctld`) is the brain of the cluster. It accepts job submissions, schedules them, and dispatches them to compute nodes.

Run this **on the head node**:

```bash
apt install slurmctld -y
```

> Do **not** start slurmctld yet — we need to configure `slurm.conf` first, which we will do in Step 10.

---

## Step 8 — Install and Configure MariaDB on Head Node

Slurm uses a database to record job history and accounting information. We use **MariaDB** (a MySQL-compatible database) as the backend.

### Install MariaDB

Run this **on the head node**:

```bash
apt install mariadb-server mariadb-client -y
```

Check that it is running:

```bash
systemctl status mariadb
```

### Secure the MariaDB Installation

Run the security script to set a root password and remove insecure defaults:

```bash
mariadb-secure-installation
```

Follow the prompts:
- **Switch to unix_socket authentication?** — Press `n` then Enter
- **Change the root password?** — Press `y`, then set a strong password (e.g., `12345` for a dev cluster — use something stronger in production)
- **Remove anonymous users?** — Press `y`
- **Disallow root login remotely?** — Press `y`
- **Remove test database?** — Press `y`
- **Reload privilege tables?** — Press `y`

### Create the Slurm Database and User

Log into MariaDB as root:

```bash
mariadb -u root
```

Inside the MariaDB prompt, run the following SQL commands:

```sql
CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY '12345';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> Replace `'12345'` with a secure password of your choice. Make sure this password matches the `StoragePass` value you will set in `slurmdbd.conf` in the next step.

### (Optional) Verify the Slurm User Can Log In

```bash
mariadb -u slurm -p
```

Enter the password when prompted. If you reach the MariaDB prompt without errors, the user was created successfully. Type `EXIT;` to leave.

---

## Step 9 — Install and Configure slurmdbd on Head Node

`slurmdbd` is Slurm's database daemon — it sits between Slurm and MariaDB, handling all accounting data.

### Install slurmdbd

```bash
apt install slurmdbd -y
```

### Configure slurmdbd.conf

Edit the configuration file:

```bash
vi /etc/slurm/slurmdbd.conf
```

Replace the contents with the following (adjust `StoragePass` if you used a different password):

```ini
AuthType=auth/munge
DbdHost=localhost
DebugLevel=info
StorageHost=localhost
StorageLoc=slurm_acct_db
StoragePass=12345
StorageType=accounting_storage/mysql
StorageUser=slurm
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurmdbd.pid
SlurmUser=slurm
```

Key settings explained:
- `AuthType=auth/munge` — use MUNGE for authentication (required).
- `DbdHost=localhost` — slurmdbd runs on the same machine as the database.
- `StorageLoc=slurm_acct_db` — the name of the database we created.
- `StorageUser=slurm` — the database user we created.
- `StoragePass=12345` — the database password (must match what you set in MariaDB).

---

## Step 10 — Configure slurm.conf on Head Node

`slurm.conf` is the main configuration file for the entire cluster. It must be **identical on every node** (head node and all compute nodes).

### Edit slurm.conf on the Head Node

```bash
vi /etc/slurm/slurm.conf
```

Use the following configuration:

```ini
# Cluster name — can be anything you like
ClusterName=slurm-dev

# The hostname of the machine running slurmctld
SlurmctldHost=headnode

MpiDefault=none
ProctrackType=proctrack/cgroup
ReturnToService=1
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
SlurmdSpoolDir=/var/lib/slurm/slurmd
SlurmUser=slurm
StateSaveLocation=/var/lib/slurm/slurmctld
SwitchType=switch/none
TaskPlugin=task/affinity,task/cgroup

# SCHEDULING
SchedulerType=sched/backfill
SelectType=select/cons_tres

# LOGGING AND ACCOUNTING
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=headnode
JobAcctGatherType=jobacct_gather/cgroup
JobCompType=jobcomp/filetxt
JobCompLoc=/var/log/slurm/job_completions
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

# COMPUTE NODES
# Update CPUs to match the actual number of CPU cores on your compute node
NodeName=computenode CPUs=2 State=UNKNOWN

# PARTITION (queue)
PartitionName=dev Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

Key settings explained:
- `ClusterName` — a label for your cluster, used in job accounting.
- `SlurmctldHost=headnode` — tells every node where to find the controller.
- `AccountingStorageType=accounting_storage/slurmdbd` — enables job accounting via slurmdbd.
- `NodeName=computenode CPUs=2 State=UNKNOWN` — defines the compute node. Change `CPUs=2` to match your compute node's actual CPU count (check with `nproc` on the compute node).
- `PartitionName=dev` — defines a job queue called "dev" that includes all nodes.

> **Important:** The `CPUs` value must match the actual number of CPUs on the compute node. Run `nproc` on the compute node to check.

### Copy slurm.conf to the Compute Node

The exact same `slurm.conf` must be on every node in the cluster. Copy it now:

```bash
scp /etc/slurm/slurm.conf ubuntu@computenode:/tmp/slurm.conf
```

On the **compute node**, move it into place:

```bash
mv /tmp/slurm.conf /etc/slurm/slurm.conf
```

---

## Step 11 — Set Correct File Permissions on Head Node

Slurm is strict about file ownership. The configuration files must be owned by the `slurm` user and have the right permissions, or the daemons will refuse to start.

Run this **on the head node**:

```bash
cd /etc/slurm

# slurmdbd.conf contains a database password — restrict it to root/slurm only
chmod 600 slurmdbd.conf
chown slurm:slurm slurmdbd.conf

# slurm.conf should also be owned by slurm
chown slurm:slurm slurm.conf

# Verify ownership
ls -l /etc/slurm/
```

You should see output like:

```
-rw------- 1 slurm slurm  ... slurmdbd.conf
-rw-r--r-- 1 slurm slurm  ... slurm.conf
```

Also ensure the Slurm log and state directories exist and are owned by `slurm`:

```bash
mkdir -p /var/log/slurm
mkdir -p /var/lib/slurm/slurmctld
mkdir -p /var/lib/slurm/slurmd
chown -R slurm:slurm /var/log/slurm
chown -R slurm:slurm /var/lib/slurm
```

---

## Step 12 — Start Slurm Services on Head Node

Now start all three services in the correct order: database first, then accounting daemon, then the controller.

```bash
# 1. Make sure MariaDB is running
systemctl enable mariadb
systemctl restart mariadb

# 2. Start the Slurm database daemon
systemctl enable slurmdbd
systemctl restart slurmdbd
systemctl status slurmdbd

# 3. Start the Slurm controller
systemctl enable slurmctld
systemctl restart slurmctld
systemctl status slurmctld
```

For each service, look for `active (running)` in the status output.

If `slurmctld` or `slurmdbd` fails to start, check the log files for error messages:

```bash
tail -50 /var/log/slurm/slurmctld.log
tail -50 /var/log/slurm/slurmdbd.log
```

### Quick Sanity Check

Verify Slurm can see the cluster (even before the compute node is running):

```bash
sinfo
squeue
```

`sinfo` will show your partition and nodes. The compute node will likely show as `down` or `unknown` — that is expected until we start `slurmd` on the compute node.

---

## Step 13 — Install and Configure Slurm on Compute Node

Now switch to the **compute node** to install the Slurm worker daemon.

### Install slurmd

```bash
apt install slurmd -y
```

### Verify slurm.conf Is in Place

The `slurm.conf` you copied from the head node should already be at `/etc/slurm/slurm.conf`. Check:

```bash
ls /etc/slurm/
cat /etc/slurm/slurm.conf
```

It should match exactly what is on the head node.

### Create Required Directories

```bash
mkdir -p /var/log/slurm
mkdir -p /var/lib/slurm/slurmd
chown -R slurm:slurm /var/log/slurm
chown -R slurm:slurm /var/lib/slurm
```

### Start slurmd

```bash
systemctl enable slurmd
systemctl restart slurmd
systemctl status slurmd
```

You should see `active (running)`. If it fails, check the log:

```bash
tail -50 /var/log/slurm/slurmd.log
```

---

## Step 14 — Test the Cluster

Go back to the **head node** to verify everything is working.

### Check Cluster Status

```bash
sinfo
```

Expected output (the compute node should now show `idle`):

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
dev*         up   infinite      1   idle computenode
```

If the node shows `down` instead of `idle`, you may need to resume it:

```bash
scontrol update NodeName=computenode State=resume
```

### Run an Interactive Job

Test that you can run a command on the compute node:

```bash
srun --pty whoami
srun --pty hostname
```

The output should show that the command ran (you'll see `root` or your username and `computenode` as the hostname).

### Submit a Batch Job

Create a test job script:

```bash
vi ~/test.sh
```

Add the following content:

```bash
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=/root/test_%j.out
#SBATCH --ntasks=1

echo "Hello from Slurm!"
echo "Running on: $(hostname)"
date
```

Submit the job:

```bash
sbatch ~/test.sh
```

Monitor the job queue:

```bash
squeue
```

Once the job finishes (disappears from `squeue`), check the output file:

```bash
cat ~/test_*.out
```

You should see output similar to:

```
Hello from Slurm!
Running on: computenode
Wed May  7 12:00:00 UTC 2026
```

**Congratulations — your Slurm cluster is working!**

---

## Troubleshooting

### MUNGE authentication fails

```bash
# On head node: test munge authentication to compute node
munge -n -t 10 | ssh computenode unmunge
```

If this fails:
- Confirm `/etc/munge/munge.key` exists on both nodes and has the same MD5 checksum (`md5sum /etc/munge/munge.key`).
- Confirm `munge.key` is owned by `munge:munge` with permissions `400` on both nodes.
- Restart munge on both nodes: `systemctl restart munge`.

### slurmctld fails to start

```bash
tail -100 /var/log/slurm/slurmctld.log
```

Common causes:
- **slurmdbd is not running** — start it first: `systemctl restart slurmdbd`.
- **Wrong file ownership** — re-run the `chown slurm:slurm` commands from Step 11.
- **CPUs mismatch** — the `CPUs=` value in `slurm.conf` does not match the compute node. Run `nproc` on the compute node and update `slurm.conf`.

### slurmdbd fails to start

```bash
tail -100 /var/log/slurm/slurmdbd.log
```

Common causes:
- **Wrong database password** — check that `StoragePass` in `slurmdbd.conf` matches the password you set in MariaDB.
- **Database does not exist** — log into MariaDB (`mariadb -u root`) and verify the `slurm_acct_db` database and `slurm` user exist.
- **File permissions** — `slurmdbd.conf` must be `chmod 600` and owned by `slurm:slurm`.

### Compute node shows as `down` in sinfo

```bash
# On the head node, force the node back online
scontrol update NodeName=computenode State=resume
```

Also check that `slurmd` is running on the compute node:

```bash
systemctl status slurmd
```

### Jobs stay pending (PD) in squeue

```bash
# Check why the job is not being scheduled
scontrol show job <JOB_ID>
```

Look for the `Reason` field. Common reasons:
- `Resources` — nodes are busy or down.
- `ReqNodeNotAvail` — the requested node is unavailable.
- `InvalidQOS` — accounting is not fully set up; restart slurmdbd.

### Check all Slurm ports are listening

```bash
apt install net-tools -y
netstat -tupln | grep slurm
```

You should see `slurmctld` on port `6817` and `slurmdbd` on port `6819`.

---

## Quick Reference

| Command | What it does |
|---|---|
| `sinfo` | Show cluster partitions and node status |
| `squeue` | Show the job queue |
| `sbatch script.sh` | Submit a batch job |
| `srun --pty bash` | Start an interactive session on a compute node |
| `scancel <job_id>` | Cancel a job |
| `scontrol show node computenode` | Show detailed info about a node |
| `scontrol show job <id>` | Show detailed info about a job |
| `sacct` | Show job accounting history |
