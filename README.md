# CICADA

## at a glance

CICADA is an alternative to the default unix cron. The biggest differences are:

- split your cron job definitions into multiple files instead of reading one large error-prone file.
- built-in logging, capture stdout and stderr, run duration and exit code
- built-in log rotation, no extra config for logrotate required
- built-in reactions to failed jobs simplify alerting
- controlled via YAML files. No hassle with large lines and support for line breaks.
- built-in double-run prevention
- an optional per-job timeout kills hanging jobs and triggers alerting
- every run is recorded in a local job database that can be browsed later

The simplest job definition will look like this:

```yaml
# ~/.config/cicada/my-job.yml
jobs:
  - name: My Job
    exec: /usr/local/bin/my-job
    crontab: "5 2 * * *"
```

See [./job_example.yml](./job_example.yml) for a demonstration of all features.

## cicadad – the daemon

`cicadad` is a constantly running daemon that discovers new configuration and dispatches jobs at the desired
schedule. `cicadad` discovers all users and processes their job definitions.

On startup, it reads [./cicada.toml](./cicada.toml). This configuration is optional. If not found, defaults apply.

Errors that cannot be written to a job log file – broken YAML, unreadable log paths, failed `on_error` commands –
go to the daemon log.

Missed runs are skipped. If the machine was off or `cicadad` was not running at the scheduled time, the job is not
executed later. This matches classic cron behavior and is a deliberate decision.

## cicada – command line utility

With `cicada` users can validate the job configuration and browse the job database.

- `cicada create-config` writes a commented default configuration to
  `${XDG_CONFIG_HOME:-$HOME/.config}/cicada.toml`.
- `cicada validate` finds errors such as invalid job YAML or an invalid `cicada.toml`. It also warns about wrong
  file permissions, for example, when foreign users can write to the job files or the job file folder.
- `cicada info` shows the folders in use, the number of jobs, the version, and similar facts.

## Jobs

Jobs and their schedule are controlled via one or multiple YAML files. By default, all files in
`$HOME/.config/cicada/*.yml` are processed.

A YAML file can contain one or multiple job definitions with a common `defaults` section.

### Keys supported only in `jobs`

- `name`: name of the job. String, maximum length is 250 characters. Mandatory.
- `exec`: the command to execute. It is passed to the configured `shell`, so pipes, quoting, and shell built-ins
  work as expected. Mandatory.
- `enabled`: set to `false` to keep a definition but stop scheduling it. Boolean, default `true`.
- the schedule keys, see [Scheduling](#scheduling). Mandatory.

### Keys supported in `jobs` and `defaults`

A declaration at job level overwrites the default entirely. Lists are not merged.

- `shell`: shell used to execute `exec`, `prerequisites`, `on_success`, and `on_error`. Default `/bin/sh`.
- `working_directory`: working directory of the job. Default `$HOME`.
- `env`: additional environment variables passed to the job.
- `time_out`: maximum runtime of the job, for example `2 hours`. When exceeded, the job is killed, the run is
  recorded as failed, and `on_error` fires. Default: no timeout.
- `block_doublerun`: when the previous run of a job is still active at the next scheduled time, the new run is
  skipped and recorded in the job database. Boolean, default `true`.
- `log_file`: per-job log file, see [Logging](#logging).
- `log_retention`: how long rotated log files are kept. Supports days, weeks, months, and years, for example
  `30 days` or `10 weeks`. Default `30 days`.
- `log_datetime`: prefix every log line with date and time. Disable it if your job already timestamps its output.
  Boolean, default `true`.
- `prerequisites`: commands that must succeed before the job runs, see
  [prerequisites, on_success, on_error](#prerequisites-on_success-on_error).
- `on_success`, `on_error`: commands executed after the job finished, see
  [prerequisites, on_success, on_error](#prerequisites-on_success-on_error).

### Scheduling

Two mutually exclusive styles are supported.

Classic cron syntax:

```yaml
crontab: "5 2 22 3 5"
```

Or the readable split keys:

```yaml
min: 05          # or */15 for every 15 minutes, leading 0 is optional
hour: 2          # or */2 for every two hours
day_of_month: 22 # 1-31
month: 3         # 1-12
day_of_week: 5   # 0-7, Sunday = 0 or 7
```

Omitted split keys behave like `*` in cron.

### Logging

All output of a job is captured and written to its `log_file`:

```text
2026-09-07T13:09:00+02:00 - Example Job: Job started
2026-09-07T13:09:01+02:00 - Example Job: Stdout: some text
2026-09-07T13:09:02+02:00 - Example Job: Stdout: some text
2026-09-07T13:09:02+02:00 - Example Job: Stderr: some error messages
2026-09-07T13:09:34+02:00 - Example Job: Job ended: Exit Code 90, Duration: 34 seconds
```

- Timestamps are RFC 3339, using the time zone of the system settings.
- Output is streamed directly to the log file. Even very verbose jobs are never buffered in memory until completion.
- A daily log rotation with compression is applied automatically and cannot be switched off. If the current log
  file is from yesterday, it gets moved and compressed, like logrotate would do. Rotation can happen while a job is
  writing; the file handle survives the move.
- Multiple jobs may share the same log file.

### prerequisites, on_success, on_error

```yaml
prerequisites:
  - ip a | grep -q 192.168.1.1
  - ping -c1 -t1 192.168.1.2
on_success: >-
  zabbix_sender -c /etc/zabbix/zabbix_agentd.conf
  -k cron.log -i "Job %job completed with exit code %exit_code after %duration seconds"
on_error:
  - >-
    zabbix_sender -c /etc/zabbix/zabbix_agentd.conf
    -k cron.log -i "Job %job failed with exit code %exit_code after %duration seconds.
    See %log_file for details."
  - cat "%log_file" | do-some-things.sh --foo
```

- Each key accepts a string or a list of strings.
- Commands are executed in the given order, not in parallel, using the job's `shell`.
- The job runs only if all `prerequisites` commands exit with exit code 0.
- `on_success` and `on_error` commands are killed after the `commands.time_out` from `cicada.toml`
  (default 15 seconds). The timeout applies to each command separately.

#### Supported macros

- `%job`: the job's `name`
- `%exit_code`: exit code of the job
- `%log_file`: path of the job's log file
- `%duration`: job runtime in seconds

## Job database

Each run is recorded in a SQLite database, by default `~/.local/share/cicada/jobs.sqlite3`, with:

- job name
- full job config as JSON
- start and end date
- exit code of the job command
- exit codes of the `prerequisites` commands as JSON
- cicada errors: all errors that do not go into the job log file
- the last bytes of the job's stderr, size controlled by `logging.stderr_bytes` from `cicada.toml`

Records are deleted after `logging.data_retention` (default 90 days). Use `cicada` to browse the database.
