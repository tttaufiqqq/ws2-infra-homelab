# Docs index

Read these in order.

1. [Architecture](architecture.md): the machines, the network, the secrets flow and the delivery pipeline, and why each choice was made. Read this first.
2. [Rebuild](deploy.md): the order the environment was stood up in, what each step produced, and how it was torn down. Read this to see the sequence.
3. [Implementation log](implementation-log.md): the timeline from the first Terraform scaffold to the monitoring drills, with what failed and why each fix was made.
4. [Troubleshooting](troubleshooting.md): symptom-to-fix reference. Not meant to be read start to finish; come back to it when something breaks.
5. [Azure mapping](azure-mapping.md): how each part would map to Azure, for the rebuild project.

## Reference write-ups

These were written alongside the build, in the application repository, and moved here with the infrastructure. They go deeper than the pages above.

- [Hardening](02-hardening.md): firewalls, auto-start and defence in depth for each machine.
- [Ansible](06-ansible.md): how the fleet is configured and the order playbooks run in.
- [Terraform](07-terraform.md): one-time prerequisites, the test loop versus the real fleet, and lessons from importing live machines.
- [Production hardening](09-production-hardening.md): deploy safety, TLS, Vault secrets and the known follow-ups.
- [Backups](10-backups.md): coordinated cross-database backups, retention, the restore drill and alerting.

## Other folders

- [`examples/`](examples/): the daily Terraform drift-check workflow, kept as an example only.
- [`images/`](images/): diagrams used by these pages.

Start at the [repository README](../README.md) if you haven't already. It covers the same ground in less detail.
