# SLURM Quick Reference

**Purpose:** Primer / on-call cheatsheet for helping a colleague debug a Slurm cluster  
**Audience:** Experienced Linux/DevOps, new to HPC scheduling  
**Mental model:** A batch scheduler + resource manager over a fixed node fleet. Jobs queue, the controller matches them to nodes, they run to completion, they exit.  

---

## Architecture (knowing the daemons *is* the troubleshooting)

| Daemon | Where | Role |
| --- | --- | --- |
| `slurmctld` | head node | The brain: queue, scheduling, node-state tracking |
| `slurmd` | every compute node | Launches and supervises the local job |
| `slurmdbd` | head node (optional) | Accounting DB + associations/QoS/limits. No dbd → no `sacct` history, no limit enforcement |
| `munge` / `munged` | every node | Auth layer. Shared key `/etc/munge/munge.key` must be **identical fleet-wide**, and **clocks must be in sync** |

**The munge gotcha:** clock skew of more than a few minutes, or a mismatched `munge.key`, makes the whole cluster look "down" with cryptic auth errors. If nothing works at all — check `munged` is running everywhere, keys match, clocks are synced (chrony/ntp) — *before* anything else.

---

## Command reference

### User-facing
| Command | Use |
| --- | --- |
| `sinfo` | Node + partition state (what exists, up/down/drained) |
| `squeue` | The queue — running + pending. `-u <user>` to filter |
| `sbatch script.sh` | Submit a batch job |
| `srun ...` | Launch a job step; interactive/blocking, also used inside batch scripts |
| `salloc ...` | Grab an interactive allocation (shell on allocated resources) |
| `scancel <jobid>` | Kill a job |
| `sacct -j <jobid>` | Historical/accounting view (needs slurmdbd) — your forensics tool |

### Admin / inspection (debugging power tools)
| Command | Use |
| --- | --- |
| `scontrol show job <id>` | Full request + current state for one job |
| `scontrol show node <name>` | Node detail, incl. `Reason=` it's drained/down |
| `scontrol update nodename=<n> state=resume` | Bring a drained node back |
| `scontrol show config` / `scontrol reconfigure` | View config / reload after edits |
| `scontrol ping` | Is the controller reachable |
| `sacctmgr ...` | Manage accounts, users, QoS, limits |
| `sprio` / `sshare` | Job priority breakdown / fair-share standing |

---

## Job states (`squeue` ST column)

`PD` pending · `R` running · `CG` completing · `CD` completed · `F` failed · `TO` timeout · `CA` cancelled · `OOM` out-of-memory · `NF` node fail · `S` suspended

---

## PENDING reason codes — `NODELIST(REASON)` (the diagnosis *is* the reason)

| Reason | Cause |
| --- | --- |
| `(Resources)` | Cluster busy, waiting for a slot. Legit. |
| `(Priority)` | Higher-priority jobs ahead in queue |
| `(ReqNodeNotAvail...)` | Requested nodes down/drained/reserved, **or** asked for a config that doesn't exist |
| `(QOSMax.../AssocGrp...Limit)` | Hit a QoS or account/association limit (e.g. allocation exhausted) |
| `(PartitionTimeLimit)` | Asked for more walltime than the partition allows |
| `(Dependency)` | Waiting on another job (`--dependency`) |
| `(JobArrayTaskLimit)` | Array `%N` concurrency throttle |
| `(BadConstraints)` | Feature/constraint can't be satisfied |

**Workflow:** `squeue` → read REASON → `scontrol show job <id>` (the full ask) → `sinfo` (what's available). The bug is almost always the gap between those two.

---

## Exit codes / failure forensics

`sacct -j <id> --format=JobID,State,ExitCode,DerivedExitCode,MaxRSS,Elapsed`

| Signal | Code | Meaning |
| --- | --- | --- |
| State `OUT_OF_MEMORY` / exit `137` | 128+9 SIGKILL | Blew past `--mem`. Compare `MaxRSS` to requested |
| State `TIMEOUT` / exit `143` | 128+15 SIGTERM | Hit `--time` walltime |

Then read the `.out` / `.err` files in the submit dir.

---

## sbatch script anatomy

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --partition=normal
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G               # hard limit, cgroup-enforced
#SBATCH --time=01:00:00        # walltime; exceed it -> TIMEOUT
#SBATCH --output=%x-%j.out     # %x=jobname %j=jobid %N=node
#SBATCH --gres=gpu:1           # GPUs as a generic resource
#SBATCH --array=1-10           # job array; see $SLURM_ARRAY_TASK_ID

module load whatever           # HPC software usually via environment modules
srun ./my_program
```

Everything above `srun` is the **request**; everything below is the **workload**.

Useful in-job env vars: `SLURM_JOB_ID`, `SLURM_ARRAY_TASK_ID`, `SLURM_CPUS_PER_TASK`, `SLURM_NODELIST`, `SLURM_SUBMIT_DIR`.

---

## Troubleshooting playbook

1. **Stuck PENDING** → `squeue`, read REASON (table above) → `scontrol show job <id>` vs `sinfo`.
2. **Died immediately** → `sacct -j <id> --format=...` → OOM (137) or TIMEOUT (143)? → read `.out`/`.err`.
3. **Node unavailable** → `sinfo` shows drain/down → `scontrol show node <name>` for `Reason=` → fix cause → `scontrol update nodename=X state=resume`.
4. **Everything broken** → munge (clocks + key) → `scontrol ping` → logs: `slurmctld.log` (head), `slurmd.log` (compute), usually under `/var/log/slurm*`.
5. **Config drift** → `slurm.conf` must be identical fleet-wide (or configless). After edits: `scontrol reconfigure`.

---

## The reframe (what you own vs what you don't)

The *concept* is easy, and the workload code you can't control is also the part you don't need to own. What you own is **(1) the request-vs-reality gap** (partition/limits/resources) and **(2) cluster health/auth** (munge, drained nodes, config drift). Both are infrastructure — exactly where Linux experience pays off.
