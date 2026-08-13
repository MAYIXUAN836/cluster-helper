# Cluster Helper

`cluster-helper` is a Codex Agent Skill for safely operating remote ML cluster workers controlled through SSH, shared filesystems, and queue-managed GPU jobs.

It was designed around SenseCore-style workers where the login/development shell is only a control plane, while the real GPU visibility belongs to the platform job entry.

## What it prevents

- Mistaking login-shell `nvidia-smi` for the real job GPU state.
- Starting long-running supervisors from SSH instead of the platform job.
- Hot-editing active supervisor/target snapshots.
- Treating healthy model-loading/NCCL logs as failures.
- Calling preliminary internal diagnostics official benchmark scores.

## Install locally

Clone the repository and copy the skill folder into your Codex skills directory:

```bash
git clone https://github.com/MAYIXUAN836/cluster-helper.git
mkdir -p ~/.agents/skills
cp -R cluster-helper/skill/cluster-helper ~/.agents/skills/cluster-helper
```

Restart Codex if the skill does not appear immediately.

## Use

Invoke it explicitly:

```text
Use $cluster-helper to check SenseCore worker 2026-08-01 and switch its target safely.
```

Or ask naturally about cluster worker health, target switching, SSH/AFS control files, or SenseCore GPU visibility.

## Expected worker layout

The skill assumes a control layout like:

```text
sensecore-worker/
├── OPERATIONS.md
├── configure_target.sh
├── supervisor.sh
└── runs/<worker_id>/
    ├── STATUS
    ├── CURRENT_TARGET
    ├── LAST_FAILURE
    ├── target.sh
    └── logs/
```

For other clusters, adapt the reference file inside `skill/cluster-helper/references/`.

## Safety rule

The central rule is:

> Do not use `nvidia-smi` from the SSH/login/development shell to judge the platform job's GPU count, occupancy, or health.

Use worker-controlled evidence instead: `STATUS`, `CURRENT_TARGET`, supervisor logs, target logs, and advancing output artifacts.
