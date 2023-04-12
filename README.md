# MyNetmaker

In this document, I describe a working setup of a Netmaker instance on Alma 9 Linux on a minimal VPS.

## VPS Specs

**Provider:** IONOS
**Subscription:** "Virtual Server Cloud S"
**CPU:** 1 vCore
**RAM:** 512 MB
**SSD:** 10 GB
**Console Type:** KVM

## Cloud-init

The cloud provider offers cloud-init at server initialisation. I'm using this option to prepare the server:

* Pre-install packages (nano, git, yum-utils, epel-release)
* fqdn (set the servername)
* Create sudo-user with ssh-key
* Set server access (remove root-login and disable password authentication)

<u>Mind:</u> The rest of this document relies heavily on the initial setup. If no cloud-init is used, it will be up to the user to install packages and prepare the sudo user in advance.

`#cloud-config
packages:
  - nano
  - git
  - yum-utils
  - epel-release
fqdn: nm-gateway
users:
  - name: nm-admin
    ssh-authorized-keys:
      - ssh-rsa <AAAAB3Nza.....changeThisKeyBeforeCloudInit......xxxxx>== nm-admin_SSH-key
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
runcmd:
  - sed -i -e '/^PermitRootLogin/s/^.*$/PermitRootLogin no/' /etc/ssh/sshd_config
  - sed -i -e '/^PasswordAuthentication/s/^.*$/PasswordAuthentication no/' /etc/ssh/sshd_config
  - sed -i -e '$aAllowUsers nm-admin' /etc/ssh/sshd_config
  - restart ssh`
  
  The linux user "nm-admin" will be used also in the second part of this document.
  
  

