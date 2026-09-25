# uhhep-koa

A small command-line driver for the University of Hawaiʻi **Koa** HPC cluster,
plus the orientation the UH HEP group needs to use Koa: what the cluster is,
which queues exist, what CPU and GPU hardware sits behind them, how a job or a
job array is written, where the group's storage lives, and how to move data on
and off the cluster.

The script `koa` runs on your laptop or workstation. It authenticates to Koa
**once** (you answer the Duo prompt), keeps an ssh control socket open, and
then every later command rides that socket without another prompt. It never
handles a password, never stores a credential, and refuses to run compute on
the login node.

Everything below was measured on the cluster on 2026-09-12 unless a link says
otherwise. Hardware and limits change; the ITS pages linked here are the
authority, and `sinfo` on the login node is the truth of the moment.

---

## 1. Koa in one page

Koa is the UH system-wide research cluster run by ITS Cyberinfrastructure with
the Hawaiʻi Data Science Institute. It replaced Mana in December 2023. Any UH
faculty member, staff member or student can get an account; there is no charge
for the shared partitions. Roughly 300 nodes, about 8,800 cores, 62 TB of RAM,
and around 140 GPUs, scheduled by **Slurm** with fair-share.

A few facts that surprise newcomers:

* **Login needs Duo.** Every ssh to `koa.its.hawaii.edu` (and to the data
  transfer node) asks for a UH password and a Duo push. Repeated failed Duo
  attempts lock your UH account, so tools that open many ssh sessions (rsync
  loops, scp in a shell loop, editors that reconnect) must go through ONE
  authenticated connection. That is the whole reason this script exists.
* **The login node is not for work.** Compile, edit, `squeue`, `git` and small
  file operations are fine. Anything that burns CPU for more than a minute or
  two belongs in a job (`sbatch`) or an interactive allocation (`srun`,
  `salloc`). Processes that abuse the login node are killed.
* **`/home` and `/mnt/lustre` are mounted `noexec` on the login node.** A
  Python you install yourself, a conda environment, a binary you built: none of
  them execute on the login node. They execute inside an allocation on a
  compute node. Test everything from a `sandbox` job, not from the login shell.
* **Scratch is purged.** Files on `/mnt/lustre/koa/scratch` not modified for
  90 days are deleted, every day, without warning. Results you want to keep
  go to the group volumes (section 5) or off the cluster.
* **There is a web portal.** Open OnDemand at the Koa Web Portal gives a
  browser terminal, a file browser with upload, and Jupyter sessions, all
  without an ssh client.

Where the details live (UH ITS HPC Confluence space; anonymous read):

| Topic | Page |
|---|---|
| Overview and getting an account | [About Koa (HDSI)](https://datascience.hawaii.edu/about-koa/) · [HPC services and onboarding](https://datascience.hawaii.edu/hpc/) |
| The cluster page | [Koa](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/429752517) |
| Slurm on Koa | [The Slurm Workload Manager](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407692/The+Slurm+Workload+Manager) |
| Partitions and limits | [The Basics: Partition Information on Koa](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407770/The+Basics+Partition+Information+on+Koa) |
| GPUs | [How to Request GPUs on Koa](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407800/How+to+Request+GPUs+on+Koa) |
| Job arrays | [Using Job Arrays to Submit Many Similar Jobs](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407730/Using+Job+Arrays+to+Submit+Many+Similar+Jobs) |
| Picking node features | [Using Constraints to Select the Resources You Want](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407821/Using+Constraints+to+Select+the+Resources+You+Want) |
| More Slurm | [Advanced Slurm Functionality](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407720/Advance+Slurm+Functionality) · [Koa Slurm Questions](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430407784/Koa+-+Slurm+Questions) |
| Storage tiers and purge policy | [Storage Systems](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/1112604696/Storage+Systems) |
| Data transfer | [Data Transfer](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339251/Data+Transfer) · [Globus on Koa](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/459833388/Globus+on+Koa) · [SFTP, scp, rsync](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339086/SFTP+scp+rsync) · [rclone](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339075/rclone) |
| Duo and transfers | [DUO/MFA considerations](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339106/DUO+MFA+considerations) · [MFA problems when transferring data](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339045) |
| Software modules | [Software](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/514097244/Software) |
| Web portal (Open OnDemand) | [Koa Web Portal](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/430571521/Koa+Web+Portal) |
| Buying nodes or storage | [Resource Investment](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/1073840152/Resource+Investment) · [Condo and Storage Group Management](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/2174550026/Condo+and+Storage+Group+Management) |
| Maintenance, policies, help | [Scheduled Maintenance](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339225/Scheduled+Maintenance) · [Policies and Procedures](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339321/Policies+Procedures) · [Contact Us](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339389/Contact+Us) |
| Acknowledging Koa in papers | [Acknowledgement Statement](https://uhawaii.atlassian.net/wiki/spaces/HPC/pages/9339244/Acknowledgement+Statement) |

Hosts:

| Host | Use |
|---|---|
| `koa.its.hawaii.edu` | login nodes (round-robin); job submission, editing, light work |
| `koa-dtn.its.hawaii.edu` | data transfer node for scp/rsync/sftp; a 10-core shared box, not for compute |
| Globus: **UH Koa Collection** | server-side Globus collection over the same filesystems |

The login nodes and the DTN mount the same `/home` and `/mnt/lustre`.

---

## 2. The queues (Slurm partitions)

Slurm calls them partitions. Two families matter to us: the **public** ones,
which every account can use, and the **kill-** ones, which let you borrow
nodes that other labs own on condition that the owner's job can preempt
(kill) yours at any moment. Plus our own.

| Partition | Max wall | Concurrent per user | Nodes per job | Default per job | Use it for |
|---|---|---|---|---|---|
| `sandbox` | 4 h | 2 running | 2 | 1 CPU, 512 MB | testing a script or an environment; short interactive work |
| `shared` **(default)** | 3 d | no limit | 1 | 1 CPU, 512 MB | ordinary single-node jobs; the workhorse |
| `shared-long` | 7 d | 2 running / 5 queued | 1 | 1 CPU, 512 MB | the rare job that genuinely needs more than 3 days |
| `exclusive` | 3 d | no limit | up to 20 | the whole node | MPI or anything that wants entire nodes |
| `exclusive-long` | 7 d | 2 / 5 | up to 20 | the whole node | as above, longer |
| `gpu` | 3 d | no limit | 1 | 1 CPU, 512 MB | GPU jobs on the university-owned GPU nodes |
| `gpu-sandbox` | 4 h | 1 / 1 | 1 | 20 CPU, 30 GB, 1 GPU | testing GPU code; MIG slices only |
| `kill-shared` | 3 d | no limit | 1 | 1 CPU, 512 MB | borrowed lab nodes, including the big GPUs; **preemptible** |
| `kill-exclusive` | 3 d | no limit | up to 20 | the whole node | borrowed whole nodes; **preemptible** |
| `belle2` | 30 d | | 1 | | **the HEP group's own node**; see below |

Rules of thumb:

* Always set `--time`. The default is one minute, and the scheduler
  backfills short jobs into gaps, so an honest short estimate starts sooner.
* Always set `--mem` (or `--mem-per-cpu`). The default 512 MB kills most
  ROOT or Python jobs at startup.
* Use `kill-shared` for throughput you can afford to lose: it sees about a
  hundred extra CPU nodes and every GPU on the cluster, but a job there can be
  killed mid-run. Write checkpoints, or make the job idempotent and resubmit.
* `sandbox` first. A four-hour test on `sandbox` starts within minutes and
  tells you whether your environment executes on a compute node at all.

### The group's own partition: `belle2`

The HEP group owns one node, `vn-08-06-01` (20 cores, about 190 GB of RAM,
no GPU), exposed as the `belle2` partition with a 30-day wall limit and
`AllowAccounts=belle2`. It is also part of `kill-shared`, so when we are not
using it other people's preemptible jobs run there, and our jobs on it
preempt theirs.

You can use it if your Slurm association includes the `belle2` account:

```bash
sacctmgr show assoc user=$USER format=account,partition,qos
```

If `belle2` is not listed, ask the group's PI to have you added. Submit with:

```bash
#SBATCH --partition=belle2 --account=belle2
```

Anyone in the group may use the node in full; there is no per-use
permission. Long fits, toy studies and anything that would otherwise queue
behind the campus belong here.

---

## 3. CPU and GPU resources

Read on 2026-09-12 with `sinfo -o "%P %l %D %c %m %G"`. Counts are nodes of
each shape; cores and memory are per node.

**CPU nodes reachable from the public partitions**

| Partition | Nodes | Cores per node | RAM per node |
|---|---|---|---|
| `shared` / `shared-long` | 91 | 10 to 48 | 40 GB to 1 TB |
| `exclusive` / `exclusive-long` | 48 | 10 to 48 | 40 GB to 1 TB |
| `kill-shared` (CPU nodes) | 101 | 20 to 192 | 128 GB to 2 TB |
| `kill-exclusive` | 40 | 21 to 192 | 159 GB to 2 TB |
| `sandbox` | 8 | 20 | 112 GB |

**GPU nodes**

| Partition | Nodes | GPUs per node | Cores / RAM per node |
|---|---|---|---|
| `gpu` | 1 | 8 × NVIDIA V100 SXM2 (32 GB) | 48 / 193 GB |
| `gpu` | 2 | 8 × RTX 5000 (16 GB) | 48 / 191 GB |
| `gpu` | 1 | 10 × RTX A4000 (16 GB) | 48 / 257 GB |
| `gpu` | 4 | 2 × A30 (24 GB) | 48 / 257 to 515 GB |
| `gpu` | 1 | 4 × A30 MIG slice 2g.12gb | 48 / 257 GB |
| `gpu-sandbox` | 1 | 8 × A30 MIG slice 1g.6gb | 48 / 257 GB |
| `kill-shared` only (lab-owned) | 1 | 4 × H200 NVL (141 GB) | 128 / 2 TB |
| `kill-shared` only | 1 | 2 × H200 NVL | 192 / 770 GB |
| `kill-shared` only | 1 | H100 PCIe + H100 NVL | 64 / 773 GB |
| `kill-shared` only | 1 | 1 × H100 | 64 / 515 GB |
| `kill-shared` only | 1 | 2 × L40 (48 GB) | 48 / 515 GB |
| `kill-shared` only | 8 | 6 to 8 × RTX 2070 / 2080 Ti | 20 / 95 GB |
| `kill-shared` also | | more V100, RTX 5000, A4000 and A30 nodes of the shapes above | |

So the `gpu` partition holds 46 GPUs on nine nodes (Slurm's own accounting:
`TRES=gres/gpu=46`). The modern large-memory parts (H100, H200, L40) are only
reachable through `kill-shared`, i.e. preemptibly.

Requesting a GPU is by generic resource. The type names are Slurm's, exactly
as `sinfo` prints them:

```bash
#SBATCH --partition=gpu
#SBATCH --gres=gpu:1                    # any GPU in the partition
#SBATCH --gres=gpu:NV-V100-SXM2:1       # a specific type
#SBATCH --gres=gpu:NV-A30:2
# preemptible, the big ones:
#SBATCH --partition=kill-shared
#SBATCH --gres=gpu:nvidia_h200_nvl:1
```

Interactive GPU test (the documented form):

```bash
srun -p gpu-sandbox --gres=gpu:1 -t 120 --pty /bin/bash
```

Software: the module tree is full (`module avail`). Of interest to us:
`data/ROOT/6.30.06-foss-2022b`, `lang/Python/…` and the
`lang/Python-bundle-PyPI` bundles, `compiler/NVHPC/…-CUDA-12.6.0`, and
`tools/Singularity` for containers. There is no pandoc and no TeX Live.
`module load` works inside jobs; a user-built environment on scratch runs on
compute nodes (Lustre is executable there, not on the login node).

---

## 4. Writing a job

### A single job

```bash
#!/bin/bash
#SBATCH --job-name=myfit
#SBATCH --partition=shared          # or belle2 with --account=belle2
#SBATCH --time=0-06:00:00           # D-HH:MM:SS; ALWAYS set it
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4           # threads your program will actually use
#SBATCH --mem=16G                   # per node; ALWAYS set it
#SBATCH --output=myfit-%j.out       # %j = job id
#SBATCH --error=myfit-%j.err
#SBATCH --mail-type=END,FAIL        # optional
#SBATCH --mail-user=<you>@hawaii.edu   # optional

module purge
module load data/ROOT/6.30.06-foss-2022b

# scratch is fast and purged; the group volume is where results are kept
WORK=/mnt/lustre/koa/scratch/$USER/myfit/$SLURM_JOB_ID
mkdir -p "$WORK" && cd "$WORK"

srun ./run_fit --threads "$SLURM_CPUS_PER_TASK" --out result.root
cp result.root /mnt/lustre/koa/lab/belle2_group/$USER/myfit/
```

```bash
sbatch myfit.slurm          # prints "Submitted batch job 1234567"
squeue -u $USER             # what is queued and running
scontrol show job 1234567   # why it is pending (Reason=...)
sacct -j 1234567 --format=JobID,State,Elapsed,MaxRSS,CPUTime   # after it ends
seff 1234567                # CPU and memory efficiency; size the next request from it
scancel 1234567
```

Pending reasons you will see: `Priority` (fair-share queue; wait),
`Resources` (nothing free of that shape; consider `kill-shared`),
`QOSMaxJobsPerUserLimit` (the 2-running caps on `sandbox`, `shared-long`),
`ReqNodeNotAvail` (the node is down or reserved for maintenance).

### A job array

One script, N independent tasks, one line of an input list each. This is
the right shape for "run the same thing over 500 files".

```bash
#!/bin/bash
#SBATCH --job-name=skim
#SBATCH --partition=shared
#SBATCH --array=1-500%50            # tasks 1..500, at most 50 running at once
#SBATCH --time=0-02:00:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G
#SBATCH --output=skim-%A_%a.out     # %A = array job id, %a = task id
#SBATCH --error=skim-%A_%a.err

module purge
module load data/ROOT/6.30.06-foss-2022b

# line N of the list is task N's input
input=$(sed -n "${SLURM_ARRAY_TASK_ID}p" inputs.lst)
out=/mnt/lustre/koa/lab/belle2_group/$USER/skim/$(basename "$input" .root).skim.root

# idempotent: a finished task is skipped on resubmission (kill-shared preemption)
[ -s "$out" ] && exit 0
srun ./skim "$input" "$out.part" && mv "$out.part" "$out"
```

Notes:

* The `%50` throttle is a courtesy to the shared queue and a guard for the
  filesystem; leave it in.
* `squeue -u $USER` shows the array as `1234567_[51-500]` pending plus one
  row per running task. `scancel 1234567` cancels the whole array,
  `scancel 1234567_7` one task.
* Task ids need not be contiguous: `--array=1,5,9-12` reruns only those.
* With the write-to-`.part`-then-rename pattern above, a preempted or failed
  array can be resubmitted as it stands and only the unfinished tasks do work.

### Interactive allocations

```bash
srun -p sandbox -c 4 --mem=8G -t 2:00:00 --pty bash     # a shell on a compute node
salloc -p shared -c 8 --mem=32G -t 8:00:00              # an allocation; then srun inside it
```

The `koa step` subcommand of this tool runs a command inside your current
allocation from the laptop, and refuses to fall back to the login node.

---

## 5. Storage

All group members see the same volumes; access is by Unix group. Check yours
with `id`: membership in `belle2_group` (gid 2076) is what grants the two
group volumes.

| Tier | Path | Size / quota | Persistence | Use |
|---|---|---|---|---|
| Home | `/home/$USER` (`~`) | 50 GB per user | backed up; never purged | dotfiles, scripts, small configs; **not** job output |
| Scratch | `/mnt/lustre/koa/scratch/$USER` (`~/koa_scratch`) | 800 TB shared pool, no per-user quota (498 TB free on 2026-09-12); 400 M files cluster-wide | **purged**: anything not modified for 90 days is deleted daily | job working space, staged inputs, intermediate products |
| Group lab volume | `/mnt/lustre/koa/lab/belle2_group` | 500 GB (free tier; 39 GB used) | persistent | small shared results, shared scripts and environments |
| Group KoaStore | `/mnt/lustre/koa/koastore/belle2_group` | 5.0 TB paid (2.6 TB used); expandable at $50 per TB per year | persistent | the group's data park: ntuples, skims, fit inputs, campaign products |

Conventions the group uses:

* Make yourself a directory under each group volume:
  `mkdir -p /mnt/lustre/koa/lab/belle2_group/$USER` and the same under
  `koastore`. Shared datasets live in named top-level directories, not under
  a person.
* A symlink in your home keeps paths short, e.g.
  `ln -s /mnt/lustre/koa/koastore/belle2_group ~/belle2_koastore`.
* Lustre likes few large files and dislikes millions of small ones. Tar up
  directories of small files before parking them; KoaStore has a file-count
  limit as well as a byte limit.
* Quotas and usage:

```bash
lfs quota -h -u $USER /mnt/lustre/koa          # your own Lustre usage and file count
lfs quota -h -g belle2_group /mnt/lustre/koa   # the group's
df -h /mnt/lustre/koa/koastore/belle2_group    # the KoaStore volume itself
```

* `/home` is NFS and `noexec`; `/mnt/lustre` is Lustre and `noexec` on the
  login node only. Build environments on scratch or the lab volume and run
  them from jobs.
* Nothing on Koa is your only copy unless it is in the group volumes or your
  home. Scratch will be purged; treat it as such from day one.

---

## 6. Moving data: Globus, rsync, scp

Three tools, three regimes. The deciding factors are volume, file count,
whether the transfer must survive interruption, and whether the far end has a
Globus collection.

| | Globus | rsync | scp / sftp |
|---|---|---|---|
| Authentication | UH SSO once in the browser; no Duo per transfer | ssh + Duo per connection (use a control socket) | ssh + Duo per connection |
| Runs unattended | yes, server to server; email on completion | only while your shell lives (use `nohup`/`tmux`) | only while your shell lives |
| Survives interruption | resumes itself; retries for days | restartable by rerunning; skips what is done | starts over |
| Integrity check | per-file checksums, built in | `--checksum` on request; otherwise size and mtime | none |
| Many small files | fine (batched) | fine | slow: one round trip per file |
| Large volumes | best: multi-stream, uses the DTN's full link | good; single stream | acceptable for a few GB |
| Incremental sync of a tree | not its model | **its model** | no |
| Laptop endpoint | needs Globus Connect Personal installed and running | any ssh client | any ssh client |
| Typical wall for 100 GB | minutes to tens of minutes | tens of minutes | tens of minutes, no resume |

**Use Globus when**

* the transfer is large (more than a few tens of GB) or has many files, or
  must complete while you are asleep;
* the other end is another facility with a Globus collection (KEKCC, another
  HPC center, a lab with Globus Connect Personal), so the bytes go server to
  server and never touch your laptop;
* you want a checksum-verified, logged transfer you can point at in a
  README a year later.

Koa's collection is **UH Koa Collection** (log in with your UH identity;
browse to `/home/<user>`, `/mnt/lustre/koa/scratch/<user>`, or the group
volumes). From a laptop, install Globus Connect Personal, name your endpoint,
and transfer in the web app or with the `globus` CLI:

```bash
pip install globus-cli
globus login
globus endpoint search "UH Koa"          # UH Koa Collection = 2ccbe971-d515-4656-b17b-c4a25df0f8db
KOA=2ccbe971-d515-4656-b17b-c4a25df0f8db
globus transfer --sync-level checksum --recursive \
    <laptop-endpoint-id>:/data/run42 $KOA:/mnt/lustre/koa/koastore/belle2_group/$USER/run42
```

**Use rsync when**

* you keep a directory tree on both sides in step (code, a results tree that
  grows), and want only the differences to move;
* the transfer is medium-sized and the far end has no Globus but has ssh;
* you may need to stop and restart.

Go through the DTN, not the login node, and through one authenticated ssh
connection so Duo is asked once:

```bash
# from this tool: koa up  (authenticate once), then
koa push /data/run42 /mnt/lustre/koa/koastore/belle2_group/$USER/run42   # tar + ship + sha256 verify
# or plain rsync over the tool's socket to the DTN:
rsync -a --partial --info=progress2 -e "ssh -o ControlPath=~/.ssh/cm-koa-dtn.its.hawaii.edu" \
    /data/run42/ $USER@koa-dtn.its.hawaii.edu:/mnt/lustre/koa/koastore/belle2_group/$USER/run42/
```

Between two remote sites that both have ssh but no Globus (for instance
KEK's login hosts and Koa), rsync from the site that can reach the other:
`rsync -a --partial -e ssh <src>/ $USER@koa-dtn.its.hawaii.edu:<dest>/`,
run inside `tmux`, then a checksum pass on both sides (`sha256sum`) before
deleting anything.

**Use scp when**

* it is one file or a handful (a plot, a log, a script, a tarball under a
  few GB) and you want it now with nothing to set up;
* the transfer is short enough that a broken connection costs nothing.

```bash
scp -o ControlPath=~/.ssh/cm-koa.its.hawaii.edu myfit.slurm $USER@koa.its.hawaii.edu:
scp -o ControlPath=~/.ssh/cm-koa-dtn.its.hawaii.edu $USER@koa-dtn.its.hawaii.edu:/mnt/lustre/koa/scratch/$USER/plot.pdf .
```

`sftp` is scp with an interactive prompt; same regime. `rclone` (ITS page
linked above) is the tool for cloud drives (Google Drive, OneDrive) and is
out of scope here.

**Never** run a transfer loop that opens a fresh ssh per file: every
connection is a Duo push, and a burst of unanswered pushes locks your UH
account.

---

## 7. The `koa` tool

### Install

```bash
git clone https://github.com/jxn511/uhhep-koa.git
cp uhhep-koa/koa ~/bin/koa && chmod +x ~/bin/koa     # or add the clone to PATH
```

Requirements: bash 4+, OpenSSH with ControlMaster support (any modern
OpenSSH), `rsync`, `tar`, `sha256sum` on both ends. Works from Linux, macOS,
and WSL2.

### Configure

Defaults are your login name and the public hostnames. Override with
environment variables or with `~/.config/koa/config` (see `koa.conf.example`):

| Key | Default | Meaning |
|---|---|---|
| `KOA_USER` | `$USER` | your UH username on Koa |
| `KOA_HOST` | `koa.its.hawaii.edu` | login host, or an alias from your ssh config |
| `KOA_DTN` | `koa-dtn.its.hawaii.edu` | the data transfer node |
| `KOA_PART_CPU` | `shared` | partition `koa batch` uses unless told otherwise |
| `KOA_PART_GPU` | `gpu` | partition for GPU submissions |
| `KOA_SCRATCH` | `/mnt/lustre/koa/scratch/$KOA_USER` | your scratch |

An ssh config block is optional but makes everything shorter:

```
Host koa
    HostName koa.its.hawaii.edu
    User <your-uh-username>
Host koa-dtn
    HostName koa-dtn.its.hawaii.edu
    User <your-uh-username>
```

### Use

```
koa up                    authenticate once (interactive; you answer Duo); opens the sockets
koa status                is the socket alive, and what is queued
koa q [user]              the queue with cores and memory per job (squeue's default
                          format hides both); the session's user unless one is named
koa job [user] <jobid>    one job in full: scontrol, live sstat, the sacct record --
                          the Slurm counterpart of LSF's bjobs -l
koa down                  close the sockets

koa run   <cmd...>        run on the LOGIN node (git, squeue, ls: cheap things only)
koa step  [--jobid=N] <cmd...>
                          run inside YOUR CURRENT ALLOCATION (a real compute node);
                          refuses if there is none, or several and none named
koa batch <sbatch-args>   submit a job; output lands in ~/logs/<name>-<jobid>.out
                          unless you pass your own -o
koa queues                partition occupancy, GPU slots per node, pending pressure,
                          and the scheduler's own start estimate for your pending jobs
koa watch <jobid>         poll until the job leaves the queue, then its sacct line
koa tail  <jobid>         last 40 lines of ~/logs/*-<jobid>.out

koa push <path> [dest]    tar + ship via the DTN + verify sha256 on the far end;
                          default dest $KOA_SCRATCH/incoming
koa stage                 what is already in $KOA_SCRATCH/incoming, and free space
koa restore-test <file> [tarball]
                          pull one file back out of a pushed tarball and cmp it
                          against the local original: prove a copy is recoverable
koa verify                prove the bridge reaches a COMPUTE node, not just the gateway
```

The control sockets live under `~/.ssh/`: if your ssh config sets a
`ControlPath` for the host, that one is used; otherwise
`~/.ssh/cm-<host>`, i.e. `~/.ssh/cm-koa.its.hawaii.edu` and
`~/.ssh/cm-koa-dtn.its.hawaii.edu` with the default hosts. Any other ssh,
scp or rsync can ride the same socket with `-o ControlPath=<that path>`, as
the examples in section 6 do.

A first session:

```bash
koa up                                   # password + Duo, once
koa run squeue -u $USER                  # rides the socket, no prompt
koa queues                               # where is room right now
koa batch -p sandbox -t 0:10:00 -J hello --wrap 'hostname; module load data/ROOT/6.30.06-foss-2022b; root -b -q'
koa watch 1234567
koa q                                    # your jobs with their cores and memory
koa job 1234567                          # everything Slurm knows about one job
koa tail 1234567                         # ~/logs/hello-1234567.out
koa batch myfit.slurm                    # a script file works the same way
koa down
```

Exit codes: 0 ok, 1 degraded or refused (for instance `step` with no
allocation), 2 cannot run (no socket).

### What it deliberately does not do

It never handles a password or stores a credential: `koa up` is a plain
interactive ssh and the only secret is in your head and your phone. It never
runs compute on the login node: `step` and `batch` are the only ways it
reaches a compute resource, and both refuse to silently fall back to the
gateway, because a job that quietly runs on the login node looks identical
to a working one until it is killed for CPU abuse.

---

## Contributing

Corrections to the numbers in this README are welcome as pull requests;
please say the date and the command you measured with. Cluster facts drift.

## License

MIT, see `LICENSE`.
