---
name: cluster-helper
description: Safely operate SSH-controlled ML cluster workers, SenseCore jobs, AFS/shared-filesystem targets, supervisor logs, GPU visibility, and benchmark/eval launchers. Use when checking or changing worker STATUS/CURRENT_TARGET, switching targets, diagnosing remote training/eval health, or avoiding development-shell nvidia-smi misreads.
---

# Cluster Helper

Use this skill for remote ML cluster operations where a login shell controls files but the real GPUs belong to a platform job.

## Core rule

Never infer a SenseCore/platform job's GPU count, occupancy, or health from `nvidia-smi` in an SSH/login/development shell. That shell can expose a different GPU namespace or unrelated users' processes.

Worker health must come from worker-owned evidence:

- `runs/<worker_id>/STATUS`
- `runs/<worker_id>/CURRENT_TARGET`
- latest supervisor log
- latest target log
- output files, metrics, checkpoints, or generated images advancing under the target output directory

## Workflow

1. Read `references/sensecore-worker.md` before making or diagnosing SenseCore-style worker changes.
2. Inspect state first: `STATUS`, `CURRENT_TARGET`, `LAST_FAILURE`, latest supervisor log, latest target log.
3. Classify the situation:
   - healthy/running: logs and outputs advance, no fatal error;
   - platform/control issue: supervisor/job not alive, wrong target, waiting for repair;
   - code/data issue: Traceback, missing path, failed contract, checkpoint mismatch;
   - resource issue inside target log: OOM, CUDA/NCCL fatal, killed process.
4. For a normal target change, prepare and syntax-check the task script, then switch only through the configured target-switch helper.
5. Do not edit live supervisor files, active target snapshots, or broad control files while a job is running.
6. Do not create `STOP` or `PUBLISH_SUCCESS` unless explicitly requested and semantically correct.
7. Label internal diagnostics clearly. Do not call held-out, reduced, custom, or different-judge results official benchmark scores.

## Completion gate

Report:

- worker id and current status;
- exact target path and snapshot path;
- latest log path;
- output directory and progress counts when applicable;
- whether the issue is healthy progress, control-plane, code/data, or platform-resource related;
- any action taken, with exact file paths.
