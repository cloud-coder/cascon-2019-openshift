# Setting up minishift on the workshop VM machine.

## Official Document:

Bascially follow this instruction doc, https://docs.okd.io/latest/minishift/getting-started/setting-up-virtualization-environment.html#for-linux

## Setting Up the KVM Driver:

1. Find out VM lunux version

   ```bash
   cat /etc/os-release
   ```

   The VM I am working on is Ubuntu 16.04.6 LTS

1. Install libvirt and qemu-kvm on your system
   ```bash
   sudo apt install libvirt-bin qemu-kvm
   ```
1. Add yourself to the libvirt(d) group
   ```bash
   sudo usermod -a -G libvirtd $(whoami)
   ```
1. Update your current session to apply the group change
   ```bash
   sudo newgrp libvirtd
   ```
1. Install the KVM driver binary and make it executable

   ```bash
   sudo curl -L https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.10.0/docker-machine-driver-kvm-ubuntu16.04 -o /usr/local/bin/docker-machine-driver-kvm

   sudo chmod +x /usr/local/bin/docker-machine-driver-kvm
   ```

## Start libvirtd service

1. Check the status of libvirtd:

   ```bash
   systemctl is-active libvirtd
   ```

1. If libvirtd is not active, start the libvirtd service
   ```bash
   sudo systemctl start libvirtd
   ```

## Configure libvirt networking

1. Check your network status:

   ```bash
   sudo virsh net-list --all

   Name       State      Autostart     Persistent
   default    active     yes           yes
   ```

   If your output looks like the above then you’re done. However, if State is not active or Autostart is not yes you’ll need to follow the steps below.

1. Start the default libvirt network:
   ```bash
   sudo virsh net-start default
   ```
1. Now mark the default network as autostart:
   ```bash
   sudo virsh net-autostart default
   ```

## Installing Minishift

1. Download the archive for your operating system from the Minishift Releases page and extract its contents, https://github.com/minishift/minishift/releases

1. Copy the excutable file "minishift" to usr/bin
