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

We use SSH to connect to Github for automated pull and push commands to our repositories (eg: sysad-scripts, etckeeper). We have a bot account, named rrc-iith, that has access to all the repositories on our RoboticsIIITH organization page. We just add the generated public key to this account on Github to be able to authenticate as this account and run our commands.

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

Further, we modify our '/root/.gitconfig' to specify the host as follows:
```bash
git config --global user.name 'rrc-bot' user.email 'root@blue.rrc.iiit.ac.in'
```

## Hostname

We add an A record, on our DNS management service on Cloudflare, to point the IP address of our new node to a hostname (for example - blue). The full hostname would be `blue.rrcx.ml`, where `rrcx.ml` is our registered domain. We have another domain `rrc.iiit.ac.in` which resides within our university servers, over which we don't have direct control, but it is already made to point to `rrcx.ml` using a DNAME record. So, `blue.rrc.iiit.ac.in` is another valid hostname for this node. If needed, we can also request our university server room to add a CNAME record to point this to `blue.iiit.ac.in`.

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

Each compute node has attached to it a 1 TB SSD drive (sometimes 2 SSDs configured as a RAID0 drive) meant for storage of temporary datasets and other I/O files for users to work with. To mount this drive, we run the following commands (with the right device file name):

```bash
mkdir /scratch
mount /dev/sdb1 /scratch
```

In addition, like before, we add the following line to `/etc/fstab` (use `blkid` to identify the right UUID):

```bash
/dev/disk/by-uuid/b63e7654-3442-4eb7-98d4-25e78618d235 /scratch ext4 defaults 0 0
```

## Software packages

Apart from the basic tools already installed in the first step, we try not to install anything using `apt`. Installing any pacakge often brings with it its own version of system dependencies which can silently break someone else's workflow. These dependencies can also get auto-updated during OS upgrades (if enabled). So we expect users to build any packages that they need locally by themselves, or install from `conda-forge` if available. That said, we build some commonly used packages from source by ourselves and install it to `/opt/`. This includes CUDA & cuDNN, Singularity, gcc, and Matlab. However we don't actually need to build all these tools. They are statically built, so we can just copy all their files from the `/opt` directory of another compute node onto this node and things will work just fine.

## Environment Modules

We often need to build and manage different versions of the same tool (eg: CUDA), for different projects. The hard way to manage them is to manually change our PATH variable to point to the right version of the tool we want to use. The easier way is to use the [Environment Modules]([https://modules.readthedocs.io/en/latest/index.html](https://modules.readthedocs.io/en/latest/index.html)) package. Modules provides a clean interface for dynamically modifying a user's environment variables using modulefiles. This allows us to easily load and work with different versions of a tool or library without having to manually change our PATH variable for each one.
To install Modules we run the following commands:

```bash
apt install automake autopoint tclsh tcl8.6-dev
mkdir -p /opt/Modules/modulefiles
./configure --prefix=/opt/Modules \
	--modulefilesdir=/opt/Modules/modulefiles
make && make install
```

Then we add the following lines to `/etc/bash.bashrc` and `/etc/zsh/zshrc` , respectively, to make Modules available at shell startup:

```bash
source /opt/Modules/init/bash
source /opt/Modules/init/zsh
```
Finally, we add modulefiles for all the tools that we want to make available as 'modules' into the `/opt/Modules/modulefiles` directory. The following is an example modulefile for CUDA 10.2, saved as `/opt/Modules/modulefiles/10.2`.

```bash
#%Module1.0
##
## CUDA Modulefile
##
proc ModulesHelp { } {
    global version

    puts stderr "\n\tThis module adds CUDA-10.2 to your environment variable"
    puts stderr "\tDirectory: $root"
}

module-whatis   "adds CUDA-10.2 to your environment variable"

set           version          10.2
set           app              CUDA
set           root             /opt/cuda/10.2
prepend-path   PATH             $root/bin
prepend-path   LD_LIBRARY_PATH  $root/lib64

```

With this file added, CUDA 10.2 can be loaded dynamically using `module load cuda/10.2`. And again, these modulefiles can be directly copied from another node.

## Custom scripts

Next, we clone our `sysad-scripts` repository from our Github page onto the home directory `/root/sysad-scripts`. We also create another directory `scripts` as a symlink to this. This repository contains many maintenance scripts, but the ones relevant for this compute node are,

- `clear-storage.sh`: Deletes files in the `/scratch` directory that haven't been accessed in two weeks. The respective owners are also sent a notification one day before.

- `quota-bot.sh`: Checks the usage of `/scratch` directory and if it is greater than 85% of capacity, a limit of 375 GB is set for every user whose usage is greater than 150 GB.

- `daily-mails.sh`: Reports files in `/tmp` that are not owned by root.ILTO="srisai.poonganam@research.iiit.ac.in,karnik.r@research.iiit.ac.in,aadilmehdi.s@students.iiit.ac.in,rahul.sajnani@research.iiit.ac.in"

We then create a crontab (`crontab.blue`) to automatically run these scripts at certain times, as well as pull any updates to the scripts from the repo.

```bash

0 */3 * * * root cd /root/sysad-scripts && /usr/bin/git pull > /dev/null
0 3 * * * root /root/scripts/clear-storage/clear-storage.sh
0 0 * * * root /root/scripts/daily-mails.sh
0 * * * * root /root/scripts/quota-bot.sh

```
This is then symlinked to `/etc/cron.d/sysad-scripts` where it's picked up by cron.

## Monitoring tools

We use Prometheus for system health monitoring. Prometheus is running on the server at `10.4.18.8:9090`, and it scrapes and stores time-series data from metrics exposed by the OS. We use `node_exporter` for exposing OS and CPU related metrics to Prometheus, and `nvidia_gpu_prometheus_exporter` for GPU related metrics. Both of these are static binaries which can directly be copied from another compute node to `/opt/metrics/`. We also copy the respective service files to `/etc/systems/system/metrics-exporter.service` and `/etc/systemd/system/gpu-metrics.exporter.service`. Then we run the following commands,

```bash
systemctl start metrics-exporter.service
systemctl start gpu-metrics.exporter.service
```

At this point the exporters would now be exposing the metrics for Prometheus. Then, on the Prometheus server, we edit the file `/opt/prometheus/prometheus.yml` to include the following:

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

Finally, we run `systemctl restart prometheus.service` to start scraping the data. We use Grafana for visualizing the scraped metrics from Prometheus, and creating alerts. Grafana is setup on the same server as Prometheus and is available at monitor.rrc.iiit.ac.in. A new dashboard with panels, and alert rules need to be created for this node. These can be directly copied from the dashboard of another node and are not discussed here.

## /etc backup

The `/etc` directory contains many important system configuration files that need to be kept track of. We use etckeeper to keep track of these files. It stores all the files inside a git repository and automatically commits any changes made to these files. It can also push these changes to a remote repository for backups. The setup is simple:

```bash
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
ipa-client-install -v \
	--mkhomedir \
	--ntp-server=time.iiit.ac.in \
	--fixed-primary \
	--hostname=blue.rrc.iiit.ac.in
```

Running this installer will require admin IPA credentials, and assumes that the host SSH keys have already been created. Our IPA server is running at `[ipa.rrc.iiit.ac.in](http://ipa.rrc.iiit.ac.in)` and the installer will find it by default by using the domain name `rrc.iiit.ac.in` that is parent to the hostname. The `--mkhomedir` is used to configure the host to create a user's home directory upon login if it does not exist yet. The `--fixed-primary` option specifies that the host will use this server as a fixed server even during failures and will not attempt to fallback to a different server.

## Enable user access

Once the node is enrolled as a host, it should appear under the `Identity→Hosts` tab (use the web ui of the IPA server). To enable users (also enrolled on the IPA server) to access this node, we create a new users group (eg: blue) under the `Identity → Groups` tab, and add the users who'd like to have access to this node to this group.

Then we go to the `Policy → Host-based Access Control` tab and create a new rule (eg: access_blue). Here we specify the name of the users group and the hostname they will get access to, and the default services.

## Firewall

Finally, we use `iptables` to setup a firewall for the node. `iptables` applies chains of rules to incoming network packets to determine what to do with matching packets. We first run the following commands to setup our table,

 ```bash
# flush existing rules
iptables -F INPUT
iptables -F FORWARD
iptables -F OUTPUT

# default targets
iptables -P INPUT ACCEPT
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

Next, we specify which ports to accept connections from.

```bash
iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT #http
iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT #https
iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT #ssh
iptables -A INPUT -p tcp -m tcp --dport 3389 -j ACCEPT #rdp
iptables -A INPUT -p tcp -m tcp --dport 9445 -j ACCEPT #node_exporter
iptables -A INPUT -p tcp -m tcp --dport 9100 -j ACCEPT #gpu_node_exporter
```

Finally, we setup known source IPs and reject everything else.
```bash
iptables -A INPUT -s 10.4.18.2/32 -j ACCEPT #green
iptables -A INPUT -s 10.4.18.3/32 -j ACCEPT #black
iptables -A INPUT -s 10.4.18.4/32 -j ACCEPT #neon
iptables -A INPUT -p icmp -j ACCEPT 
iptables -A INPUT -s 127.0.0.0/8 -j ACCEPT
iptables -A INPUT -s 10.4.18.7/32 -p tcp -m state --state NEW -m tcp --dport 5666 -j ACCEPT #not sure?
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A INPUT -j REJECT --reject-with icmp-port-unreachable
```
