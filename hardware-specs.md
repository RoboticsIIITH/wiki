## GPU cluster

|Name | Purpose | CPU | RAM | GPU |Disks|
|:-----:|:-----:|:-----:|:-----:|:-----:|:-------:|
|Black| Storage for user home directories (NFS mount) <br> and datasets ([datasets.rrc.iiit.ac.in](datasets.rrc.iiit.ac.in)) | Core i7-5930K (6 physical cores)| 4x 8 GB G-Skill DDR4 | GTX 680 | 1) 500 GB Crucial SSD (OS) <br> 2) 12 TB RAID5 array using 2x 4 TB WesternDigital NAS HDD and 2x 4TB Seagate NAS HDD (storage)<br> 3) 6 TB Seagate NAS HDD (unmounted)|
|Blue|Compute node| Xeon (?) (20 physical cores)|4x 16 GB Micron DDR4|2x 12 GB GTX Titan X Maxwell|1) 500 Crucial GB SSD (OS)<br> 2) 1 TB WesternDigital HDD (Scratch) |
|Green|Compute node|Xeon (?) (12 physical cores)|3x 16 GB Crucial DDR4 + 1x 8 GB Crucial DDR4|3x 11 GB GTX 1080 Ti Pascal|1) 500 GB Crucial SSD (OS)<br> 2) 1 TB RAID0 array using 2x 500 GB Kingston SSD (Scratch)<br> 3) 500 GB Transcend SSD (Scratch 2)|
|Neon|Compute node|Xeon (?) (12 physical cores)|4x 16 GB Crucial DDR4|4x 12 GB GTX Titan X Maxwell|1) 240 GB Kingston SSD (OS) 2) 1 TB RAID0 array using 2x 500 GB Kingston SSD (Scratch)<br> 3) 3 TB Seagate HDD (Unmounted)|
|Nucleus|IPA & Monitoring|Core i7-2600K (4 physical cores)|4x 4 GB Corsair DDR3|-|1) 1 TB Toshiba HDD|

## Desktop PCs
|Name | Purpose | CPU | RAM | GPU |Disks|
|:-----:|:-----:|:-----:|:-----:|:-----:|:-------:|
|TARS|ROS (Ubuntu 16.04/18.04) + AirSim (Windows 10)|Xeon (?) (12 physical cores)| 4x 8 GB G-Skill DDR4 | 1x 12 GB GTX Titan X Maxwell | 1) 500 GB Cruical NVMe SSD (18.04)<br> 2) 500 GB Crucial SSD (Windows 10) <br> 3) 1 TB Seagate HDD (Storage) <br> 4) 480 GB Kingston SSD (18.04) <br> 5) 250 GB WesternDigital SSD (16.04) <br> 6) 3 TB WesternDigital HDD (Storage)|
|Simulator PC|ROS + Simulators (Ubuntu 18.04)|Ryzen 7 2700x (8 physical cores)|4x 16 GB Corsair DDR4 | 1x 8 GB GTX 1070 Ti Pascal | 1) 512 GB Adata NVMe SSD (OS) <br> 2) 1 TB Seagate HDD (Scratch) <br>3) 1 TB Crucial SSD (Scratch)

## Laptops
TODO
