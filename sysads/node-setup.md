# Compute node setup

This document details the process involved in setting up a new compute node, and adding it to the RRC compute cluster. This can serve as a reference for sysadmins, and for users to understand how things are setup.

We assume that all the required hardware is properly installed and we only discuss the software setup. We begin with a fresh installation of Ubuntu server 18.04.4 LTS.

## Working environment

We first copy our public SSH key to `/root/.ssh/authorized_keys` on the server, and login as root. Root login using password is disabled by default, and only works with key-based authentication.

Inside our root environment, we begin by installing our favourite cli tools:

```bash
apt update
apt install vim tmux htop iotop nload sysstat git python-pip python3-pip zsh corkscrew
```

We then set zsh to be our default shell, and install prezto for its configuration. We also setup the proxy environment variables, even though this is optional. To do all this we run the following script:

```bash
curl -L https://researchweb.iiit.ac.in/~srisai.poonganam/setup/shell-setup.zsh | zsh -
```

We then add the following lines to the system-wide `/etc/bash.bashrc` file to export them to the users' shell environments.

```bash
# Force logout idle sessions
if [[ -n $SSH_CONNECTION ]] ; then
        # Close idle user session after 2 hours
        export TMOUT=7200
fi

# Query and show home quota usage from black
echo ""
curl -s black.iiit.ac.in:8080/$USER 2> /dev/null
echo ""
```

### SSH keys

We then generate SSH keys for this node using the following command:

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

Generating these keys is a required setup for IPA enrolment (explained below), but it is also handy for several other purposes.

We use SSH to connect to Github for automated pull and push commands to our repositories (eg: sysad-scripts, etckeeper). We have a dummy admin account, named rrc-iith, that has access to all the repositories on our RoboticsIIITH organization page. We just add the generated public key to this account on Github to be able to authenticate as this account and run our commands.

SSH connections to Github (or any external server) however are blocked within our network. Fortunately, Github supports SSH over the HTTPS port to the ssh.github.com server. To force all connections to Github to run through this server and port, we add the following lines to the `~/.ssh/config` file:

```bash
Host github.com
	User rrc-iiith
	Port 443
	Hostname ssh.github.com
	TCPKeepAlive yes
	IdentitiesOnly yes
	ProxyCommand corkscrew proxy.iiit.ac.in 8080 %h %p
```
Further, we setup our '/root/.gitconfig' as follows:
```bash
git config --global user.name 'rrc-bot' user.email 'root@blue.rrc.iiit.ac.in'
```

## Hostname

We add an A record, on our DNS management service on Cloudfare, to point the IP address of our new node to a new hostname, for example - blue. The full hostname would be `blue.rrcx.ml` which then gets redirected to`blue.rrc.iiit.ac.in` by our university DNS servers. Optionally, we can also create a CNAME request to our university to point this to `blue.iiit.ac.in`.

## Home directories mount

All the home directories of our users are stored on the black server under the `/mnt/rrc/home` directory. This directory is exported to our compute nodes and mounted using NFS.

To enable our node to access this export, we add the following line (with the node IP) to the `/etc/exports` file on black:

```bash
/mnt/rrc/home 10.4.18.9(rw,async,no_root_squash,no_subtree_check)
```

We then make sure its contents are read by the NFS daemon by running `exportfs -a`.

On the node (client side), we run the following commands to manually mount the exported home directories:

```bash
apt install nfs-common
mount -v -t nfs 10.4.18.3:/mnt/rrc/home /home
```

To enable automatic mounting (on startup), we add the following line to `/etc/fstab:`

```bash
10.4.18.3:/mnt/rrc/home /home nfs rw,soft,intr 0 0
```

## Scratch space mount

Each compute node has attached to it a 1 TB SSD drive (sometimes 2 SSDs configured as a RAID0 drive) meant for storage of temporary dataset and other I/O files for users to work with. To mount this drive, we run the following commands (with the right device file name):

```bash
mkdir /scratch
mount /dev/sdb1 /scratch
```

In addition, like before, we add the following line to `/etc/fstab` (use `blkid` to identify the right UUID):

```bash
/dev/disk/by-uuid/b63e7654-3442-4eb7-98d4-25e78618d235 /scratch ext4 defaults 0 0
```

## Software packages

`TODO`

We build all the packages we need from source and install it to `/opt/`.

CUDA, cuDNN, Singularity, gcc, Matlab

## Environment Modules

We often need to build and manage different versions of the same tool (eg: CUDA), for different projects. The hard way to manage them is to manually change our PATH variable to point to the right version of the tool we want to use. The easier way is to use the [Environment Modules]([https://modules.readthedocs.io/en/latest/index.html](https://modules.readthedocs.io/en/latest/index.html)) package. Modules provides a clean interface for dynamically modifying a user's environment variables using modulefiles. This allows us to easily load and work with different versions of a tool or library without having to manually change our PATH variable for each one.

To install Modules we run the following commands:

```bash
apt install automake autopoint tclsh tcl8.6-dev
mkdir -p /opt/Modules/modulefiles
./configure --prefix=/opt/Modules 
	--modulefilesdir=/opt/Modules/modulefiles
make && make install
```

Then we add the following lines to `/etc/bash.bashrc` and `/etc/zsh/zshrc` , respectively, to make Modules available at shell startup:

```bash
source /opt/Modules/init/bash
source /opt/Modules/init/zsh
```

## Monitoring tools

`TODO`

Prometheus node metrics exporter

 `/opt/metrics/node_exporter`

`systemctl start metrics-exporter.service`

Prometheus GPU metrics exporter

`/opt/metrics/nvidia_gpu_prometheus_exporter.service`

`systemctl start gpu-metrics.exporter.service`

`/opt/prometheus/prometheus.yml`

```bash
- targets: ['blue.iiit.ac.in:9100']
      labels:
        group: 'compute_nodes'
        hostname: blue.rrc.iiit.ac.in
        app: node_exporter

- targets: ['blue.iiit.ac.in:9445']
      labels:
        group: 'gpu_nodes'
        hostname: blue.rrc.iiit.ac.in
        app: nvidia_gpu_prometheus_exporter
```

`systemctl restart prometheus.service`

## Custom scripts

`TODO`

sysad-scripts

## /etc backup

The `/etc` directory contains many important system configuration files that need to be kept track of. We use etckeeper to keep track of these files. It stores all the files inside a git repository and automatically commits any changes made to these files. It can also push these changes to a remote repository for backups. The setup is simple:

```jsx
apt install etckeeper
cd /etc
etckeeper init
```

To sync it to a remote repository, we create a *private* repository (eg: blue-etckeeper) on the RoboticsIIITH organization page and give administrative access to the SysAds team. Then we run `git remote add origin git@github.com:RoboticsIIITH/blue-etckeeper.git` inside the `/etc` folder, and edit the PUSH_REMOTE option in `/etc/etckeeper/etckeeper.conf` with the name of the remote - origin. The SSH config for Github needs to be properly setup for this to work, as described earlier.

## Disable auto-upgrades

The apt package manager periodically updates the package lists to . However, we never intend to install any of these updates as they can easily break someone's workflow. So to disable this we set `APT::Periodic::Update-Package-Lists` to "0" in the file `/etc/apt/apt.conf.d/20-autoupgrades`. 

Moreover, apt also tends to automatically apply any critical security updates. But most of these are updates at the kernel level which require a reboot, so we decided not to install any of these security updates too (!). To disable this we set `APT:Periodic::Unattended-Upgrade` to "0" in the same file.  

## IPA enrolment

We use a FreeIPA server for user and host identity management, and authentication. To enrol this node as a host to the IPA server we run the following commands:

```bash
apt install freeipa-client
ipa-client-install -v
	--mkhomedir 
	--ntp-server=time.iiit.ac.in
	--fixed-primary
	--hostname=blue.rrc.iiit.ac.in
```

Running this installer will require admin IPA credentials, and assumes that the host SSH keys have already been created. Our IPA server is running at `[ipa.rrc.iiit.ac.in](http://ipa.rrc.iiit.ac.in)` and the installer will find it by default by using the domain name `rrc.iiit.ac.in` that is parent to the hostname. The `--mkhomedir` is used to configure the host to create a user's home directory upon login if it does not exist yet. The `--fixed-primary` option specifies that the host will use this server as a fixed server even during failures and will not attempt to fallback to a different server.

## Enable user access

Once the node is enrolled as a host, it should appear under the `Identity→Hosts` tab (use the web ui of the IPA server). To enable users (also enrolled on the IPA server) to access this node, we create a new users group (eg: blue) under the `Identity → Groups` tab, and add the users who'd like to have access to this node to this group.

Then we go to the `Policy → Host-based Access Control` tab and create a new rule (eg: access_blue). Here we specify the name of the users group and the hostname they will get access to, and the default services.
