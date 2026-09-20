# Set VAGRANT_ARCH to switch between architectures:
#   arm64 (default) - native performance on Apple Silicon Macs
#   amd64           - x86_64, emulated on Apple Silicon or native on Intel Macs
ARCH = ENV.fetch('VAGRANT_ARCH', 'arm64')

# Set VAGRANT_PROFILE to add tooling on top of the base image - base itself
# always runs exactly once regardless of what's listed here, so profiles
# never need to (and no longer do) import base.yml themselves.
# See playbooks/profiles/ for the available profiles.
#   (unset/empty) - podman, skopeo, and general dev tools only
#   terraform      - + Terraform, kubectl, Helm, AWS CLI
#   wireshark      - + wireshark-cli (tshark/dumpcap; CLI-only, no GUI)
#   claude-cli     - + Claude Code CLI (via Anthropic's dnf repo)
#   agent-lab      - + Python, the Anthropic SDK, python-dotenv
# Combine profiles with a comma-separated list, e.g.:
#   VAGRANT_PROFILE=terraform,agent-lab vagrant up
PROFILES = ENV.fetch('VAGRANT_PROFILE', '').split(',').map(&:strip).reject { |p| p.empty? || p == 'base' }

# Set VAGRANT_BOX_VERSION to pin to a different almalinux/9 box build than
# the default below, e.g. to try a newer release before bumping it here.
BOX_VERSION = ENV.fetch('VAGRANT_BOX_VERSION', '9.8.20260810')

Vagrant.configure("2") do |config|
  config.vm.provider "vmware_desktop"
  config.ssh.insert_key = false
  config.vm.box_check_update = false
  host_git_dir = "#{Dir.home}/git"
  if Dir.exist?(host_git_dir)
    config.vm.synced_folder host_git_dir, "/home/vagrant/git"
  else
    puts "NOTE: #{host_git_dir} not found — skipping the /home/vagrant/git shared folder. Create it and run `vagrant reload` to enable it."
  end
  config.ssh.forward_agent = true
  config.vm.provider "vmware_desktop" do |v|
    v.memory = 8096
    v.cpus = 4
    v.vmx["cpuid.coresPerSocket"] = "2" if ARCH == 'amd64'
  end

  # podman1 VM definition
  config.vm.define "podman1" do |podman1|
    podman1.vm.box = "almalinux/9"
    podman1.vm.box_version = BOX_VERSION
    podman1.vm.hostname = "podman1.lab"
    podman1.vm.network :private_network, ip: "192.168.88.4"
    podman1.vm.provision :shell, inline: "sudo dnf install -y epel-release; sudo dnf config-manager --set-enabled crb; sudo dnf install -y ansible"

    def provision_profile(vm, name, playbook)
      vm.provision :ansible_local, name: name do |ansible|
        ansible.playbook = playbook
        ansible.install = false
        ansible.compatibility_mode = "2.0"
        ansible.inventory_path = "/vagrant/inventory"
        ansible.config_file = "/vagrant/ansible.cfg"
        ansible.limit = "all"
      end
    end

    provision_profile(podman1.vm, "base", "/vagrant/playbooks/profiles/base.yml")
    PROFILES.each do |profile|
      provision_profile(podman1.vm, "profile-#{profile}", "/vagrant/playbooks/profiles/#{profile}.yml")
    end
  end
end
