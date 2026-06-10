# SLURM "Break It On Purpose" Drills

**Purpose:** Practice the diagnosis side against a sandbox cluster  
**Companion to:** slurm-cheatsheet.md (every symptom maps to its reason-code / exit-code tables)  
**Setup:** Run from a shell on the control node — `docker exec -it docker-slurm-tutorial-control-1 bash`  
**How to use:** Keep the cheatsheet open. For each drill, predict the symptom *before* running it, then confirm with the diagnostic command.  

---

## Drill 1 — Legit queue wait (the most common "problem" that isn't one)

Fill the cluster, then queue behind it.

```bash
sbatch --nodes=2 --exclusive --wrap="sleep 300"   # grabs both nodes
sbatch --wrap="hostname"                           # this one has to wait
squeue
```

**Symptom:** second job sits `PD`, `NODELIST(REASON)` = `(Resources)`.
**Diagnose:** `squeue` (read REASON), then `scontrol show job <id2>`.
**Lesson:** `(Resources)` is *not* a bug — the cluster is just full. The fix is patience or freeing capacity, not config.
**Reset:** `scancel <job1>` (or wait it out).

---

## Drill 2 — Asking for more than exists (request-vs-reality gap)

Try both and notice they fail *differently*.

```bash
sbatch --mem=900G --wrap="hostname"      # nodes have ~6.7 GB
sbatch --nodes=5  --wrap="hostname"      # only 2 nodes exist
```

**Symptom:** often a *submit-time rejection* ("Requested node configuration is not available" / invalid node count) — or it parks in `PD` if the scheduler treats it as not-yet-satisfiable.
**Diagnose:** read sbatch's own stderr first; if it queued, `squeue` REASON + `scontrol show job <id>` vs `sinfo`.
**Lesson:** not every failure is a queue state — some die at submission. Knowing which is half the battle on a call.
**Reset:** `scancel` anything that queued.

---

## Drill 3 — Out of memory (OOM kill)

First check whether memory is even enforced on this cluster:

```bash
grep -i constrain /etc/slurm/cgroup.conf
```

Then:

```bash
sbatch --mem=20M --wrap="python3 -c 'b=bytearray(300*1024*1024); import time; time.sleep(5)'"
sacct -j <id> --format=JobID,State,ExitCode,MaxRSS,ReqMem,Elapsed
```

**Symptom (if `ConstrainRAMSpace=yes`):** `State=OUT_OF_MEMORY`, exit code in the `137` family (128+9 SIGKILL).
**Symptom (if NOT enforced):** the job just *succeeds* — which is itself a finding: this cluster isn't enforcing memory limits, so one job can starve a node. (This is also why `hello.sh`'s `--mem=1` may or may not blow up.)
**Lesson:** OOM is the #1 silent job killer. `MaxRSS` vs `ReqMem` in `sacct` is how you prove it.
**Reset:** none needed.

---

## Drill 4 — Walltime exceeded (TIMEOUT)

```bash
sbatch --time=00:00:10 --wrap="sleep 120"   # 10s limit, 2min job
watch -n1 squeue                             # watch R -> CG -> gone (Ctrl-C to stop)
sacct -j <id> --format=JobID,State,Elapsed,ExitCode
```

**Symptom:** ~10s in, the job is killed; `State=TIMEOUT`.
**Lesson:** users constantly under-set `--time`. TIMEOUT looks like a crash but the fix is just a bigger walltime request.
**Reset:** none needed.

---

## Drill 5 — Drained node (the classic on-call scenario)

```bash
scontrol update nodename=node1 state=drain reason="drill"
sinfo                                  # node1 now 'drain'
sbatch --nodes=2 --wrap="hostname"     # can't get 2 nodes -> PD
scontrol show node node1               # read Reason=
```

**Symptom:** `node1` in `drain` state; multi-node jobs can't be satisfied and pend.
**Diagnose:** `sinfo` + `scontrol show node node1` (the `Reason=` field tells you *why* — here, your "drill" string).
**Lesson:** a drained/down node silently shrinks the cluster. Real `Reason=` values point at failed health checks, OOM events, or admin action.
**Fix / reset:** `scontrol update nodename=node1 state=resume`, then `scancel` the pending job.

---

## Drill 6 — Munge auth break (advanced; the "everything is broken" call)

```bash
docker exec docker-slurm-tutorial-node1-1 systemctl stop munge
# submit work / wait ~1 min, then from control:
sinfo                                  # node1 trends to down* / not responding
scontrol show node node1
docker exec docker-slurm-tutorial-node1-1 cat /var/log/slurm/slurmd.log | tail -20
```

**Symptom:** `node1` goes unreachable (`down*`); slurmd can't authenticate to slurmctld; logs show credential/munge errors.
**Lesson:** this is the cheatsheet's "everything broken → munge (keys + clocks)" path. In the real world the trigger is usually **clock skew** between nodes (which can't be cleanly simulated here — containers share the host clock — so stopping munge is the stand-in). On a real cluster, suspect chrony/ntp first.
**Fix / reset:** `docker exec docker-slurm-tutorial-node1-1 systemctl start munge`, then `scontrol update nodename=node1 state=resume` if it stayed down.

---

## After each drill

Return the cluster to clean before the next one:

```bash
squeue                                              # anything stuck?
scancel -u root                                     # clear all your jobs
sinfo                                               # both nodes back to 'idle'?
scontrol update nodename=node1 state=resume         # if any node left drained/down
```

When `sinfo` shows `node[1-2]  idle` with no asterisk on the state, you're reset and ready for the next one.
