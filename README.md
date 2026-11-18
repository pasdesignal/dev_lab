# dev_lab

A portable, RHEL-based development VM, built with Vagrant + Ansible. Think of it as the VM equivalent of a VS Code dev container: `vagrant up`, connect your editor, and you have a consistent AlmaLinux 9 environment with Podman ready to go — on Apple Silicon or Intel.

- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Choosing a host architecture](#choosing-a-host-architecture)
- [Choosing a profile](#choosing-a-profile)
- [Pinning a different box version](#pinning-a-different-box-version)
- [Running a tagged version](#running-a-tagged-version)
- [Everyday commands](#everyday-commands)
- [Connecting your editor](#connecting-your-editor)
- [Troubleshooting](#troubleshooting)
- [Acknowledgments](#acknowledgments)

## Prerequisites

- [Vagrant](https://www.vagrantup.com/) 2.4+
- [VMware Fusion](https://www.vmware.com/products/fusion.html) (Mac) or VMware Workstation (Linux/Windows)
- [vagrant-vmware-desktop](https://developer.hashicorp.com/vagrant/docs/providers/vmware) plugin
- [VMware Vagrant Utility](https://developer.hashicorp.com/vagrant/install/vmware)
- Optional: a `~/git` directory on your host — if present, it's shared into the VM (see note below)

## Getting started

1. Install VMware Fusion and the [VMware Vagrant Utility](https://developer.hashicorp.com/vagrant/install/vmware).
2. Install Vagrant and the VMware desktop plugin:
   ```sh
   vagrant plugin install vagrant-vmware-desktop
   ```
3. Clone this repo and step into it:
   ```sh
   git clone https://github.com/pasdesignal/dev_lab.git
   cd dev_lab
   ```
4. Bring the VM up:
   ```sh
   vagrant up
   ```
5. Connect:
   ```sh
   vagrant ssh podman1
   ```

> [!NOTE]
> The first `vagrant up` downloads the base image (a few GB) and can take a while. Subsequent runs are fast.

> [!NOTE]
> If a `~/git` directory exists on your host when you run `vagrant up`, it's shared into the guest at `/home/vagrant/git` — handy for editing host-side repos from inside the VM. If it doesn't exist, the Vagrantfile just skips the shared folder (with a console note) rather than failing. Create `~/git` and run `vagrant reload` any time to pick it up later.

The VM uses 8192MB RAM and 4 vCPUs, and gets a private-network IP at `192.168.88.4`.

## Choosing a host architecture

Controlled by `VAGRANT_ARCH`, defaults to `arm64`.

| Architecture | Use when | How to run |
|---|---|---|
| `arm64` (default) | Apple Silicon Mac (M1/M2/M3/M4) | `vagrant up` |
| `amd64` | Intel Mac, or you need x86_64 parity | `VAGRANT_ARCH=amd64 vagrant up` |

> [!TIP]
> On Apple Silicon, `amd64` VMs run under VMware's x86_64 emulation. It works, and for typical lab use (Podman, Terraform, kubectl, cloud CLIs) the overhead isn't noticeable — but native `arm64` is faster if you don't specifically need x86_64. Building images for `amd64` production infra from an `arm64` VM? Use `--platform linux/amd64` with `podman build` rather than switching the whole VM.

## Choosing a profile

Controlled by `VAGRANT_PROFILE`, defaults to empty (base only). `base` always runs exactly once regardless of what you list — profiles are just the extra tooling layered on top.

| Profile | What it adds | How to run |
|---|---|---|
| (none, default) | Podman and general dev tools: podman, podman-plugins, skopeo, bash-completion, make, vim, nmap-ncat, python3-pip | `vagrant up` |
| `terraform` | terraform, kubectl, helm, awscli | `VAGRANT_PROFILE=terraform vagrant up` |
| `wireshark` | wireshark-cli (tshark/dumpcap) for CLI packet capture/analysis | `VAGRANT_PROFILE=wireshark vagrant up` |
| `claude-cli` | Claude Code CLI, for agentic coding-tool development/experiments | `VAGRANT_PROFILE=claude-cli vagrant up` |
| `claude-sandbox` | bubblewrap, socat — needed to run Claude Code with its Bash sandbox enabled (CLI or VS Code extension) | `VAGRANT_PROFILE=claude-sandbox vagrant up` |
| `agent-lab` | anthropic, python-dotenv (pip), for hand-rolled LLM agent experiments | `VAGRANT_PROFILE=agent-lab vagrant up` |

Want more than one? List them comma-separated — `base` still only runs once either way:
```sh
VAGRANT_PROFILE=terraform,agent-lab vagrant up
```

Already have a VM running and want to add/switch profiles? `VAGRANT_PROFILE=terraform vagrant provision` re-runs Ansible without rebuilding the VM (same comma-separated list syntax works here too).

## Pinning a different box version

The Vagrantfile pins a specific `almalinux/9` box build by default. Override it with `VAGRANT_BOX_VERSION` if you need to try a newer release before the default gets bumped:
```sh
VAGRANT_BOX_VERSION=9.9.20270101 vagrant up
```

## Running a tagged version

Releases are tagged directly in this repo using basic semver (`vMAJOR.MINOR.PATCH` — minor for additive changes like a new profile, patch for fixes, major only for breaking changes to how you invoke `vagrant up`). `vagrant up` always uses whatever's currently checked out, so to run a specific tagged version rather than the latest commit on `main`:

```sh
git fetch --tags
git tag -l                  # see what's available, e.g. v1.1.0
git checkout v1.1.0         # detached HEAD at that release
vagrant up
```

To get back to the latest development state afterwards: `git checkout main`.

## Everyday commands

| Command | What it does |
|---|---|
| `vagrant up` | Boot and provision the VM |
| `vagrant halt` | Shut it down (restart with `vagrant up`) |
| `vagrant destroy -f` | Shut down and **erase** the VM entirely |
| `vagrant provision` | Re-run Ansible against an already-running VM |
| `vagrant ssh podman1` | SSH in via Vagrant |

## Connecting your editor

`vagrant ssh podman1` works out of the box — no key setup needed. To connect an editor instead of a terminal (VS Code's Remote-SSH is the closest equivalent to opening a dev container):

```sh
vagrant ssh-config podman1 >> ~/.ssh/config
```

Then in VS Code: **Remote-SSH: Connect to Host…** → `podman1`. That opens a remote window on the VM the same way opening a dev container would — just backed by a VM instead of an OCI container, and it picks up whichever profile you provisioned with.

You can also reach the VM directly by IP, though this is less reliable than going through Vagrant (depends on host firewall/network state):
```sh
ssh vagrant@192.168.88.4
```

### Claude Code history survives rebuilds

Claude Code's session transcripts and per-project memory (what `/resume` and the VS Code extension's past-conversations list read) are kept on your host in `~/data/claude/`. That folder is mounted in the VM at `/data`, and Vagrant creates it if it's missing (set `VAGRANT_DATA_DIR` to use a different host path). The base profile symlinks `~/.claude/projects`, `~/.claude/plans` and `~/.claude/file-history` in the VM to `/data/claude`, so a `vagrant destroy && vagrant up` brings your history back. This works for both the CLI and the VS Code extension with no extra setup. It also sets `cleanupPeriodDays` in `/etc/claude-code/managed-settings.json` so old transcripts aren't deleted after the default 30 days. Your login (`~/.claude/.credentials.json`) and the rest of `~/.claude` stay inside the VM, so you'll still sign in again after a rebuild. Skipped if `/data` isn't mounted.

## Troubleshooting

**SSH ends up on a port other than 2222 (e.g. 2250).**
On macOS 26.7+, a Vagrant bug makes every host port look in use, so `vagrant up` reports "Fixed port collision for 22 => 2222" and moves SSH. `vagrant ssh podman1` still works; for your editor, re-run `vagrant ssh-config podman1` and update `~/.ssh/config` with the new port. Tracked upstream in [hashicorp/vagrant#13845](https://github.com/hashicorp/vagrant/issues/13845), which also has a workaround.

**No host-key warning after a rebuild (`vagrant destroy && vagrant up`).**
Expected: `vagrant ssh-config` disables host-key checking for this disposable VM, so the new key on each rebuild is accepted silently.

**Clock looks wrong inside the VM.**
Check with `chronyc tracking` and `timedatectl`. Chrony is the sole time source (VMware Tools time sync is disabled), so it should self-correct within seconds. If it doesn't, re-run `vagrant provision`.

**`git push` fails with `Permission denied (publickey)` in a VS Code terminal.**
The VM uses your host's SSH key through agent forwarding, and a terminal opened before VS Code reconnected points at a stale agent socket. The base profile's `/etc/profile.d/ssh-agent-refresh.sh` repairs this before each prompt. If it still fails, run `ssh-add -l` in the VM: "no identities" means the key isn't loaded in your host's agent (run `ssh-add` there).

**A tool from a profile (e.g. `terraform`) isn't there.**
You're probably on the `base` profile. Add it without rebuilding: `VAGRANT_PROFILE=terraform vagrant provision`.

**Provisioning fails partway through `vagrant up`.**
The VM itself is usually fine. Re-run just Ansible with `vagrant provision` (repeat `VAGRANT_PROFILE=...` if you're not using the default) rather than rebuilding.

**`/home/vagrant/git` isn't there inside the VM.**
The shared folder is only mounted if `~/git` existed on your host at `vagrant up` time. Create it (`mkdir -p ~/git`) and run `vagrant reload`.

**Can't reach `192.168.88.4` directly, but `vagrant ssh` works.**
That's host networking, not the VM: check macOS firewall settings and VMware's network service. `vagrant ssh` goes over NAT and doesn't need the private network, so prefer it when in doubt.

**`vagrant plugin install vagrant-vmware-desktop` fails, or Vagrant can't find VMware.**
Make sure VMware Fusion/Workstation and the VMware Vagrant Utility are both installed and up to date; a missing or mismatched utility is the most common cause.

## Acknowledgments

This project started as a personal recreation of the lab environment for Red Hat Training's **DO180** course ("Introduction to Containers, Kubernetes, and Red Hat OpenShift"). None of the original course-derived material remains in this repo, but it's the reason this project exists, so it's credited here.

Licensed under MIT — see [LICENSE](LICENSE).
