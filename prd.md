# PRD cicada

## Features

### job logging

```yaml
  log_file: ~/cicada-logs/my-job.log
  log_retention: 30 days # or 10 weeks, or 4 months
```

- Multi-line log file.
- datetime according to system settings time zone, RFC 3339

```text
2026-09-07T13:09:00+02:00 - Example Job: Job started
2026-09-07T13:09:01+02:00 - Example Job: Stdout: some text
2026-09-07T13:09:02+02:00 - Example Job: Stdout: some text
2026-09-07T13:09:02+02:00 - Example Job: Stderr: some error messages
2026-09-07T13:09:34+02:00 - Example Job: Job ended: Exit Code 90, Duration: 34 seconds
```

- Some jobs are very verbose and produce a lot of output on Stdout and Stderr. Output must be streamed
  directly to the log file instead of holding in memory and writing to the log file once the job is completed.
- A daily log rotation with compression is applied automatically, and it cannot be switched off. If the current log file
  is from yesterday, it gets moved and compressed the day after like logrotate would do.
  Rotation can happen while a job is writing to the file. Moving should not destroy the file handle.
- Multiple jobs might use the same log files.

### on_success, on_error, prerequisites

```yaml
  on_success: >-
    zabbix_sender -c /etc/zabbix/zabbix_agentd.conf
    -k cron.log -i "Job %job completed with exit code %exit_code after %duration seconds"
  on_error:
    - >-
      zabbix_sender -c /etc/zabbix/zabbix_agentd.conf
      -k cron.log -i "Job %job failed with exit code %exit_code after %duration seconds
      See %log_file for details."
    - >-
      cat "%log_file" | do-some-things.sh --foo
```

- `on_success`, `on_error`, `prerequisites` can be a string, or a list of strings.
- Commands are executed in the given order, not in parallel.
- Commands are executed using the configured shell of the job.
- Jobs runs only if all prerequisites-commands exit with exit code 0.

#### Supported makros

- %job (jobs->name)
- %exit_code
- %log_file
- %duration (seconds)

### job db

Each job is recorded in the job database, with

- job name
- full job config as json
- date start
- date end
- job command exit code
- prerequisites commands exit codes as json
- cicada errors, all errors that don't go into the job log file
- job stderr, last bytes of stderr according to `logging.stderr_bytes` from cicada.toml

### Command line utility cicada

create-config -> ${XDG_CONFIG_HOME:-$HOME/.config/cicada.toml
validate, finds errors, such as invalid jobs yaml, invalid cicada.toml. creates warnings about wrong file permissions
when foreign users can write to existing job files or the job files folder
info shows folders used, numer of jobs, version, etc.
