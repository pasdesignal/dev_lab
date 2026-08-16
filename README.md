# dev_lab

A portable, RHEL-based development VM, built with Vagrant + Ansible. Think of it as the VM equivalent of a VS Code dev container: `vagrant up`, connect your editor, and you have a consistent AlmaLinux 9 environment with Podman ready to go — on Apple Silicon or Intel.

## Prerequisites

- [Vagrant](https://www.vagrantup.com/) 2.4+
- [VMware Fusion](https://www.vmware.com/products/fusion.html) (Mac) or VMware Workstation (Linux/Windows)
- [vagrant-vmware-desktop](https://developer.hashicorp.com/vagrant/docs/providers/vmware) plugin
- VMware Vagrant Utility
- Optional: a `~/git` directory on your host — if present, it's shared into the VM (see note below)

## Getting started

1. Install VMware Fusion and the VMware Vagrant Utility.
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

The VM uses 8096MB RAM and 4 vCPUs, and gets a private-network IP at `192.168.88.4`.

## Choosing a host architecture

Controlled by `VAGRANT_ARCH`, defaults to `arm64`.

| Architecture | Use when | How to run |
|---|---|---|
| `arm64` (default) | Apple Silicon Mac (M1/M2/M3/M4) | `vagrant up` |
| `amd64` | Intel Mac, or you need x86_64 parity | `VAGRANT_ARCH=amd64 vagrant up` |

> [!TIP]
> On Apple Silicon, `amd64` VMs run under VMware's x86_64 emulation. It works, and for typical lab use (Podman, Terraform, kubectl, cloud CLIs) the overhead isn't noticeable — but native `arm64` is faster if you don't specifically need x86_64. Building images for `amd64` production infra from an `arm64` VM? Use `--platform linux/amd64` with `podman build` rather than switching the whole VM.

## Choosing a profile

Controlled by `VAGRANT_PROFILE`, defaults to `base`. Profiles are additive — every profile includes `base` plus whatever it adds.

| Profile | Use when | How to run |
|---|---|---|
| `base` (default) | You just need Podman and general dev tools | `vagrant up` |
| `terraform` | You also need Terraform, kubectl, Helm, and the AWS CLI | `VAGRANT_PROFILE=terraform vagrant up` |

Already have a VM running and want to switch profiles? `VAGRANT_PROFILE=terraform vagrant provision` re-runs Ansible without rebuilding the VM.

## Pinning a different box version

The Vagrantfile pins a specific `almalinux/9` box build by default. Override it with `VAGRANT_BOX_VERSION` if you need to try a newer release before the default gets bumped:
```sh
VAGRANT_BOX_VERSION=9.9.20270101 vagrant up
```

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

## Troubleshooting

> [!TIP]
> **Connecting after a rebuild (`vagrant destroy && vagrant up`) just works, with no warning.** The VM gets a new host key each time, but `vagrant ssh-config`'s output sets `StrictHostKeyChecking no` and `UserKnownHostsFile /dev/null` for this host, so SSH doesn't track or verify it — no "REMOTE HOST IDENTIFICATION HAS CHANGED" prompt to clear. This is a deliberate tradeoff Vagrant makes for disposable VMs like this one, not a bug: since the VM itself is recreated from scratch on every rebuild, there's nothing meaningful for a host-key check to protect here.

> [!TIP]
> **Clock looks wrong inside the VM.** Check with `chronyc tracking` and `timedatectl`. This VM is provisioned to keep chrony as the sole time source (VMware Tools time sync is disabled to avoid the two fighting each other), so it should self-correct within a few seconds. If it doesn't, re-run `vagrant provision` to reapply the time-sync configuration.

> [!TIP]
> **A tool from a profile (e.g. `terraform`) isn't there.** You're probably on the `base` profile. Check what you provisioned with, then run `VAGRANT_PROFILE=terraform vagrant provision` to add it without rebuilding.

> [!TIP]
> **Provisioning fails partway through `vagrant up`.** The VM itself is usually fine — re-run just the Ansible step with `vagrant provision` (add `VAGRANT_PROFILE=...` again if you're not using the default) rather than destroying and rebuilding.

> [!TIP]
> **`/home/vagrant/git` isn't there inside the VM.** The shared folder only gets mounted if `~/git` existed on your host at `vagrant up` time. Create it (`mkdir -p ~/git`) and run `vagrant reload` to pick it up — no need to destroy anything.

> [!TIP]
> **Can't reach `192.168.88.4` directly, but `vagrant ssh` works.** This is host networking, not the VM — check macOS firewall settings and VMware's network service. `vagrant ssh` (NAT-based) doesn't depend on the private network being reachable, so prefer it when in doubt.

> [!TIP]
> **`vagrant plugin install vagrant-vmware-desktop` fails, or Vagrant can't find VMware.** Confirm VMware Fusion/Workstation and the VMware Vagrant Utility are both installed and up to date — Vagrant's VMware provider needs both, and a mismatch is the most common cause of this failure.

## Included tools

**podman1.lab** (`192.168.88.4`)

| Always installed (`base`) | Added by `terraform` profile |
|---|---|
| podman, podman-plugins, skopeo | terraform |
| bash-completion, make, vim | kubectl |
| | helm |
| | awscli |

## Acknowledgments

This project started as a personal recreation of the lab environment for Red Hat Training's **DO180** course ("Introduction to Containers, Kubernetes, and Red Hat OpenShift"). None of the original course-derived material remains in this repo, but it's the reason this project exists, so it's credited here.

Licensed under MIT — see [LICENSE](LICENSE).
