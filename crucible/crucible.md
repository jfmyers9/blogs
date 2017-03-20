# Crucible

Here at Pivotal, we are heavy users of [BOSH](https://github.com/cloudfoundry/bosh),
a powerful tool which provides a generic abstraction for deploying software to a variety of infrastructures.
BOSH leverages [monit](https://mmonit.com/monit/) to enable developers the ability to manage the lifecycle
of the software that they are deploying.  One side effect of this generic interface, is the proliferation
of bash scripts as a bootstrapping and launching mechanism for jobs.  While bash enables a developer with
a broad amount of power to perform all of the necessary actions to bootstrap their software,
it has proven to be difficult to test reliably and is prone to bugs, many of which have both security and performance implications.
Over the past week, [Christopher Brown](https://github.com/xoebus) and I have been working on defining
a reduced interface for BOSH jobs that will relieve some of this responsibility from the developer,
and hopefully results in a safer and more stable job definitions through BOSH.

## crucible

[Crucible](https://github.com/xoebus/crucible-release) is the prototype of a tool that aims to solve the
problem outlined above. Crucible is responsible for launching processes on a BOSH VM in a container while
also performing and system setup that is required for successful operation. The high level interface that
crucible provides can be explained in two simple commands:

- `crucible start JOB`
- `crucible stop JOB`

These two commands paired with a simple configuration file and [runC](https://github.com/opencontainers/runc)
provides developers with a large majority of the functionality that they need through a tool that is easily
vetted and tested.

### Configuration

Most bash scripts throughout various BOSH release, primarily known as `ctl` scripts, perform the same
class of tasks to bootstrap a job: create directories, tune parameters, drop permissions, etc
Crucible aims to allow the developer to express this setup through simple configuration.
An example schema for this configuration:

```yaml
---
# A run block used for declaring how to launch the job
run:
  path: /path
  args: [arg1, arg2]
  env: [KEY=VALUE]

# Directories that need to be created and made available to the job
directories:
- /path/to/directory

# Limits that should be enforced on the job while it is running
limits:
  open_files: 1024
  memory: 1024
  disk: 1024

# Capabilities that are made available to job
capabilities: [CAPABILITY]

# Sysctl tuning parameters that the job depends on
sysctl:
  net.ipv4.tcp_fin_timeout: 5
```

With such a configuration, a developer can specify the necessary setup and environment required
for successful operation, and then depend on a well tested Crucible tool to provide this
environment in a suitable and stable way. This is a massive improvement on the hundreds of lines
of untested bash that are written across countless BOSH releases.

### Containers

Another massive benefit of Crucible is process isolation through containers. By utilizing containerization,
Crucible provides a consistent and secure environment for jobs to execute. Crucible only allows jobs to 
access their own configuration and resources preventing local interference from co-located jobs. It also
allows the setting of various limits on resources to ensure that jobs do not overstep their bounds and become
a noisy neighbor for other processes on the VM. Crucible can also provide a consistent directory structure
to a job that will abstract it from the implementation of the BOSH VM. For example, we can now mount the BOSH
packages directory as `/packages` abstracting it's actual location on disk from the job. Lastly, Crucible controls
job execution and launches all processes with de-escalated privileges to prevent the security impact of vulnerabilities.

### Standards

Crucible can also enforce standards on jobs to provide a consistent experience for operators that are using
BOSH as a deployment tool. Some examples of standards that can/will be enforced are:

- Logging. Crucible redirects the jobs stdout/stderr to consistent locations on a BOSH VM.
- Draining/Shutdown. Crucible can provide a consistent shutdown experience for all BOSH jobs, including things like sending SIGQUIT on timeout to force a process to dump it's stack.

### Unanswered Questions

We believe there to be a great amount of value in the features that a tool like Crucible would provide, however
there is a great amount of difficulty in provided a generic tool that has the ability to solve everyone's
bootstrapping needs. We may not be able to predict the needs and dependencies of every process and in rare
cases a job will need setup that is outside of the scope of the interface that Crucible provides. How a tool
like crucible deals with situations like these is still an unknown, and it may in fact not be feasible to
use crucible as a generic solution to all BOSH jobs.

Despite these edge cases, Crucible provides a secure and well tested interface for BOSH job developers and
a standard and consistent experience for BOSH job operators.
