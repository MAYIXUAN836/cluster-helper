# SenseCore-style worker reference

Use this reference when operating workers controlled by files under `sensecore-worker/runs/<worker_id>/`.

## Mental model

- The SSH/login/development shell is for reading logs, editing code on AFS/shared storage, and writing small control files.
- The real SenseCore/platform job entry owns GPU visibility and launches the worker.
- The supervisor snapshots `runs/<worker_id>/target.sh` and runs that snapshot.
- Control files are consumed only while the real platform supervisor is alive.

## GPU visibility rule

Do not use `nvidia-smi` from SSH/login/development shell to decide:

- whether the worker has 1 GPU or 8 GPUs;
- whether the real job GPUs are busy;
- whether a target is stuck or healthy.

That shell may show a different GPU namespace or unrelated users' processes.

Valid health evidence:

- `sensecore-worker/runs/<worker_id>/STATUS`
- `sensecore-worker/runs/<worker_id>/CURRENT_TARGET`
- latest `supervisor_*.log`
- latest `target_*.log`
- output files/metrics/checkpoints actually advancing

NCCL/model-loading lines in target logs plus advancing artifacts usually mean the real platform job is running. A login-shell `nvidia-smi` snapshot is not evidence against that.

## Read-only health check

```bash
cat sensecore-worker/runs/<worker_id>/STATUS
cat sensecore-worker/runs/<worker_id>/CURRENT_TARGET
cat sensecore-worker/runs/<worker_id>/LAST_FAILURE 2>/dev/null || true
ls -lt sensecore-worker/runs/<worker_id>/logs | head
tail -n 200 sensecore-worker/runs/<worker_id>/logs/<latest-target-or-supervisor-log>
```

If evaluating a generator, also count concrete outputs:

```bash
find <output-dir>/<variant> -maxdepth 1 -name '*.png' | wc -l
find <output-dir>/rows -maxdepth 1 -name '*.json' | wc -l
```

## Safe target switch

Only switch after the task script is complete and syntax-checked.

```bash
bash -n sensecore-worker/<task_script>.sh
bash sensecore-worker/configure_target.sh \
  <worker_id> \
  /mnt/afs_mayixuan/v-jepa-image-edit-main/activate_env.sh \
  /mnt/afs_mayixuan/v-jepa-image-edit-main \
  /mnt/afs_mayixuan/v-jepa-image-edit-main/sensecore-worker/<task_script>.sh
```

Then confirm:

```bash
cat sensecore-worker/runs/<worker_id>/STATUS
cat sensecore-worker/runs/<worker_id>/CURRENT_TARGET
tail -n 100 "$(ls -t sensecore-worker/runs/<worker_id>/logs/target_*.log | head -1)"
```

## Forbidden actions

Do not run these from SSH/login shell:

```bash
nohup bash sensecore-worker/<worker_id>.sh &
bash sensecore-worker/<worker_id>.sh
bash sensecore-worker/supervisor.sh
touch sensecore-worker/runs/<worker_id>/STOP
kill <supervisor_pid>
```

Do not hot-edit:

- `supervisor.sh`
- dated worker entries such as `2026-08-01.sh`
- `new_session.py`
- active snapshot under `runs/<worker_id>/running/`

## Status interpretation

- `RUNNING`: target or holder is active. Check target log and output progress.
- `WAITING_FOR_REPAIR`: previous target failed; inspect `LAST_FAILURE` and first real error.
- `COMPLETED_WAITING_FOR_PUBLISH`: target exited 0 and holder is keeping the job alive for further target switches.
- `SUCCEEDED`: platform task likely ended; control-file edits cannot restart it.

## Evaluation naming rule

For benchmark or oracle diagnostics:

- Official benchmark score requires same data, split, generation protocol, judge, and metric.
- Held-out internal subsets are diagnostics, not official benchmarks.
- PICA-style Qwen judging on custom data is not PICABench.
- PSNR/SSIM/LPIPS/DINO/CLIP against held-out targets are reference metrics, not official PICA metrics.
- Non-edited-region metrics require real masks/edit areas; never infer masks silently.
