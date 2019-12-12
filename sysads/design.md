# Design decisions

## Deep learning cluster
We have four nodes at the moment in this cluster.

- nucleus.rrc.iiit.ac.in - Hosts services like LDAP, monitoring.
- black.rrc.iiit.ac.in - Storage backend for the cluster.
- green.rrc.iiit.ac.in - Compute node.
- neon.rrc.iiit.ac.in - Compute node.

### Authentication
We use a centralized authentication mechanism. The main motivation for is
the ability to manage the access rules from a single place. The only 
drawback at the moment is that we do not have a highly available directory
service. So, if the LDAP service goes down, things will be messy.

> What do we use for LDAP?

We use [FreeIPA](https://www.freeipa.org/page/Main_Page). It is simple to
setup and comes with a clean web UI.

> Where all is this LDAP used?

We want to use it everywhere ideally. For now, it is on

- `green` and `neon` to control user access.
- `black` for quota management of the NFS mounts. 

### Disks
In the past, we got several complaints from users that storage was too
restrictive (it was indeed, they had only 15GB per user). We want to make
sure that users are able to store their data reliably. Unfortunately, we
do not have a solid backup solution. At the moment, we backup everything to
Google Drive of `robotics@iiit.ac.in`. We are also restricted by the network
uplink (100 Mbits/s). Considering the network bandwidth caps, we decided to
start with 50 GB of storage per user. We benchmarked the time taken for
archived directory-by-directory sync and file-by-file sync on Google Drive.
The archive based uploads take ~3 days to complete and file by file
sync takes ~1 day for the backup (assuming users do not dump thousands of
files in a single day). We should be able to handle ~200GB quota for 2 or 3
users assuming that the data is not frequently updating without effecting
the backup times.

The compute nodes will have `/home` mounted from `black`. Black as a raid5
of 4x 4TB HDDs (5400 RPM each :sad:). This will serve as a single slow high
capacity storage. All the home directories of the users will be on this
machine. We backup these home directories to Google Drive.

Each compute node will also have a raid0 of 2 SSDs (we plan to make them 3).
This raid volume is mounted at `/scratch` on each compute node. This is 
where we want the users to work. Once they are done, they can copy their
outputs, final code, etc., to their directories.

> TLDR; /home is your super slow cold storage with data guarantees.
> /scratch is your super fast, low latency storage for all your
> operations.

#### Disk quotas
The quota for home directories is controlled on `black`. We usually do not
have quota for `/scratch`. We have a script that checks for free disk space
on the compute node and caps the users to 300 GB if there is only 15% free
space left. The quota will be relaxed as soon as there is more than 15%
disk space available.

The quota for home directory is refreshed every day at 00:00 using a cron job.

#### Datasets
We also store the frequently used datasets on black and make it available via
`datasets.rrc.iiit.ac.in`. These datasets will be stored on the same
raid volume that has home directories of the users.

### Software Packaging
There will be different groups using different versions of everything. We
need to make sure that the OS updates won't effect the user activity.
So we came to the conclusion that **we should not install anything using OS
package manager (`apt`) unless it is something that belongs to the OS (like
X server).**

Any package needs to be available on the server should be compiled from
source and installed to `/opt`. We use
[environment modules](https://github.com/cea-hpc/modules) to add necessary
environment variables to use the package. A typical use case can be found
[here](../using-servers/packages.md).

## TODOs
- FreeIPA supports high availability setups out of the box. We need to set up
one. We also plan to decommission `nucleus`. So LDAP and monitoring should
move to `black` eventually.
- College is shutting down GSuite by end of the academic year. We have a 
shaky and hacky way of backing things up to Onedrive (takes ~4 days, lol).
At the moment, we tried archiving user directory and pushing it using
`rclone`. Things are not pretty. We need a better way to do this.
- We want to move away from this model as it is not easy to always build
packages from the source (we also need to figure out all the environment
variables needed to make it work as well). We are experimenting a kubernetes
based solution where we want to assign a container to user with specific CPU,
Memory and GPU caps. We have a clear idea of what to do but we need more
hands to build this. 