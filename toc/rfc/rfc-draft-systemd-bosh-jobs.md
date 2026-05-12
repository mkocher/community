# Meta
[meta]: #meta
- Name: Support BOSH Jobs Managed by systemd Without Monit
- Start Date: 2026-05-11
- Author(s): @mkocher
- Status: Draft
- RFC Pull Request: (fill in with PR link after you submit it)
- Related RFCs: (none)
- Affected Component(s): bosh-agent, bosh (director), bosh-cli


## Summary

Add a first-class way for BOSH jobs to be managed by systemd instead of monit. Release developers opt in by adding a `processes.yml` to their job; the BOSH agent generates systemd units automatically, each delegating to BPM for process isolation. Monit-managed and systemd-managed jobs coexist on the same VM, enabling gradual, per-release migration.

## Problem

Monit is the process supervisor for every BOSH-deployed job. It has several problems:

**1. Monit is frozen on an old version due to a license change.**
BOSH stemcells ship monit **5.2.5**, released in March 2011. The current monit version is **5.35.2**. After 5.2.5, monit relicensed to AGPL-3+, which is incompatible with the Apache 2.0 license used by BOSH. The community cannot upgrade without a license exception. [This has been a known blocker since at least 2018.](https://github.com/cloudfoundry/bosh/issues/1295) We are permanently stuck on a 14-year-old process supervisor.

**1. We have already adopted systemd elsewhere in the BOSH stack.**
The BOSH agent itself moved to systemd (replacing runit) when the Noble stemcell was introduced. systemd is the standard process supervisor on every Linux distribution BOSH targets. Continuing to run a separate, frozen process supervisor alongside systemd creates unnecessary complexity.

**1. For jobs that already use BPM, the monit file is pure boilerplate.**
The vast majority of CF BOSH jobs use BPM for process isolation. Their monit files are identical copies of the same template:
```
check process <name>
  with pidfile /var/vcap/sys/run/bpm/<name>/<name>.pid
  start program "/var/vcap/jobs/bpm/bin/bpm start <name>"
  stop program "/var/vcap/jobs/bpm/bin/bpm stop <name>"
  group vcap
```

## Proposal

Release developers can opt any BOSH job into systemd management by adding a `processes.yml` file to the job. When the BOSH agent finds `processes.yml` in a deployed job's directory, it generates systemd unit files for that job. Jobs without `processes.yml` continue to use monit unchanged.

BPM is required for the systemd path. The generated systemd units call `bpm run` (foreground mode) instead of the old `bpm start` (background + PID file).

---

### The `processes.yml` File

`processes.yml` is a new job template file with an **identical schema to `bpm.yml`**. It is the only file release developers need to write for a systemd-managed job — `bpm.yml` is not required when `processes.yml` is present:

```yaml
processes:
- name: cloud_controller_ng
  executable: /var/vcap/packages/cloud_controller_ng/bin/cloud_controller_ng
  args:
  - -c
  - /var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml
  env:
    RAILS_ENV: production
  limits:
    memory: 1500M
```

The agent reads `processes.yml` to generate systemd units and passes it to BPM to run the actual process. Because the schema is identical, release developers can migrate from `bpm.yml` by simply renaming the file.

For releases that need to remain compatible with older stemcell lines still using monit, both `bpm.yml` and `processes.yml` can coexist in the same job. The agent prefers `processes.yml` when present; older agents ignore it and fall back to monit via `bpm.yml` as they do today.

Release developers do not need to learn anything about systemd.

---

### What the Agent Generates

For each entry in `processes.yml`, the agent writes a `bosh-job-<jobname>[-<processname>].service` unit under `/etc/systemd/system/`. Each unit uses `ExecStart=/var/vcap/jobs/bpm/bin/bpm run <jobname> [-p <processname>]`, which runs BPM in foreground (synchronous) mode so systemd manages the process lifecycle directly. Units are configured with `Restart=always` to replicate monit's automatic restart behaviour. All units belong to a `bosh-jobs.target` grouping unit, which allows the agent to start and stop all BOSH-managed services together. The agent writes `bosh-jobs.target` on first use if it is not already present on the stemcell.

---

### Coexistence: Monit and systemd on the Same VM

The agent uses a composite job supervisor that maintains both a monit supervisor and a systemd supervisor simultaneously. Routing is based on the config file suffix:

- Job has `processes.yml` → systemd supervisor
- Job has a `monit` file → monit supervisor (unchanged)
- Job has both → systemd supervisor wins

This means individual releases can migrate independently. An operator can run a VM with some jobs managed by monit and others by systemd. Releases stay deployable on old stemcells until they remove their `monit` file. 

---

### Health Check Dependencies

Monit supports restarting a service when a health check fails. The systemd path needs an equivalent. Rather than building health-check logic into the supervisor itself, this RFC proposes a composable pattern: any process in `processes.yml` can declare that its exit should trigger a restart of another named process.

The systemd path makes this pattern a first-class feature. A process in `processes.yml` can declare `restarts_on_exit: []`:

```yaml
processes:
- name: cloud_controller_ng
- name: cc_healthchecker
  restarts_on_exit:
    - cloud_controller_ng
```

When the agent generates systemd units, it adds `BindsTo=bosh-job-cloud_controller_ng.service` to the `cc_healthchecker` unit. When the health check process exits (for any reason — including a detected failure), systemd stops the bound main unit; `Restart=always` brings it back up.

This design:

- Lets release authors write whatever health check logic they need in their process binary.
- Allows health check processes to be packaged and shared across jobs (the same pattern as healthchecker-release).
- Does not require any changes to the process being monitored — the main process is unaware of the health check.

---

### Run `pre-start` on VM Reboot

Today, `pre-start` scripts only run during director-orchestrated deployments. When a VM reboots without director involvement, monit immediately starts all jobs from its stored configuration — `pre-start` is never called. This is a long-standing gap in BOSH semantics.

The systemd path creates a clean opportunity to fix this. The agent can generate a `Type=oneshot` `bosh-prestart-<job>.service` unit alongside each job unit. Systemd ordering (`After=bosh-prestart-<job>.service`) ensures the pre-start script runs before the job unit starts, both on the initial `bosh start` (via the director calling `run_script("pre-start")`, which delegates to `systemctl start bosh-prestart-<job>.service`) and on subsequent VM reboots (via systemd's boot chain). This makes pre-start and job-start semantically consistent regardless of whether the job was started by a director or recovered from a reboot.

---

### What Is Not In Scope

**Raw systemd unit templates**: This RFC does not add support for release developers writing `.service.erb` templates directly. The agent always generates units from `processes.yml` and delegates to BPM. Pushing releases to run within BPM will lead to more secure bosh releases and avoid exposing the ensire job supervisor surface to release authors. If a future need arises for raw systemd units (e.g., for jobs that truly cannot use BPM), the architecture supports adding this path without reworking the current design.

**Monit removal**: Monit is not removed by this RFC but it should be considered deprecated. Monit continues to work for all jobs that use it today. Monit will be removed in a subsequent major stemcell line.

---

### Director and CLI Changes

The BOSH Director currently requires every job to contain a monit file during upload validation and template rendering. To support systemd jobs, the Director must:

- Allow a job to be uploaded with no monit file, provided a `processes.yml` template is present.
- Skip monit template rendering and packaging for such jobs.

The BOSH CLI already tolerates missing monit files in job directories (`dir_reader.go` guards on file existence). The CLI's `create-release` flow will need to stop treating a missing monit file as an error when `processes.yml` is present.

---

## Migration Path for Release Authors

For an existing BPM-managed job, migration to the systemd path requires three steps:

1. **Add `processes.yml`**: Create `jobs/<jobname>/templates/processes.yml.erb` listing the processes from `bpm.yml`. For most jobs, this is a short file enumerating process names.

2. **Cut a new release**: The new release includes `processes.yml` in the job's template archive.

3. **Deploy**: The agent finds `processes.yml`, generates systemd units, and starts the job through systemd. The monit file, if still present, is silently ignored.

The monit file can be removed from the job at the same time or in a subsequent release. Removing it is recommended but not required.

No changes to `bpm.yml` are needed.

---

## Workstreams

### bosh-agent

A proof-of-concept implementation is available on the `feat/mk/monit-alternative` branch of [cloudfoundry/bosh-agent](https://github.com/cloudfoundry/bosh-agent). It includes:

- `systemdJobSupervisor`: reads `processes.yml`, generates and installs `.service` units, manages their lifecycle via `systemctl`, and watches journald for failure events.
- `compositeJobSupervisor`: wraps both supervisors, routes `AddJob()` calls by config-path suffix, and aggregates `Status()` and `Processes()` results.
- Updated `rendered_job_applier.Configure()`: prefers `processes.yml` over monit when both are present.

Remaining work: health check dependency (`restarts_on_exit`) unit generation, integration tests against a real VM, and validation that the composite supervisor's aggregate status reporting is correct for mixed deployments.

### bosh (Director)

- Relax `validate_monit` in `release_job.rb` to permit jobs with no monit file when a `processes.yml` template is present.
- Guard monit file reading in `job_template_loader.rb`, writing in `rendered_templates_writer.rb`, and hashing in `rendered_job_template.rb` and `rendered_job_instance.rb` against nil/absent monit content.

### bosh-cli

- Relax the error in `create-release` when a job directory has no monit file but does have a `processes.yml`.
- Update `job_renderer.go` to skip monit rendering when monit is absent.

### CF Release Authors

When ready, a release author:

- Adds `processes.yml` to each job.
- Optionally removes the monit file.
- Cuts a new release version.

## Hat Tip

This RFC borrows heavily from @jpalermo's [original proposal](https://github.com/cloudfoundry/bosh-linux-stemcell-builder/discussions/338).