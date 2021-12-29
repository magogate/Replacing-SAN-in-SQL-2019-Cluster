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
   5. Change the folder - C:\iSCASI_SAN2
   6. Specify Disk Name
      - quorum2 (1.5 GB)
      - data02 (5 GB)
      - log02 (4 GB)
   7. Specify Disk Size
   8. Add to an existing iSCSI target if not available new
   9. Repeat process for all 3 drives which needs to be added

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

## Adding storage to Windows Failover Cluster - Available Storage
  1. Once disks are configured on both Node1 & Node2, next step is to add them into Win Failover Cluster
  2. Make sure both Node1 and Node2 are running
  3. Go to Failover Cluster Manager --> Storage --> Disk --> click on "Add Disk"
  4. Newly added 3 disks will appear as "Available Storage"
  
## Adding storage to SQL Failover Cluster
  1. Open Failoever Cluster Manager
  2. Click on Roles --> go to Resources tab
  3. Select SQL Server (SQL2019) resource and at right panel click on "Add Storage"
  4. Remaining 3 drives will get added as part of SQL 2019 cluster
  5. Once they are part of Win Failover Cluster, select "SQL Server (SQL2019) resource --> Properties
  6. New window will get open, click on Dependancies tab --> add newly created 3 disks as dependancies

## How to test if disks are part of SQL Cluster
  1. Bring down newly added disk
  2. if SQL Server service goes down, then newly added disk is part of SQL Cluster

## Removing depedancy from SAN1 to SAN2
  There are following different components of SQL2019 Cluster which are dependant on SAN
  1. MDF & LDF files of User databases
  2. MDF & LDF files of System databases (excluding master database)
  3. MDF & LDF files of master database
  4. SQL Log file path
  5. SQLAgent.out log file
  6. Quorum drive
  7. Old drive dependancies
  
### 1. Changing MDF & LDF files of User Databases
  1. Find out logical file name and path of existing database
     ``` 
        SELECT  DatabaseName = d.name
        , DatabaseState = d.state_desc
        , FileName = mf.name
        , FileState = mf.state_desc
        , FilePath = mf.physical_name
        FROM sys.master_files mf 
          INNER JOIN sys.databases d ON mf.database_id = d.database_id
        WHERE 1=1
        ORDER BY d.name
          , mf.name;
     ```
  2. Change path of MDF & LDF file with alter command
      ``` 
        ALTER DATABASE HR
        MODIFY FILE ( NAME = HR,
        FILENAME = 'Y:\MSSQL\DATA\HR.mdf');
        GO
    
        ALTER DATABASE HR
        MODIFY FILE ( NAME = HR_log,
        FILENAME = 'Z:\MSSQL\LOG\HR_log.ldf');
        GO      
      ```
  3. Bring database offline
      ```
        ALTER DATABASE HR SET OFFLINE;
        GO    
      ```
  5. Move the physical files
  6. Grant access to new folder structure
  7. Bring database online
      ```
        ALTER DATABASE HR SET ONLINE;
        GO
      ```  

### 2. Changing MDF & LDF files of System Databases (excluding master)
  Follow exact same steps which we did for User databases, but since we can't take individual databases offline, we have to re-start the service. Note - There will be a downtime due to this activity.


### 3. Changing MDF & LDF files of master database
  1. Find out logical file name and path of existing database
     ``` 
        SELECT  DatabaseName = d.name
        , DatabaseState = d.state_desc
        , FileName = mf.name
        , FileState = mf.state_desc
        , FilePath = mf.physical_name
        FROM sys.master_files mf 
          INNER JOIN sys.databases d ON mf.database_id = d.database_id
        WHERE 1=1
        ORDER BY d.name
          , mf.name;
     ```
  2. Change path of MDF & LDF file with alter command
      ``` 
        ALTER DATABASE master
        MODIFY FILE ( NAME = master,
        FILENAME = 'Y:\MSSQL\DATA\master.mdf');
        GO
    
        ALTER DATABASE master
        MODIFY FILE ( NAME = mastlog,
        FILENAME = 'Z:\MSSQL\LOG\mastlog.ldf');
        GO      
      ```
  3. As we observed for other system databases, even master database we can't bring offline individually. Hence, let's try to stop the service
  4. Move the physical files 
  5. Try to bring the service online? it won't come online and it fails. 
  6. Check the error logs to understand why it fails? How to get the error logs? you will find the path in Startup Parameters of Sql Server Service.
  7. Change the startup parameters<br>
      OLD
      ```
        -dG:\MSSQL15.SQL2019\MSSQL\DATA\master.mdf
        -eG:\MSSQL15.SQL2019\MSSQL\Log\ERRORLOG
        -lG:\MSSQL15.SQL2019\MSSQL\DATA\mastlog.ldf
      ```
      NEW
      ```
        -dY:\MSSQL\DATA\master.mdf
        -eZ:\MSSQL\LOG\ERRORLOG
        -lZ:\MSSQL\LOG\mastlog.ldf
      ```
   8. Change the Default Location of MDF and LDF files

### 4. Removing SQL Server Cluster dependancies from old SAN
  1. Validate if all sql 2019 services are running 
  2. To remove shared drives dependacy from SQL Cluster - Open "Failover Cluster Manager"
  3. Click on Roles --> SQL Server (SQL2019)
  4. At bottom panel select the Shared Disk which you want to take offline
  5. Moment you take the disk offline, SQL Server Serivce will also go down
  6. Right click on "SQL Server (SQL 2019)" under "Other Resources" section
  7. Go to Dependancy tab - and remove disk which are not needed anymore
  8. Now, try to take Shared Disks offline, and all should work.
  9. Once offline, right click on the disk --> Remove it from SQL Server (SQL2019)

### 5. SQLAGENT.OUT
  1. After taking Shared Disks offline, SQL Server service did not go down, rather all services on SQL Cluster were working fine
  2. However, try restarting SQL Server Agent service now. It will fail.
  3. Why? its because we forgot to modify path for SQLAGENT.OUT. Since Shared Disks are offline now, how can we find this path?
  4. There are 2 ways
     - Validate [Registry](https://www.sqlservercentral.com/blogs/working-with-the-registry-from-within-sql-server). You can directly chagne the registry entry and issue wil get fixed
     - Check using SQL Server [Query](https://www.mssqltips.com/sqlservertip/3093/how-to-change-the-sql-server-agent-log-file-path/). Execute the mentioned commands in this blog to fix the issue
