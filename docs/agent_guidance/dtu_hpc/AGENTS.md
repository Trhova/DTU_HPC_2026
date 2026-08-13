# DTU HPC agent instructions

These instructions apply to agent work in this directory and its descendants.
They are a reusable baseline for DTU HPC projects. A project may add stricter
local rules, explicit roots, approved workflows, and cleanup authorizations;
those additions must not weaken DTU policy or these safety constraints.

Read `GOOD_TO_KNOW.md` for environment context. Follow current official DTU HPC
documentation and direct HPC Support instructions when they supersede recorded
historical details.

## Scope and authority

- Distinguish read-only inspection from state-changing work. Do not infer
  permission to download cohorts, submit production jobs, cancel jobs, delete
  data, publish results, or alter external systems from a request to inspect or
  explain.
- Use one active workflow operator. Other agents may inspect read-only state,
  but must not concurrently edit, submit, cancel, repair, or clean the same
  workflow without an explicit handoff.
- Before acting, identify the exact project root, checkout, workflow, job IDs,
  data paths, and authorized scope.
- Do not run broad filesystem discovery outside the project. Prefer targeted
  `git`, `rg`, `ls`, `sed`, and exact-path checks.

## Login nodes, compute nodes, and transfers

- Treat login nodes as a control plane. Keep all commands there short and
  bounded.
- Submit sustained downloads, large checksum or compression tests, database
  validation, profiling, assembly, and analysis through LSF.
- Use `transfer.gbar.dtu.dk` for bulk transfers. If it is unavailable or
  SFTP-only, use a currently approved transfer method or contact HPC Support.
  Do not improvise by routing bulk traffic through a login node.
- Never expose SSH private keys, tokens, passwords, or credential-bearing
  configuration in logs, repositories, or responses.
- Use the user's actual quota command, such as
  `/apps9/dcc/bin/getquota_work3.sh`, rather than `df`, when evaluating `/work3`
  capacity.

## Git and checkout safety

- Before every Git mutation, explicitly enter the intended repository and
  verify `git rev-parse --show-toplevel`, the branch, `git status`, and remotes.
- Preserve unrelated or user-owned changes. Never alter `.vscode/` unless the
  user explicitly requests that exact change.
- Use GitHub as the source of truth between local and HPC checkouts: commit and
  push before leaving one checkout, then `git pull --ff-only` in the other.
- Do not edit the same tracked files in two checkouts with uncommitted changes.
- Never run `git pull` inside a batch job. Execute a reviewed,
  commit-identified snapshot.
- Keep reads, databases, workflow work directories, runs, and large generated
  logs out of Git. Commit intended code, configuration, manifests,
  documentation, and small reports only.

## Before submitting an LSF job

- Confirm the user authorized the workload and its scale.
- Record the repository commit, workflow/submodule commit, tool versions,
  database identity, input checksums, parameters, and intended job name in the
  ignored run directory.
- Make the wrapper self-contained. Freeze required paths, identifiers,
  parameters, and non-interactive environment activation in the wrapper or an
  immutable parameter file. Do not rely on variables exported only by the
  caller's interactive shell.
- Include explicit `#BSUB` directives for job name, queue, walltime, cores,
  memory, host placement, stdout, and stderr.
- Request one core for a serial job. Use `span[hosts=1]` for a shared-memory
  job. Specify total cores and per-node placement for distributed work.
- Remember that DTU LSF applies `rusage[mem=...]` per slot. Calculate and report
  the total reservation as slots multiplied by memory per slot.
- Inventory all active reservations and check `blimits` before releasing
  concurrent work. Leave explicit headroom for required jobs.
- Verify every directly executed script with `test -x`, inspect its shebang,
  and run the relevant syntax/environment checks. If a script is intentionally
  non-executable, invoke it through an explicit interpreter.
- Submit exactly one representative compute-node smoke job after creating or
  repairing a wrapper. Validate outputs, read/sample counts, logs, CPU, memory,
  process/thread counts, and downstream gates before releasing the production
  array.
- Treat submitted scripts and parameter files as immutable. Use a new versioned
  path for every repair.

## Resource sizing and accounting

- Right-size resources from a monitored production-like run. A reservation is
  not an observed peak, and a low LSF `Max Memory` value is not proof of low
  physical use.
- For file-backed or memory-mapped databases, record both LSF accounting and an
  independent residency/process measurement. Ask HPC Support for the approved
  node-level measure when they disagree.
- After completion, inspect scheduler accounting, including runtime, CPU
  efficiency, memory, process/thread counts, and oversubscription. Use
  `bstat -C JOBID` and `bstat -M JOBID` where available.
- Treat account limits as limits on reserved resources. Verify current values;
  do not encode a historical or temporary limit as permanent policy.
- Physical node availability and personal limits are separate. Do not promise
  an immediate start merely because a request fits beneath the personal limit.
- Do not use `bmod`; it is administrator-only on this DTU installation. Replace
  jobs safely with a new versioned chain when dependencies or resources must
  change.

## Monitoring

- After a user-approved submission, inspect `bjobs`, bounded stdout/stderr, and
  workflow progress through completion or failure whenever practical.
- If persistent monitoring is requested, continue at the latest agreed cadence
  until the chain completes or the user stops it. Interpret and act on each
  check; a background recorder alone is insufficient.
- Sleep on the local machine. Make each remote SSH query short and bounded.
- Interpret `PEND`, `RUN`, `DONE`, and `EXIT` explicitly. `DONE` and `EXIT` are
  terminal even if `bjobs` still lists a recently completed job.
- For arrays, inspect representative and failed elements. For workflow
  managers, inspect the wrapper and child jobs.
- At every production check, cover parent/child state, recent error logs,
  validation and cleanup gates, submission ledgers, and the user's actual
  `/work3` quota.
- Do not attach an unbounded `tail` to slow scheduler commands. Capture bounded
  output, use a timeout, and verify the command runner and children exited.
- Do not leave editor agents, language servers, Codex processes, workflow
  managers, polling shells, or watchers resident on a login node. Terminate
  only an exact, verified stale user-owned PID after checking its process tree
  and active clients; never use broad process-name killing.
- When the user says to stop polling, stop the local loop, verify its runner and
  children exited, and do not restart it without a new request.

## Failure repair and dependency chains

- Diagnose with exact job IDs, logs, dependencies, scripts, and expected output
  paths before changing state.
- After bounded per-file or per-sample retries, quarantine and register a
  failure while allowing independent valid work to continue when safe.
- Never treat a failed input as successful or clean it up as though it passed.
- If an upstream job fails, cancel or replace downstream jobs whose
  dependencies can never be satisfied.
- Before cancelling or replacing any part of an automated chain, inventory its
  complete intended lifecycle: staging, compute, validation, retention,
  authorized cleanup, downstream release, monitoring, and ledger entries.
- Compare the old and replacement dependency graphs. Map every cancelled job
  to a verified replacement or explicitly document why it is no longer needed.
- A repair is complete only when compute, validation, cleanup gates,
  downstream progression, and ledger state are restored and verified.
- After repairing a persistently monitored workflow, resume the monitoring
  loop; a successful repair does not end the monitoring task.

## Validation, retention, and cleanup

- Process exit code alone does not validate scientific output. Require
  non-empty, schema-valid retained tables, expected sample counts, checksums,
  logs, parameters, and provenance.
- Make workflows safe to rerun: fail clearly on missing inputs, do not silently
  overwrite a run directory, and preserve diagnostic evidence.
- Do not delete data merely to relieve pressure without identifying exact
  paths, reacquirability, retained outputs, and recovery implications.
- Cleanup requires validated retained outputs and an exact deletion manifest,
  and must remain within the user's explicit authorization.
- Verify a copied archive by size and checksum before making its source cleanup
  eligible.

## Report after changes or submissions

Report concisely:

- verified repository root, branch, commit, and Git status;
- exact files changed;
- syntax, permission, environment, and smoke checks performed;
- submitted, cancelled, or replacement job IDs and their dependency mapping;
- parent and child job health;
- resource request and aggregate-reservation calculation;
- output validation and cleanup-gate state; and
- remaining risks, time-sensitive assumptions, or required human decisions.
