---
# required metadata

title: Export copies of Finance and Operations databases to restore later
description: This topic explains how to export a Microsoft Dynamics 365 for Finance and Operations database to a file, and then reimport that file into the same instance or another instance of the application.
author: LaneSwenka
manager: AnnBe
ms.date: 10/29/2018

ms.topic: article
ms.prod: 
ms.service: dynamics-ax-platform
ms.technology: 

# optional metadata

# ms.search.form: 
# ROBOTS: 
audience: Developer, IT Pro
# ms.devlang: 
ms.reviewer: margoc
ms.search.scope: Operations
# ms.tgt_pltfrm: 
ms.custom: 269254
ms.assetid: 7464b180-8ace-4b65-8b53-ba608d0611e1
ms.search.region: Global
# ms.search.industry: 
ms.author: LaneSwenka
ms.search.validFrom: 2016-11-30
ms.dyn365.ops.version: Platform update 3

---

# Export copies of Finance and Operations databases to restore later

[!include [banner](../includes/banner.md)]

GB22052019 - This topic explains how to export a Microsoft Dynamics 365 for Finance and Operations database to a file, and then reimport that file into the same instance or another instance of the application. This procedure can be used only in non-production environments.

> [!NOTE]
> This topic applies to Microsoft Azure SQL databases that are connected to sandbox user acceptance testing (UAT) environments.

There are several situations where you might want to keep a copy of a Finance and Operations database process:

- You want to make strategic backups that you can restore to later. For example, before or after a major code update, you might want copies that you can use for reference later.
- You want to back up a database before destructive testing and then restore it after the testing is completed.
- When you upgrade to a new major release of Finance and Operations, you can use this process to export your old test database and bring it forward to the new version.

Be aware that Microsoft also provides a standard feature that lets you restore an Azure SQL database environment to a specific point in time within the last 35 days. This restore is done via a service request. For more information, see [Request a point-in-time database restore on a non-production environment](request-point-in-time-restore.md).

> [!IMPORTANT]
> This topic documents the only supported method of retaining a copy of a Finance and Operations database. In a Finance and Operations environment, no copies of Azure SQL database may be kept running. Therefore, use of the CREATE DATABASE AS COPY OF statement is disallowed. Any unsupported copies of Azure SQL databases older than 7 days will be deleted without warning.

## Prerequisites
To export a database from a sandbox environment, you must install the latest version of Microsoft SQL Server Management Studio for Microsoft SQL Server 2016 on the computer that runs Application Object Server (AOS) in that environment. You must then do the export on that AOS computer. There are two reasons for this requirement:

- Because of an Internet Protocol (IP) access restriction on the sandbox instance of Microsoft SQL Server, connections are allowed only from a computer in that environment.
- The version of Management Studio that is installed by default is for a previous version of SQL Server and can't perform the required tasks.

## Export the Finance and Operations database

### Stop services

Use Remote Desktop to connect to all the computers in the environment, and stop the following Microsoft Windows services by using services.msc. These services will have open connections to the Finance and Operations database.

- World wide web publishing service (on all AOS computers)
- Microsoft Dynamics 365 for Finance and Operations Batch Management Service (on non-private AOS computers only)
- Management Reporter 2012 Process Service (on business intelligence \[BI\] computers only)

### Run sqlpackage to export the Finance and Operations database

Open a **Command Prompt** window as an administrator, and run the following commands.

```
cd C:\Program Files (x86)\Microsoft SQL Server\130\DAC\bin

SqlPackage.exe /a:export /ssn:<server>.database.windows.net /sdn:<database to export> /tf:D:\Exportedbacpac\my.bacpac /p:CommandTimeout=1200 /p:VerifyFullTextDocumentTypesSupported=false /sp:<SQL password> /su:<SQL user>
```

Here is an explanation of the parameters:

- **ssn (source server name)** – The name of the Azure SQL Database server to export from.
- **sdn (source database name)** – The name of the database to export.
- **tf (target file)** – The path and name of the file to export to.
- **sp (source password)** – The SQL password for the source SQL Server.
- **su (source user)** – The SQL user name for the source SQL Server. We recommend that you use the **sqladmin** user. This user is created on every SQL instance during deployment. You can retrieve the password for this user from your project in Microsoft Dynamics Lifecycle Services (LCS).

The command creates a .bacpac file in the D:\\Exportedbacpac folder. By copying or uploading this file to secure location, you can import it into another environment later. You can use the AzCopy command-line utility to upload the file to an Azure storage account and then download it to the target AOS computer. For more information, see [Copy or upload the file to an Azure storage account](/azure/storage/storage-use-azcopy).

> [!NOTE]
> Microsoft doesn't provide a storage account as part of your Finance and Operations agreement. You must either purchase a storage account or use a storage account from a separate Azure subscription.

> [!IMPORTANT]
> Be aware of the behavior of drive D on Azure virtual machines (VMs). Don't permanently store your exported database files on this drive. Otherwise, you might lose them. For more information, see the [Understanding the temporary drive on Windows Azure virtual machines](https://blogs.msdn.microsoft.com/mast/2013/12/06/understanding-the-temporary-drive-on-windows-azure-virtual-machines/) blog post.

### Start services

Use services.msc to restart the services that you stopped earlier:

- World wide web publishing service (on all AOS computers)
- Microsoft Dynamics 365 for Finance and Operations Batch Management Service (on non-private AOS computers only)
- Management Reporter 2012 Process Service (on BI computers only)

## Import the Finance and Operations database

### Stop services

Use Remote Desktop to connect to all the computers in the environment, and stop the following Windows services by using services.msc. These services will have open connections to the Finance and Operations database.

- World wide web publishing service (on all AOS computers)
- Microsoft Dynamics 365 for Finance and Operations Batch Management Service (on non-private AOS computers only)
- Management Reporter 2012 Process Service (on BI computers only)

### Import the .bacpac file

Copy the .bacpac file that was generated during the export step to the AOS computer in the target environment. For performance reasons, we recommend that you put the .bacpac file on drive D on the AOS computer.

Open a **Command Prompt** window as an administrator, and run the following commands.

```
cd C:\Program Files (x86)\Microsoft SQL Server\130\DAC\bin\

SqlPackage.exe /a:import /sf:D:\Exportedbacpac\my.bacpac /tsn:<Azure DSQL database server name>.database.windows.net /tu:sqladmin /tp:<password from LCS> /tdn:<new database name> /p:CommandTimeout=1200 /p:DatabaseEdition=Premium /p:DatabaseServiceObjective=<See below>
```

Here is an explanation of the parameters:

- **tsn (target server name)** – The name of the Azure SQL Database server to import into. You can find the name in LCS on the environment page under Database Accounts. Add the suffix **database.windows.net** to it.
- **tdn (target database name)** – The name of the database to import into. The database should **not** already exist. The import process will create it.
- **sf (source file)** – The path and name of the file to import from.
- **tu (target user)** – The SQL user name for the target Azure SQL database instance. We recommend that you use the standard **sqladmin** user. You can retrieve the password for this user from your LCS project.
- **tp (target password)** – The password for the target Azure SQL database user.
- **DatabaseServiceObjective** – Specifies the performance level of the database such as S1, P2 or P4. To meet performance requirements and comply with your service agreement, use the same service objective level as the current Finance and Operations database (AXDB) on this environment. To query the service level objective of the current database, run the following query.

    ```
    SELECT  d.name,   
        slo.*    
    FROM sys.databases d   
    JOIN sys.database_service_objectives slo    
    ON d.database_id = slo.database_id;  
    ```

### Run a script to update the Finance and Operations database

If the source and target environments have different SQL user passwords, you must run the following script. You must also run this script if you aren't sure whether the passwords differ. Run the script against the imported database. The script drops the database users and then re-creates them so that they have the correct passwords for the target environment.

```
ALTER AUTHORIZATION ON Fulltext Catalog::[<name of the full text catalog in your database] TO [dbo];

declare @userSQL varchar(1000)
set quoted_identifier off

declare userCursor CURSOR for
select 'DROP USER ' + name
from sys.sysusers
where issqlrole = 0 and hasdbaccess = 1 and name <> 'dbo'

OPEN userCursor
FETCH userCursor into @userSQL
WHILE @@Fetch_Status = 0
BEGIN
exec(@userSQL)
FETCH userCursor into @userSQL
END
CLOSE userCursor
DEALLOCATE userCursor

CREATE USER axdeployuser FROM LOGIN axdeployuser
EXEC sp_addrolemember 'db_owner', 'axdeployuser'

CREATE USER axdbadmin WITH PASSWORD = '<password from LCS>'
EXEC sp_addrolemember 'db_owner', 'axdbadmin'

CREATE USER axruntimeuser WITH PASSWORD = '<password from LCS>'
EXEC sp_addrolemember 'db_datareader', 'axruntimeuser'
EXEC sp_addrolemember 'db_datawriter', 'axruntimeuser'

CREATE USER axmrruntimeuser WITH PASSWORD = '<password from LCS>'
EXEC sp_addrolemember 'ReportingIntegrationUser', 'axmrruntimeuser'
EXEC sp_addrolemember 'db_datareader', 'axmrruntimeuser'
EXEC sp_addrolemember 'db_datawriter', 'axmrruntimeuser'

CREATE USER axretailruntimeuser WITH PASSWORD = '<password from LCS>'
EXEC sp_addrolemember 'UsersRole', 'axretailruntimeuser'
EXEC sp_addrolemember 'ReportUsersRole', 'axretailruntimeuser'

CREATE USER axdeployextuser WITH PASSWORD = '<password from LCS>'
EXEC sp_addrolemember 'DeployExtensibilityRole', 'axdeployextuser'

GO
-- Begin Refresh Retail FullText Catalogs
DECLARE @RFTXNAME NVARCHAR(MAX);
DECLARE @RFTXSQL NVARCHAR(MAX);
DECLARE retail_ftx CURSOR FOR
SELECT OBJECT_SCHEMA_NAME(object_id) + '.' + OBJECT_NAME(object_id) fullname FROM SYS.FULLTEXT_INDEXES
	WHERE FULLTEXT_CATALOG_ID = (SELECT TOP 1 FULLTEXT_CATALOG_ID FROM SYS.FULLTEXT_CATALOGS WHERE NAME = 'COMMERCEFULLTEXTCATALOG');
OPEN retail_ftx;
FETCH NEXT FROM retail_ftx INTO @RFTXNAME;

BEGIN TRY
	WHILE @@FETCH_STATUS = 0  
	BEGIN  
		PRINT 'Refreshing Full Text Index ' + @RFTXNAME;
		EXEC SP_FULLTEXT_TABLE @RFTXNAME, 'activate';
		SET @RFTXSQL = 'ALTER FULLTEXT INDEX ON ' + @RFTXNAME + ' START FULL POPULATION';
		EXEC SP_EXECUTESQL @RFTXSQL;
		FETCH NEXT FROM retail_ftx INTO @RFTXNAME;
	END
END TRY
BEGIN CATCH
	PRINT error_message()
END CATCH

CLOSE retail_ftx;  
DEALLOCATE retail_ftx; 
-- End Refresh Retail FullText Catalogs
```
## Synchronize the database

1. Use Remote Desktop to connect to all the computers in the target environment, and stop the following Windows services by using services.msc. These services will have open connections to the Finance and Operations database. After you stop the services, you can replace the existing Finance and Operations database with the newly imported database.

    - World wide web publishing service (on all AOS computers)
    - Microsoft Dynamics 365 for Finance and Operations Batch Management service (on non-private AOS computers only)
    - Management Reporter 2012 Process service (on business intelligence \[BI\] computers only)

2. On the AOS computer where the bacpac import was performed, run the following script in Management Studio. This script renames the original database and then renames the newly imported database so that it uses the original database name. In this example, the original database was named axdb\_123456789, and the newly imported database was named importeddb.

    > [!NOTE]
    > Make sure that you're using the SQL Server 2016 version of Management Studio.

    ```
    ALTER DATABASE [axdb_123456789] MODIFY NAME = [axdb_123456789_original]
    ALTER DATABASE [importeddb] MODIFY NAME = [axdb_123456789]
    ```

3. Synchronize the database. Open a **Command Prompt** window as an administrator, and run the following commands.

    ```
    cd F:\AosService\WebRoot\bin

    Microsoft.Dynamics.AX.Deployment.Setup.exe -bindir "F:\AosService\PackagesLocalDirectory" -metadatadir F:\AosService\PackagesLocalDirectory -sqluser axdbadmin -sqlserver <Azure SQL database server name>.database.windows.net -sqldatabase <database name> -setupmode sync -syncmode fullall -isazuresql true -sqlpwd <SQL password> >log.txt 2>&1
    ```

4. Use services.msc to restart the services that you stopped earlier:

    - World wide web publishing service (on all AOS computers)
    - Microsoft Dynamics 365 for Finance and Operations Batch Management service (on non-private AOS computers only)
    - Management Reporter 2012 Process service (on BI computers only)

5. At this point, you can open the Finance and Operations application URL and sign in. Verify that the application works as you expect. Then drop the original database by running the following script in Management Studio on the AOS computer where you performed the bacpac import.

    ```
    DROP DATABASE [axdb_123456789_original]
    ```


### Re-provision Retail components in the target environment

Reprovisioning is only required if you are restoring or importing the database on another environment.

[!include [environment-reprovision](../includes/environment-reprovision.md)]

## Limitations
After you import a database, the link between the database and document handling documents that are stored in Azure blob storage might be broken. If you have custom code that uses the X++ **FileUpload** class to put files in blob storage, the links to those files might also be broken.
