# Official OpenClaw Ansible Adoption Tiers

Reviewed on 2026-04-30 against:

- https://docs.openclaw.ai/install/ansible
- https://github.com/openclaw/openclaw-ansible

This repo is not trying to become a blind copy of the official installer. The goal is to keep the Raspberry Pi 4 stateless-node design intact while borrowing official OpenClaw practices that save time, avoid packaging surprises, or harden the deployment.

## Local Design That Stays Ours

- Raspberry Pi 4 as a constrained, purpose-built cognition node.
- NFS-backed persistent data path, currently aimed at Synology.
- ZRAM and RAM-backed SQLite temporary storage.
- Low-privilege OpenClaw runtime user.
- Health playbook tailored to Pi thermals, throttling, memory pressure, NFS, and service state.

## Minimum Starting Assumptions

These assumptions define the clean starting point for this repo. If any of them are false, pause and make the deviation explicit before running the playbooks.

### Control Machine

- Ansible runs from a control machine, not from the Pi itself.
- The control machine has this repo cloned and can reach the Pi over SSH.
- Ansible 2.14+ is available.
- Required collections are installed from `requirements.yml`.
- SSH key-based login to the Pi works for the inventory user.
- The inventory user can become root with sudo.

### Raspberry Pi Target

- Raspberry Pi 4 with 64-bit Raspberry Pi OS / Debian, or another Debian 11+ / Ubuntu 20.04+ target.
- Python 3 is installed at the inventory path, currently `/usr/bin/python3`.
- The Pi has internet access for apt, NodeSource, npm/pnpm, and OpenClaw package installation.
- The Pi has stable power, wired networking if possible, and enough free disk for package installation.
- The active boot config path is detected or set correctly. On the current Pi it is `/boot/firmware/config.txt`.
- The target is preferably fresh: no existing OpenClaw user, service, data directory, Docker setup, firewall policy, or Tailscale state unless we are deliberately migrating.

### Storage And NAS

- If using the NFS bridge, the NAS already exists, is reachable from the Pi, and exports the configured path.
- NFS permissions are planned before mounting. The OpenClaw runtime user must be able to read/write persistent data.
- If NFS is not ready, use the local-storage playbook path intentionally instead of half-configuring both.
- SQLite temp/scratch storage should stay local/RAM-backed; persistent memory can live on NFS.

### Secrets And Onboarding

- API keys and provider tokens belong in `group_vars/secrets.yml`, encrypted with Ansible Vault.
- The vault file should not be committed.
- Installing OpenClaw is not the full setup. The official flow still expects onboarding/configuration as the `openclaw` user.
- Messaging provider login should happen from the `openclaw` account, not from the admin SSH user.

### Safety Checks Before Mutating The Pi

- `ansible -i inventory.ini pi_openclaw -m ping` returns `pong`.
- `ansible -i inventory.ini pi_openclaw -b -m command -a 'id'` returns `uid=0(root)`.
- Syntax checks pass for all playbooks we intend to run.
- Firewall changes are not enabled until the remote access path is clear.
- Existing data is backed up or intentionally ignored before changing storage mounts.

### Current Verified Host State

As of 2026-04-30, the current Pi target was verified as:

- Host: `rpi4a` at `192.168.178.101`.
- Inventory user: `maxadmin`.
- OS: Debian 13.4 `trixie`, aarch64.
- Kernel: Raspberry Pi 6.12 series.
- Active boot config: `/boot/firmware/config.txt`.
- Ansible SSH and sudo escalation: working.
- OpenClaw, Node.js, npm, pnpm, Docker, UFW, fail2ban, Tailscale, and `/home/openclaw`: not yet installed.

## A. Must Keep From The Official Repo

These are compatibility and correctness items. If we ignore them, we risk fighting OpenClaw packaging or runtime assumptions.

- Install OpenClaw through its supported package/CLI path, not by hardcoding a Node module entrypoint.
- Prefer the official `pnpm install -g openclaw@latest` release-mode shape unless we deliberately choose development mode.
- Run the official OpenClaw onboarding/config flow as the `openclaw` user, especially `openclaw onboard --install-daemon` or the equivalent documented CLI sequence.
- Keep OpenClaw host-based. The official model runs the gateway on the host; Docker is for agent/tool sandboxes, not for the gateway itself.
- Keep a dedicated unprivileged `openclaw` user and avoid running the gateway as root.
- Keep Ansible syntax-checkable and idempotent. Official docs explicitly position the installer as safe to rerun; our playbooks should aim for the same property.
- Document Ansible collection requirements. At minimum we use `community.general`; if we adopt official Docker/firewall tasks, add `community.docker` and `ansible.posix`.
- Treat Debian 11+ / Ubuntu 20.04+ with sudo/root access and internet package access as the official baseline. Our current Pi reports Debian 13.4, which fits.

## B. Should Keep

These are good practices from the official installer that fit our design, but they need adaptation to the Pi/NFS layout.

- Systemd hardening for the OpenClaw service: `NoNewPrivileges`, `PrivateTmp`, and a carefully scoped `ProtectSystem`/write-path setup.
- Scoped sudo permissions for service management, if OpenClaw's daemon commands need them. Avoid granting the `openclaw` user broad sudo.
- A configurable Node.js version. The docs currently mention Node.js 24 plus pnpm, while the official repo README still mentions Node.js 22.x plus pnpm. We should keep the version as a variable and verify the current OpenClaw requirement before install.
- A first-class `openclaw` shell environment: PATH, pnpm global bin path, service env, and ownership should all be explicit.
- Official troubleshooting habits: test manually as `openclaw`, inspect `journalctl`, verify the daemon status, and run channel/provider login from the `openclaw` account.
- Check/diff friendly execution before destructive changes: `ansible-playbook --check --diff` where possible.
- Clear release-vs-development install mode, even if this Pi normally uses release mode.
- Security defaults that do not surprise us: explicit SSH access assumptions, explicit exposed ports, and no accidental public gateway binding.

## C. Might Be Nice To Keep

These are useful, but they are not automatically correct for a Raspberry Pi 4.

- Docker CE and Compose V2 for OpenClaw agent/tool sandboxes.
- The official Docker isolation idea, especially DOCKER-USER chain rules that prevent accidental public container ports.
- UFW firewall rules, if we first decide the intended remote-access path and avoid locking ourselves out.
- Fail2ban for SSH brute-force protection.
- Unattended security upgrades, with a Pi-friendly reboot/maintenance policy.
- Tailscale VPN for remote administration, especially if the Pi will be managed away from the LAN.
- Official linting conventions such as ansible-lint and yamllint.
- Development mode support for testing OpenClaw from source.
- Helper scripts similar to the official `run-playbook.sh`, if they make the local flow safer and less typo-prone.

## D. Other / To Be Evaluated Later

These need more context, hardware testing, or a deliberate ops decision before adoption.

- Whether to restructure this repo into official-style roles, or keep the current numbered playbook sequence.
- Whether the OpenClaw home/data layout should stay under `/home/openclaw`, move closer to `/opt/openclaw`, or split code/config/data across both.
- How Docker sandbox storage behaves on a Pi 4 when persistent state is NFS-backed.
- Whether OpenClaw's internal SQLite behavior can be fully steered by environment variables, or whether the current WAL/tmpdir assumptions need verification against OpenClaw source/runtime behavior.
- Whether Tailscale should be mandatory, optional, or excluded because this Pi is LAN-only.
- Whether UFW is worth enabling before Tailscale is configured; firewall changes should be staged to avoid cutting off SSH.
- Whether OpenClaw should track `latest`, a specific version, or a release channel once the first working deployment is established.
- How unattended upgrades should interact with long-running agent work and scheduled Pi reboots.
- Whether official OpenClaw Ansible should be consumed as a Git collection dependency, vendored as reference material, or simply watched upstream.
- How much of the official security model is still necessary if the Pi is physically local, behind NAT, and reachable only from the trusted LAN.

## Known Quirks To Remember

- The official docs call `openclaw-ansible` the source of truth, but the docs page and repo README may lag each other on small details such as Node.js major version.
- The official flow assumes a post-install onboarding step. Installing packages alone is not the full deployment.
- Docker is part of the official sandbox story, not the gateway hosting story.
- The official network-security posture assumes only SSH and Tailscale are publicly reachable. If we skip Tailscale or UFW, we need our own explicit exposure model.
- The official repo disables bare-metal macOS targets. That does not block this project because macOS is only our Ansible control machine, not the deployment target.
- Raspberry Pi OS boot config paths vary by generation. On the current Pi, the active path is `/boot/firmware/config.txt`, not `/boot/config.txt`.
