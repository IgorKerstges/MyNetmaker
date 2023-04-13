# Server config

In this document, I describe a working setup of a Netmaker instance on Alma 9 Linux on a minimal VPS.

## VPS Specs

| Description | Details |
| :--- | :--- |
| Provider | IONOS |
| Subscription | "Virtual Server Cloud S" |
| CPU | 1 vCore |
| RAM | 512 MB |
| SSD | 10 GB |
| Console Type | KVM |

## Cloud-init

The cloud provider offers cloud-init at server initialisation. I'm using this option to prepare the server:

* Pre-install packages (nano, git, yum-utils, epel-release)
* fqdn (set the servername)
* Create sudo-user with ssh-key
* Set server access (remove root-login and disable password authentication)

:memo: Mind: The rest of this document relies heavily on the initial setup. If no cloud-init is used, it will be up to the user to install packages and prepare the sudo user in advance.

```yaml
#cloud-config
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
  - restart ssh
```

The linux user "nm-admin" will be used also in the second part of this document.

# Netmaker config

In this second part, we will take care of all the requirements to run the Netmaker server instance. Since Netmaker server will run in Docker, I want to make sure that we won't need the root user to login but we will make sure to add configurations to our user 'nm-admin' home directory and will take care that 'nm-admin' can run docker.

:memo: Login with sudo-user account for all steps below!

## Step 1: Configure firewall

First we want to take care to add a policy so we can get our Let's Encrypt keys:

`sudo iptables --policy FORWARD ACCEPT`

Alma Linux uses firewalld, we will install firewalld and ensure HTTP, HTTPS and port range 51821-51830/udp can pass:

```bash
sudo dnf install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --zone=public --permanent --add-port=51821-51830/udp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

:memo: Make sure that the output of the `--list-all` shows like following:

```
[nm-admin@nm-gateway ~]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens192
  sources:
  services: cockpit dhcpv6-client http https ssh <-- WE NEED THESE SERVICES ACTIVE/OPEN!
  ports: 51821-51830/udp <-- WE NEED THESE PORTS TO BE OPEN!
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
 When satisfied, we can save our changes with:
 
```bash
sudo firewall-cmd --runtime-to-permanent
```

## Step 2: Install Docker

Instead of the recommended 'docker.io' we will install 'docker-ce', adding the repository and install

```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo systemctl enable docker
sudo systemctl start docker
```

With docker installed, we make sure that our user can run it

```bash
sudo usermod -a -G docker nm-admin
newgrp docker
```

Now we verify the docker version

```
docker version
```

## Step 3: Install Docker Compose

At the time of writing of this document, the current version of docker-compose is v2.17.2. We can find the current latest release in the [docker Github repository](https://github.com/docker/compose/releases). In the next step, this version number is needed to construct the path to the download location for our curl command.

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.17.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version
```

:memo: The version number is part of the curl command in the first line

## Step 4: Enable Wireguard Kernel Module

Wireguard is part of the kernel and we can activate it. After activating, we need to briefly step into the root account ('su') to make this change also load after reboot. As final step, we install the wireguard-tools, so we can configure it

```bash
sudo modprobe wireguard
su
echo wireguard > /etc/modules-load.d/wireguard.conf
exit
sudo dnf install wireguard-tools -y
```

## Step 5: Prepare MQ(TT)

Netmaker needs MQTT to communicate configuration changes between nodes. wget downloads the necessary files to our user's home directory

```bash
sudo wget -O mosquitto.conf https://raw.githubusercontent.com/gravitl/netmaker/master/docker/mosquitto.conf
sudo wget -q -O wait.sh https://raw.githubusercontent.com/gravitl/netmaker/master/docker/wait.sh
sudo chmod +x wait.sh
```

## Step 6: Install Netclient

We will install Netclient so the first node can be created with the server as part of the network

```bash
curl -sL 'https://rpm.netmaker.org/gpg.key' | sudo tee /tmp/gpg.key
curl -sL 'https://rpm.netmaker.org/netclient-repo' | sudo tee /etc/yum.repos.d/netclient.repo
sudo rpm --import /tmp/gpg.key
sudo dnf check-update
sudo dnf install netclient -y
```

## Step 7: Install Netmaker

To install netmaker, we will download the relevant config files and modify them to our needs

```bash
wget -O docker-compose.yml https://raw.githubusercontent.com/gravitl/netmaker/master/compose/docker-compose.yml
wget -O Caddyfile https://raw.githubusercontent.com/gravitl/netmaker/master/docker/Caddyfile
```

With the files 'docker-compose.yml' and 'Caddyfile' downloaded to our homedirectory, we need some modifications:

### Modify references to root-directory

The downloaded 'docker-compose.yml' contains some references to the '/root/' directory, but we need to change these to the home directory of our 'nm-admin' user

```bash
sed -i 's+/root/Caddyfile+/home/nm-admin/Caddyfile+g' docker-compose.yml
sed -i 's+/root/mosquitto.conf+/home/nm-admin/mosquitto.conf+g' docker-compose.yml
sed -i 's+/root/wait.sh+/home/nm-admin/wait.sh+g' docker-compose.yml
```

### Insert the version number

The downloaded 'docker-compose.yml' contains some references to the Netmaker version. At the time of writing of this document, the current version of Netmaker is v0.18.5. We can find the current latest release in the [official Netmaker Github repository](https://github.com/gravitl/netmaker/releases). In the next step, this version number is needed to insert the relevant version in our 'docker-compose.yml' file

```bash
sed -i 's+REPLACE_SERVER_IMAGE_TAG+v0.18.5+g' docker-compose.yml
sed -i 's+REPLACE_UI_IMAGE_TAG+v0.18.5+g' docker-compose.yml
```

:memo: The version number is part of both sed-commands

### Take care of some last bits and pieces

The last four lines follow similar steps as explained in the [Quick start guide of the Netmaker docs](https://docs.netmaker.org/quick-start.html).

:memo: Although the exact same information needs to be feeded into the sed commands, please be aware that the lines below are slightly modified to reference the correct path-to-file

```bash
sed -i 's/NETMAKER_BASE_DOMAIN/<your-subdomain-goes-here>/g' docker-compose.yml
sed -i "s/NETMAKER_BASE_DOMAIN/<your-subdomain-goes-here>/g" Caddyfile
sed -i 's/SERVER_PUBLIC_IP/<your-netmaker-server-ip-address-goes-here>/g' docker-compose.yml
sed -i 's/YOUR_EMAIL/<your-email-address-goes-here>/g' Caddyfile
```

:checkered_flag: With all the above, the server should be good to go:

```bash
docker-compose up -d
```

Good luck, cheers!

----------

## Addendum

With this configuration, we land in an 'empty box'. When opening "https://dashboard.your.subdomain.com", no network exists and so no nodes are created. To get going, following steps are needed:

Step 1- Create a network.
Step 2- Generate enrollment keys for the network of step 1.
Step 3- Open the details of the newly generated enrollment key, copy the whole line in 'Register Command' to clipboard.
Step 4- Start a terminal session and login with sudo-user 'nm-admin'.
Step 5- paste the 'Register Command' content with sudo, Example:

```
sudo netclient register -t eyJzZXJ2ZXIiOiJhcGkubmV0Lndvb2R3b3JrZXIubGlmZSIsInZhbHVlIjoieUtHeThpN0pwRkh1eVFGaklzSkxIOWk1bEhGMjJZd2QifQ==
```

Step 6- Back in the browser ("https://dashboard.your.subdomain.com"), you will now have a node connected.

With these steps, other nodes can be added as needed. Either add other netclient nodes (repeating steps 3, 4 & 5 on the other machine) or use wireguard to connect.

To add a Wireguard connection: In the browser, choose 'Ext. clients' (left side-menu) and from the dropdown, choose the network which was created in Step 1. Now we find the option to "Add External client" which can be done by click on the "+"-sign. This will create a Wireguard profile which shows in the 'Clients' section of the screen. Either use the QR-Code option to connect (especially easy for mobile devices) or download the .ovpn file to use for a Wireguard connection from laptop or pc.
