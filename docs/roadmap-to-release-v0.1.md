# Roadmap To Release v0.1

This milestone is intentionally modest. The goal is not to finish the ideal Raspberry Pi/NFS architecture; the goal is to make the next release boring, repeatable, and honest.

## Release Name

`v0.1` - first reliable Raspberry Pi install

## Release Contract

`v0.1` supports a fresh Raspberry Pi 4 prepared from a separate Ansible control machine. It installs the OpenClaw CLI, prepares a single OpenClaw state directory, supports local storage as the default path, keeps the NFS bridge guarded behind explicit configuration, and documents the manual onboarding step.

After `v0.1`, changes should mostly be bugfixes, small refactors, and documentation corrections until the next planned milestone is opened.

## Supported Use Case

- Control machine: macOS or Linux with Ansible and this repo cloned.
- Target: fresh Raspberry Pi 4 running 64-bit Raspberry Pi OS / Debian.
- Target access: SSH key login to the inventory user.
- Privilege: the inventory user can use sudo; mutating playbooks may require `--ask-become-pass`.
- Storage: local OpenClaw state directory is supported by default.
- OpenClaw setup: CLI install is automated; onboarding remains manual as the `openclaw` user.

## Required Behaviours

- `00-preflight.yml` runs read-only and reports actionable blockers.
- All active playbooks pass `ansible-playbook --syntax-check`.
- `00-bootstrap.yml` handles both `/boot/firmware/config.txt` and `/boot/config.txt`.
- Exactly one storage playbook is selected before installing OpenClaw.
- `10-storage-local.yml` prepares `openclaw_state_dir`.
- `10b-storage-nfs-bridge.yml` fails early when NAS settings are placeholders.
- Storage setup manages `/etc/tmpfiles.d/openclaw.conf` so `/dev/shm/openclaw-tmp` is recreated after reboot.
- `20-openclaw.yml` installs OpenClaw through the pnpm/CLI path.
- Managed environment is written to `{{ openclaw_state_dir }}/.env`.
- `OPENCLAW_STATE_DIR` is set for OpenClaw install and interactive `openclaw` shells.
- Manual onboarding instructions are printed and documented.
- `30-health.yml` does not promise checks it does not perform.

## Covered Workflows

### Fresh Local Install

1. Run preflight.
2. Bootstrap the Pi.
3. Run local storage setup.
4. Install OpenClaw CLI.
5. Run OpenClaw onboarding manually.
6. Verify service and health output.

### Guarded NFS Attempt

1. Leave `nas_host` as the placeholder.
2. Run the NFS playbook in check mode or normal mode.
3. Confirm it fails before sysctl, package, directory, mount, or fstab changes.

This is a supported safety behaviour, not a full NFS deployment guarantee.

## Acceptance Checklist

- [ ] `ansible-playbook --syntax-check 00-preflight.yml 00-bootstrap.yml 10-storage-local.yml 10b-storage-nfs-bridge.yml 20-openclaw.yml 30-health.yml` passes.
- [ ] `ansible-playbook 00-preflight.yml` passes on the target Pi.
- [ ] `ansible-playbook 10b-storage-nfs-bridge.yml --check` fails early while `nas_host` is still a placeholder.
- [ ] `10-storage-local.yml` creates `{{ openclaw_state_dir }}` with `openclaw:openclaw` ownership.
- [ ] `/dev/shm/openclaw-tmp` exists after reboot.
- [ ] `20-openclaw.yml` installs `openclaw` and verifies `openclaw --version`.
- [ ] `{{ openclaw_state_dir }}/.env` exists with mode `0600`.
- [ ] Manual onboarding as `openclaw` completes.
- [ ] Generated OpenClaw daemon is inspected after onboarding.
- [ ] Daemon receives required environment, especially provider keys and `OPENCLAW_STATE_DIR`.
- [ ] README clearly says to choose local storage or NFS, not both.
- [ ] README and `30-health.yml` agree on health dashboard behaviour.
- [ ] Stale pre-trash material is removed or explicitly retained as historical reference.

## Issues To Close For v0.1

- #3 - Ensure `/dev/shm/openclaw-tmp` exists after reboot.
- #7 - Clarify local versus NFS storage path in README.
- #8 - Verify OpenClaw daemon loads the managed `.env` file.
- #9 - Align health playbook with README NFS write-latency claim.

## Already Addressed For v0.1

- `20-openclaw.yml` parses successfully.
- OpenClaw install no longer hardcodes `/usr/lib/node_modules/openclaw/index.js`.
- Raspberry Pi boot config path is detected before editing.
- NFS state path is aligned with `openclaw_state_dir`.
- NFS bridge has an early placeholder guard.
- Managed env file is written to `{{ openclaw_state_dir }}/.env`.
- Storage playbooks manage a tmpfiles.d rule for `/dev/shm/openclaw-tmp`.

## Explicit Non-Goals

- Fully production-ready NFS persistence.
- Docker sandbox installation and hardening.
- UFW, fail2ban, Tailscale, or unattended-upgrades policy.
- Automated OpenClaw onboarding.
- Proving SQLite journal/temp behaviour inside OpenClaw internals.
- Full role-based refactor of the Ansible project.
- Support for non-Pi Debian hosts beyond graceful preflight/health behaviour.

## Deferred Milestones

### v0.2 - NFS Persistence

- Configure a real NAS export.
- Prove OpenClaw state is backed by NFS.
- Add safe NFS write-latency health probing or intentionally avoid that claim.
- Document NAS UID/GID/export requirements.

### v0.3 - Security And Sandbox Layer

- Evaluate Docker sandbox support.
- Decide whether UFW, fail2ban, unattended upgrades, and Tailscale belong in this repo.
- Add systemd hardening once the generated OpenClaw daemon shape is known.

### v0.4 - Cleanup And Structure

- Decide whether to keep numbered playbooks or move to roles.
- Add linting if useful.
- Remove historical artifacts and polish contributor onboarding.

## Release Discipline

Until `v0.1` is reached:

- Prefer small fixes that move checklist items to done.
- Avoid adding optional security/networking layers.
- Avoid changing storage strategy unless it fixes the release contract.
- Document known limitations instead of chasing perfect architecture.

After `v0.1`:

- Default to bugfixes and small refactors.
- Open a new milestone before adding broad new behaviour.
