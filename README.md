![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Prep For SQL AlwaysOn With This SQL Restore Logic
**Post Date: March 21, 2016**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>The following SQL logic will backup all databases excluding ('master', 'model', 'msdb', 'tempdb', 'reportserver', 'reportservertempdb'), then it will produce 2 restore scripts. Restore Full Database Backups, and Restore Transaction Logs. The restore scripts will restore WITH NORECOVERY leaving the databases ready for the AlwaysOn configurations.
  
Here's what the produced SQL logic will do:
1. Sets the Default Backup Compression before the databases are backed up.
2. Creates a single Full Database backup (for all databases) to: e:\MyFullBackupsFolder\
3. Creates a single Transaction Log backup (for all databases) to: e:\MyLogBackupsFolder\
4. Procues a Database Restore Script in the form of an XML link. Once clicked you'll see the entire restore logic for all database backups.
5. Produces a Transaction Log Restore Script in the form of an XML Link. Once clicked you'll see the entire restore logic for all transaction log backups.

Note: The logic will only backup databases that have an ONLINE status, and ensures the databases that are backed up are not part of a secondary, or set as a 'Mirrors' of an existing Database Mirror scheme. However the databases will be backup Principal databases if they exist. This is quite helpful if you are moving AlwaysOn or Mirrored databases to other servers.
Additionally; it doesn't matter how many data files you have under each database. The logical and physical paths are read directly from the FILELISTONLY results per each backup file so all files should be addressed during the restore. This will also restore the data files in their normal hierarchy where the first data, and first log files are restored, then subsequent .trn's or fulltext data files are hit next. This will avoid any precedence failures that might occur.

Ok so how does this work? What do I need to do for this to run?
First; disable any transaction log backups you are currently running on any regular schedule. If a transaction log backup runs while this process is carried out it will through off the LSN's and the subsequent transaction logs that follow will error out.

You'll need the procedure 'sp_killallprocessindb' which you can find here: http://www.databasejournal.com/scripts/article.php/3634276/Kill-All-Processes-in-a-Particular-DataBase.htm
Be sure to create this under your master database on the destination server, but it's a really good proc to use whatever server you're working on.

All you have to do is create 2 folders to hold the backup files. (both full backups and transaction log backups). In this example I am using the following:
e:\MyFullBackupsFolder
e:\MyLogBackupsFolder

This can be wherever you feel is best as long as you modify the backup logic AND the restore logic to be the appropriate path before running it.

Next simply copy all the the SQL logic below, and paste into Management Studio. Once it runs it will automatically create all the backups you need, and all the restore scripts necessarry. Once this is done; you will need to copy over the backup folders you created formerly (with all the new backups and transaction log backups) to the new server where you want to restore the databases. Then take the produced restore scripts and run them on the new server. The databases will all be restored WITH NORECOVERY so you'll be all set to setup your AVAILABILITY GROUPS for your AlwaysOn configurations and it should synch up no problem.

Note: As a fail-safe measure to make sure no databases are automatically restored before you had a real chance to view the restore logic; you'll see the following line of code after each restore set:
select/**/ — replace this line with: exec
Basically; all you have to do when you're ready to run the logic is do a find and replace.
Replace this: select/**/ — replace this line with: exec
With this: exec

This was placed here to give you the opportunity to see the restore process as it is written for each database.
On with the script…</p>    



## SQL-Logic
```SQL
use master;
set nocount on
 
-- BACKUP ALL DATABASES LOCOALLY.  SINGLE FULL BACKUP AND SINGLE TRANSACTION LOG BACKUP PER EACH DATABASE
 
declare
        @sao    int = (select cast([value] as int) from master.sys.configurations where [name] = 'show advanced options')
,       @bcd    int = (select cast([value] as int) from master.sys.configurations where [name] = 'backup compression default')
,       @xpc    int = (select cast([value] as int) from master.sys.configurations where [name] = 'xp_cmdshell')
declare @backup_all_user_databases      varchar(max)
set     @backup_all_user_databases      = ''
select  @backup_all_user_databases      = @backup_all_user_databases +
'use master;'       + char(10) +
'backup database        [' + upper(sd.name) + '] to disk = ''e:\MyFullBackupsFolder\' + upper(sd.name) + '.bak''    with format;'   + char(10) +
'backup log         [' + upper(sd.name) + '] to disk = ''e:\MyLogBackupsFolder\' + upper(sd.name) + '.trn''     with format;'   + char(10) + char(10)
from
    sys.databases sd join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where   
    name not in ('master', 'model', 'msdb', 'tempdb', 'reportserver', 'reportservertempdb')
    and     state_desc = 'online'
    and     sd.source_database_id   is null
    and     sdm.mirroring_role_desc is null
    or      sdm.mirroring_role_desc != 'mirror'
order by
    name asc
if      @sao = 0    begin exec  master..sp_configure 'show advanced options', 1         reconfigure end
if      @bcd = 0    begin exec  master..sp_configure 'backup compression default', 1    reconfigure end
if      @xpc = 0    begin exec  master..sp_configure 'xp_cmdshell', 1                   reconfigure end
 
exec    (@backup_all_user_databases)
 

-- CREATE RESTORE LOGIC FOR FULL DATABASE BACKUPS.  RUN THIS AT THE DESTINATION SERVER.
 
declare     @create_restore_database_full_logic varchar(max) 
set         @create_restore_database_full_logic = '' 
select      @create_restore_database_full_logic = @create_restore_database_full_logic + '
use master;
set nocount on
go
declare @database_name      varchar (255) 
declare @backup_file_name   varchar (255) 
set     @database_name      = ''' + replace(name, '''', '''''') + '''
set     @backup_file_name   = ''e:\MyFullBackupsFolder\' + replace(name, '''', '') + '.bak''
declare @filelistonly       table
(
    logicalname     nvarchar (128)
,   physicalname    nvarchar (260) 
,   [type]          char (1)
,   filegroupname   nvarchar (128) 
,   size            numeric (20,0) 
,   maxsize         numeric (20,0) 
,   fileid          bigint
,   createlsn       numeric (25,0) 
,   droplsn         numeric (25,0) 
,   uniqueid        uniqueidentifier
,   readonlylsn     numeric (25,0) 
,   readwritelsn    numeric (25,0) 
,   backupsizeinbytes bigint
,   sourceblocksize int
,   filegroupid     int
,   loggroupguid    uniqueidentifier
,   differentialbaselsn     numeric (25,0) 
,   differentialbaseguid    uniqueidentifier
,   isreadonl       bit
,   ispresent       bit
,   tdethumbprint   varbinary (32)
)
insert into
        @filelistonly 
exec    (''restore filelistonly from disk = '''''' + @backup_file_name + '''''''') 
 
 
declare @restore_line0  varchar (255) 
declare @restore_line1  varchar (255) 
declare @restore_line2  varchar (255) 
declare @stats          varchar (255) 
declare @move_files     varchar (max) 
set     @restore_line0  = (''use master; '')
set     @restore_line1  = (''exec master..sp_killallprocessindb '''''' + @database_name + '''''';'')
set     @restore_line2  = (select ''restore database ['' + @database_name + ''] from disk = '''''' + @backup_file_name + '''''' with replace, norecovery, '') 
set     @stats          = (''stats = 20;'')
set     @move_files     = ''''
select  @move_files     = @move_files + ''move '''''' + logicalname + '''''' to '''''' + physicalname + '''''','' + char(10) from @filelistonly order by fileid asc
  
select/**/ -- replace this line with: exec
(
    @restore_line0
+   @restore_line1
+   @restore_line2
+   @move_files
+   @stats
)
go
'
from    sys.databases 
where   name not in ('master', 'model', 'tempdb', 'msdb', 'reportserver', 'reportservertempdb') 
select  (@create_restore_database_full_logic) for xml path (''), type
 
 
 
 
</p>      </p>      </p>      </p>      </p>      </p>      </p>      </p>      </p>      
-- CREATE RESTORE LOGIC FOR TRANSACTION LOG BACKUPS.  RUN THIS AT THE DESTINATION SERVER.
 
 
declare     @create_restore_database_logs_logic varchar(max) 
set         @create_restore_database_logs_logic = '' 
select      @create_restore_database_logs_logic = @create_restore_database_logs_logic + '
use master;
set nocount on
go
declare @database_name      varchar (255) 
declare @backup_file_name   varchar (255) 
set     @database_name      = ''' + replace(name, '''', '''''') + '''
set     @backup_file_name   = ''e:\MyLogBackupsFolder\' + replace(name, '''', '') + '.trn''
declare @filelistonly       table
(
    logicalname     nvarchar (128) 
,   physicalname    nvarchar (260) 
,   [type]          char (1)
,   filegroupname   nvarchar (128) 
,   size            numeric (20,0) 
,   maxsize         numeric (20,0) 
,   fileid          bigint
,   createlsn       numeric (25,0) 
,   droplsn         numeric (25,0) 
,   uniqueid        uniqueidentifier
,   readonlylsn     numeric (25,0) 
,   readwritelsn    numeric (25,0) 
,   backupsizeinbytes bigint
,   sourceblocksize int
,   filegroupid     int
,   loggroupguid    uniqueidentifier
,   differentialbaselsn     numeric (25,0) 
,   differentialbaseguid    uniqueidentifier
,   isreadonl       bit
,   ispresent       bit
,   tdethumbprint   varbinary (32)
)
insert into
        @filelistonly   
exec    (''restore filelistonly from disk = '''''' + @backup_file_name + '''''''')
 
 
declare @restore_line0  varchar (255) 
declare @restore_line1  varchar (255) 
declare @restore_line2  varchar (255) 
declare @stats          varchar (255) 
declare @move_files     varchar (max) 
set     @restore_line0  = (''use master; '')
set     @restore_line1  = (''exec master..sp_killallprocessindb '''''' + @database_name + '''''';'')
set     @restore_line2  = (select ''restore log ['' + @database_name + ''] from disk = '''''' + @backup_file_name + '''''' with norecovery, '') 
set     @stats          = (''stats = 5;'')
  
select/**/ -- replace this line with: exec
(
    @restore_line0
+   @restore_line1
+   @restore_line2
+   @stats
)
go
'
from    sys.databases 
where   name not in ('master', 'model', 'tempdb', 'msdb', 'reportserver', 'reportservertempdb') 
select  (@create_restore_database_logs_logic) for xml path (''), type
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

