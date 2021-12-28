# Replacing-SAN-in-SQL-2019-Cluster
How to replace Storage Area Network (SAN) in SQL 2019 Cluster Environment


## Videos


### SQL-2019 Active - Passive Cluster Environement - With SAN Replacement
<img src="ActiveActiveSQLCluster.gif" alt="SQL Cluster">

# Intruduction
In last few videos we had seen how to confirgure Active - Active SQL 2019 cluster. In this video series, we will see what if we have to replace a [SAN](https://www.snia.org/education/storage_networking_primer/san/what_san) device? Most of the SANs available had to replace in 4-5 years. Since our SQL Cluster is dependant on these SAN devices, if there is a change in SAN storage, there will be a change in SQL Cluster as well.
In this video series, we will see how to analyze and metigate the impact, if we had to replace the SAN device in our data center.

## What's needed to replicate this
#### SQL 2019 & Win 2019 
  - To form Win 2019 Cluster and after that SQL 2019 cluster, you would need corresponding softwares. You can find those details in my prior videos.
#### You would need 3 VMS
  - As shown in Image, 1st VM will act as Domain Controller as well as SAN. Since we can't afford to have a SAN, we have to utilize our VM as SAN.
  - 2 VMs for failover. Note, we have to confirgure Active / Passive SQL 2019 cluster - unlike prior Active / Active cluster

## Creating Shared Disks on Domain Controller
   1. Open Server Manager
   2. Select "File & Storage Services"
   3. Select "iSCSI"
   4. Tasks --> "New iSCSI Virtual Disk"
   5. Specify Disk Name
      - quorum2 (1.5 GB)
      - data02 (5 GB)
      - log02 (4 GB)
   6. Specify Disk Size
   7. Add to an existing iSCSI target if not available new
   8. Repeat process for all 3 drives which needs to be added

## Configuring Shared Drives on gogate-node-1 & gogate-node-2
  1. Open "iSCSI initiator"
  2. Specify target - 10.0.0.10 if not already connected. In our case its already connected.
  3. Click on Volumes & Devices --> Auto Configure
  4. To format disk - Open "Disk Management"
  5. Select disks and make online (Right click on each disk)
  6. Once its online -- right click --> Initialize Disk
  7. Select "GPT - GUID Partition Table" --> 
  8. Click on Drive (once its online) & click "New Simple Volume"
  9. Specify Drive Letter
     - X for Quorum
     - Y for Data02
     - Z for Log02
  10. Specify following settings
    - File System - NTFS
     - Allocation Unit Size -- 64 KB (https://blog.sqlserveronline.com/2017/12/08/sql-server-64kb-allocation-unit-size/)
     - Volume Label
     - Select "Perform Quick Format"
     - Finish
  11. Repeat same for all available drives
  12. Once node-1 configuration is finished, make all drives offline
  13. Open "iSCSI Initiator" on 2nd node
  14. Specify target - 10.0.0.10 (If its not connected already, in our case its already connected)
  15. Click on Volumes & Devices --> Auto Configure
  16. Open "Disk Management"
  17. Select disks and make online (Right click on each disk)
  18. Right click on each drive --> Select "Change Drive Letter" & match it exactly what we had given for Node1

## Removing depedancy from SAN1 to SAN2
  There are following different components of SQL2019 Cluster which are dependant on SAN
  1. MDF & LDF files of User databases
  2. MDF & LDF files of System databases (excluding master database)
  3. MDF & LDF files of master database
  4. SQL Log file path
  5. SQLAgent.out log file
  6. Quorum drive
  7. Old drive dependancies
  
