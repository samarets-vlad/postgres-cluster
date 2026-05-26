# -*- mode: ruby -*-
# vi: set ft=ruby :

# PostgreSQL HA Cluster — 3-node VirtualBox lab
# Nodes: haproxy (56.10), pg1 (56.11), pg2 (56.12)
# Requires: vagrant-vbguest plugin (optional but recommended)
#   vagrant plugin install vagrant-vbguest

VMS = [
  { name: "haproxy", ip: "192.168.56.10", memory: 1024, cpus: 1 },
  { name: "pg1",     ip: "192.168.56.11", memory: 2048, cpus: 2 },
  { name: "pg2",     ip: "192.168.56.12", memory: 2048, cpus: 2 },
]

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_check_update = false

  # Shared SSH key for Ansible — all VMs get the same key pair
  config.ssh.insert_key = false

  VMS.each do |vm|
    config.vm.define vm[:name] do |node|
      node.vm.hostname = vm[:name]
      node.vm.network "private_network", ip: vm[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name   = "pg-lab-#{vm[:name]}"
        vb.memory = vm[:memory]
        vb.cpus   = vm[:cpus]
        # Disable unnecessary virtual hardware
        vb.customize ["modifyvm", :id, "--audio",      "none"]
        vb.customize ["modifyvm", :id, "--uartmode1",  "disconnected"]
      end

      # Provision: create 'vlad' user with passwordless sudo and copy SSH key
      node.vm.provision "shell", inline: <<~SHELL
        set -e
        id -u vlad &>/dev/null || useradd -m -s /bin/bash vlad
        echo 'vlad ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/vlad
        chmod 440 /etc/sudoers.d/vlad
        mkdir -p /home/vlad/.ssh
        # Copy the default Vagrant insecure public key so Ansible can connect
        cp /home/vagrant/.ssh/authorized_keys /home/vlad/.ssh/authorized_keys
        chown -R vlad:vlad /home/vlad/.ssh
        chmod 700 /home/vlad/.ssh
        chmod 600 /home/vlad/.ssh/authorized_keys
      SHELL

      # Only run Ansible from the last VM to ensure all nodes are up
      if vm[:name] == "pg2"
        node.vm.provision "ansible" do |ansible|
          ansible.playbook       = "site.yml"
          ansible.inventory_path = "inventory.ini"
          ansible.limit          = "all"
          ansible.verbose        = false
          ansible.extra_vars     = {
            ansible_ssh_private_key_file: "~/.vagrant.d/insecure_private_key"
          }
          ansible.vault_password_file = ".vault_pass"
        end
      end
    end
  end
end
