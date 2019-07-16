# Working with servers

## Available storage volumes

Space | Mount | Capacity | Quota per user | Available On | Backup Policy | Wiping Policy
:--- | :---:| :---: | :---: | :---: | :---: | :---:
RRC Home | `~/` | 11 TB | 50 GB | Green, Neon | At most once a day<sup>#</sup> | None
Scratch | `/scratch` | 880 GB | None* | Green, Neon | None | 14 days<sup>+</sup>
Hybrid HDD | | 1 TB | None | Matrix | None | None

<sup>*</sup> Quota of 300 GB per user exists on Green.

<sup>+</sup> The wiping policy is based on `atime`.

<sup>#</sup> The backup scripts are fired at 03:00 everyday. The scripts do not start if there is an ongoing backup from the previous day. Retrieving data that is older than 30 days is not possible.

## Choosing the right storage

The following are the recommendations that will help you get the best the storage systems have to offer.

- Use `/scratch` for anything that your code uses. This is a `raid0` of two SSDs and hence is expected to have extremely high throughput and low latencies.
- Use your home directories to store the models (or other data that you want to persist).
- Do **NOT** use your home directory for IO intensive tasks. The home directory is a network mount and heavy IO on this volume is not only bad for the performance of your code but also may lead to network congestion there by effecting other users.
- Try to save the files on your home directory in an archived format.

## Extending the storage quotas

Please drop a mail to rrc-sysads@lists.iiit.ac.in in case you want the quota to be extended. In most of the cases, you may not need Prof's approval for quota extensions.