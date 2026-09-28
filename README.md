# queuectl — Background Job Queue

A command-line background job queue written in pure Python (standard library only). Jobs are shell commands persisted in SQLite; a pool of worker processes claims them atomically, retries failures with exponential backoff, enforces timeouts and moves permanently failing jobs to a dead-letter queue. A small HTTP endpoint exposes queue metrics.

## Features

- **Persistent queue** — SQLite (`~/.queuectl/queue.db`) with `jobs` and `config` tables; survives restarts.
- **Parallel workers** — foreground workers or a background daemon (PID file) that spawns N worker processes via `multiprocessing`.
- **Atomic claiming** — a worker moves a job to `processing` inside a transaction, so two workers never run the same job.
- **Retries with exponential backoff** — `next_run_at = now + backoff_base ^ attempts`; configurable base and default retry limit.
- **Dead-letter queue** — jobs that exhaust their retries move to `dead`; list them and re-queue with `dlq retry`.
- **Priorities, scheduling and tags** — higher `priority` runs first; `run_at` delays a job until a given time.
- **Timeouts and per-job logs** — `job-timeout` config; stdout/stderr written to `~/.queuectl/logs/<job_id>.log`.
- **Metrics** — `queuectl metrics serve` exposes job counts per state as JSON at `/metrics`.

## Job Lifecycle

```
pending ──claim──► processing ──exit 0──► completed
   ▲                   │
   └── backoff wait ◄──┤ non-zero exit / timeout
                       └── retries exhausted ──► dead (DLQ) ──dlq retry──► pending
```

## Usage

Requires Python 3.8+; no third-party packages.

```bash
# enqueue jobs (JSON payload)
python -m queuectl enqueue '{"id":"job1","command":"echo hello","max_retries":3}'
python -m queuectl enqueue '{"id":"job2","command":"./backup.sh","priority":10,"run_at":"2026-01-01T02:00:00Z"}'

# run workers
python -m queuectl worker start --count 3 --daemon
python -m queuectl worker stop

# inspect
python -m queuectl status
python -m queuectl list --state pending
python -m queuectl dlq list
python -m queuectl dlq retry job1

# configure
python -m queuectl config set backoff-base 2
python -m queuectl config set default-max-retries 3
python -m queuectl config set job-timeout 30

# metrics
python -m queuectl metrics serve --port 8000     # GET /metrics
```

## Project Structure

```
queuectl/
  cli.py       argparse CLI and daemon management
  db.py        SQLite schema, atomic claim, state transitions, DLQ
  worker.py    job execution, timeouts, backoff, logging
  config.py    runtime paths and defaults
  metrics.py   HTTP metrics endpoint
tests/         pytest tests for persistence, priority ordering and timeouts
```

## Testing

```bash
pip install pytest
pytest -q
```

## Demo

[CLI walkthrough video](https://drive.google.com/file/d/1KDWyP1jSF5j1Df9UPtUxYo9I175ybrEc/view?usp=sharing) — enqueueing, parallel workers, retries, DLQ handling and the metrics endpoint.
