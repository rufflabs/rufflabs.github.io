---
layout: post
title:  "Fixing Kernel Panic in Ubuntu Packer Builds"
categories: 
excerpt: "When building an Ubuntu image using Packer I was building out my own .pkr.hcl template file. Everything seemed to be fine, except I could never get the build to boot into the cloud-init and start installing the operating system. This post discusses the kernel panic and how to solve it."
image: 
permalink: 
---

When building an Ubuntu image using Packer I was building out my own `.pkr.hcl` template file.
Everything seemed to be fine, except I could never get the build to boot into the cloud-init and
start installing the operating system. 

The VM was built and after the boot command was entered the VM would show a Kernel panic error like below.

![Kernel panic - not syncing: No working init found.](/images/posts/2023-03-24-packer-ubuntu-kernel-panic/packer-kernel-panic.png)

After a lot of trial and error I finally found the extremely simple solution. I wasn't giving the VM enough RAM.
After adding the following to the `source` declaration the VM booted up and auto installed using my `cidata` files.

```
  cpus = 2
  memory = 2048
```

Here is the full `ubuntu2204-vmware.pkr.hcl` file that worked for me:

```
packer {
  required_version = ">= 1.7.0"
  required_plugins {
    vmware = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/vmware"
    }
  }
}

source "vmware-iso" "ubuntu" {
  boot_wait   = "10s"
  ssh_timeout = "20m"
  boot_command = [
    "<wait>e<down><down><down><end>",
    " autoinstall ds=nocloud;",
    "<F10>",
  ]
  cd_files = [
    "./cidata/meta-data",
    "./cidata/user-data"
  ]
  cd_label = "cidata"

  iso_url          = "./ISO/ubuntu-22.04.2-live-server-amd64.iso"
  iso_checksum     = "5e38b55d57d94ff029719342357325ed3bda38fa80054f9330dc789cd2d43931"
  ssh_username     = "vagrant"
  ssh_password     = "vagrant"
  shutdown_command = "sudo shutdown -P now"
  // cpus             = 2
  // memory           = 2048
}

build {
  name = "packer-ubuntu2204"
  sources = [
    "sources.vmware-iso.ubuntu"
  ]

  post-processor "vagrant" {

  }
}
```

Here is the associated user-data file, it uses the Vagrant insecure public key and credentials `vagrant:vagrant`:

```
#cloud-config
autoinstall:
  version: 1
  early-commands:
    - sudo systemctl stop ssh
  apt:
    disable_components: []
    geoip: true
    preserve_sources_list: false
    primary:
    - arches:
      - amd64
      - i386
      uri: http://us.archive.ubuntu.com/ubuntu
    - arches:
      - default
      uri: http://ports.ubuntu.com/ubuntu-ports
  drivers:
    install: false
  identity:
    hostname: ubuntu
    password: $6$uNphLrAzLra/kNRo$L1umupXWSPFHA34UiYGHWzC4paSr/Ru9lw8JKMXKd48sVHUT0W0S8hv0n.C2.bHHXbfxSiwt0gXbOXUkIeF0Q.
    realname: Vagrant
    username: vagrant
  kernel:
    package: linux-generic
  keyboard:
    layout: us
    toggle: null
    variant: ''
  locale: en_US.UTF-8
  network:
    ethernets:
      ens33:
        dhcp4: true
    version: 2
  source:
    id: ubuntu-server
    search_drivers: false
  ssh:
    allow-pw: true
    authorized-keys: [
      "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key"
    ]
    install-server: true
  storage:
    config:
    - ptable: gpt
      path: /dev/sda
      wipe: superblock-recursive
      preserve: false
      name: ''
      grub_device: true
      type: disk
      id: disk-sda
    - device: disk-sda
      size: 1048576
      flag: bios_grub
      number: 1
      preserve: false
      grub_device: false
      offset: 1048576
      type: partition
      id: partition-0
    - device: disk-sda
      size: 1902116864
      wipe: superblock
      number: 2
      preserve: false
      grub_device: false
      offset: 2097152
      type: partition
      id: partition-1
    - fstype: ext4
      volume: partition-1
      preserve: false
      type: format
      id: format-0
    - device: disk-sda
      size: 19569573888
      wipe: superblock
      number: 3
      preserve: false
      grub_device: false
      offset: 1904214016
      type: partition
      id: partition-2
    - name: ubuntu-vg
      devices:
      - partition-2
      preserve: false
      type: lvm_volgroup
      id: lvm_volgroup-0
    - name: ubuntu-lv
      volgroup: lvm_volgroup-0
      size: 10737418240B
      wipe: superblock
      preserve: false
      type: lvm_partition
      id: lvm_partition-0
    - fstype: ext4
      volume: lvm_partition-0
      preserve: false
      type: format
      id: format-1
    - path: /
      device: format-1
      type: mount
      id: mount-1
    - path: /boot
      device: format-0
      type: mount
      id: mount-0
  updates: security
  late-commands:
    - echo 'vagrant ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/ubuntu
    - curtin in-target --target=/target -- chmod 440 /etc/sudoers.d/ubuntu
```