# DTU HPC: good to know

This document explains the DTU HPC environment and records practical lessons
learned while running large metagenomics workflows. It is background for human
users. The adjacent `AGENTS.md` contains the operational instructions that an
AI or automation agent must follow.

Time-sensitive facts are labelled explicitly. Recheck live DTU configuration
before relying on queue limits, walltimes, available nodes, or transfer-service
capabilities.

## The basic mental model

| Component | Purpose | Appropriate work |
|---|---|---|
| Local computer / WSL | Editing, Git, lightweight inspection, and initiating transfers | Develop code, review results, and issue short SSH commands |
| HPC login node | Control plane | Edit small files, inspect state, submit jobs, and run short bounded checks |
| LSF compute node | Computation | Downloads submitted through LSF, validation, profiling, assembly, analysis, and other sustained work |
| `transfer.gbar.dtu.dk` | Bulk-data gateway | Upload and download large files |
| Home directory | Small, important, backed-up files | Code, configuration, documentation, and small irreplaceable data |
| `/work3/<user>` | Large working storage, not backed up | Databases, reads, workflow work directories, runs, and large outputs |

The login node is not a workstation or compute node. Heavy or long-running work
belongs in LSF. Large transfers belong on the transfer service.

## Connecting from this WSL installation

The current computer keeps the SSH alias and key in Windows:

- SSH configuration: `C:\Users\trhova\.ssh\config`
- Host alias: `dtu-hpc`
- DTU login endpoint: `login1.hpc.dtu.dk`

WSL can use the Windows OpenSSH client without copying the private key into
Linux:

```bash
/mnt/c/Windows/System32/OpenSSH/ssh.exe dtu-hpc 'hostname; id -un'
```

Never print, copy into a repository, or expose the private key.

## Storage and quota

The home directory is backed up but small. `/work3` is scratch space and is not
backed up. Important code and manifests belong in Git; irreplaceable results
need another verified copy.

Check the user's actual `/work3` quota rather than global filesystem free
space:

```bash
/apps9/dcc/bin/getquota_work3.sh
```

`df` describes the shared filesystem and does not answer how much quota an
individual user has left.

## Transfers

Use `transfer.gbar.dtu.dk` for bulk data. Depending on the current DTU service
configuration, the endpoint may permit SFTP but not `rsync`. In an SFTP-only
period, use SFTP/WinSCP or ask DTU HPC Support for the approved alternative.

An old project note contains an `rsync` example routed through `login2`. It may
have been a temporary workaround while the normal transfer service was down,
but the note does not preserve enough evidence to establish that. It is not a
current operating procedure. If the transfer service is unavailable, check the
DTU service status or contact Support; do not silently redirect a bulk transfer
through a login node.

After a transfer, verify file sizes and checksums at the destination before
removing the source.

## LSF resource requests

DTU uses LSF for batch work. A typical shared-memory request declares the
queue, job name, walltime, logs, cores, memory, and single-host placement.

The important accounting detail is that `rusage[mem=...]` is requested per
slot. For example, eight slots at 60 GiB per slot reserve 480 GiB in total:

```text
8 slots x 60 GiB/slot = 480 GiB reserved
```

LSF scheduling limits are based on reserved resources, not merely what a job
appears to be using at a particular moment. Personal aggregate limits and
physical node availability are separate constraints: fitting under the former
makes a job eligible, not certain to start.

Useful live checks include:

```bash
blimits
bqueues
bjobs -l JOBID
bstat -C JOBID
bstat -M JOBID
```

`bmod` is administrator-only on this DTU installation. A submitted job cannot
be repaired by modifying it in place; safely replace the affected pending
dependency suffix with new, versioned jobs.

### Account-specific, time-bound example

HPC Support temporarily set the `trhova` account limits to 120 slots and
1,478,656 MiB of aggregate reserved memory, with a review date of
**2026-10-01**. This is historical/account-specific information, not a general
DTU rule and not a permanent entitlement. Always check `blimits` before
planning concurrency.

## Memory accounting can be misleading

File-backed and memory-mapped databases may occupy substantial physical RAM
without appearing fully in ordinary LSF `Max Memory` accounting. In one MEDI
case, a roughly 412-GiB Kraken database was independently observed as resident
while LSF reported only about 6–7 GiB.

Therefore:

- do not infer that a large memory request is unnecessary from LSF `Max Memory`
  alone;
- measure the complete workload, including file-backed residency;
- record both LSF accounting and independent evidence; and
- ask HPC Support for the site-approved node-level measure when the figures
  disagree.

The inverse lesson also matters: a reservation is not a measured peak. Resource
requests should be right-sized with one monitored production-like job before
requesting broad concurrency.

## A safe workflow pattern

1. Keep scripts and manifests in Git; keep large data and run directories out
   of Git.
2. Freeze required paths, parameters, environment setup, input identity, tool
   versions, database identity, and Git commit in a self-contained wrapper.
3. Check the wrapper's syntax, shebangs, and executable permissions.
4. Submit one representative compute-node smoke job.
5. Validate its expected outputs, logs, resource use, and cleanup gates.
6. Only then release the full array or rolling production chain.
7. Treat submitted scripts as immutable.
8. Validate retained outputs and provenance before cleanup.

When moving between local and HPC checkouts, commit and push the checkout being
left, then use `git pull --ff-only` in the checkout being entered. GitHub is the
source of truth for code; avoid simultaneous uncommitted edits in both places.

## Monitoring concepts

- `PEND`: waiting for resources or dependencies.
- `RUN`: allocated and running.
- `DONE`: terminal success, subject to output validation.
- `EXIT`: terminal failure.

For arrays, inspect failed and representative members rather than only the
parent summary. For Nextflow or another workflow manager, inspect both the
orchestrator and its child jobs. An idle orchestrator should not retain slots
unless it is actively supervising useful work.

Monitoring from WSL should sleep locally and make short, bounded SSH queries.
Do not leave editor agents, watchers, delayed polling shells, or language
servers resident on an HPC login node.

## Official references

- [DTU HPC: Batch Jobs under LSF 10](https://www.hpc.dtu.dk/?page_id=1416)
- [DTU HPC: Managing jobs](https://www.hpc.dtu.dk/?page_id=1519)
- [DTU HPC: Storage](https://www.hpc.dtu.dk/?page_id=59)
- [DTU HPC: Moving files to/from the HPC system](https://www.hpc.dtu.dk/?page_id=4377)

The official documentation and direct instructions from DTU HPC Support take
precedence over historical project notes.
