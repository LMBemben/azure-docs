---
title: Secure a database
description: Tips for securing a database and developing solutions in the SQL pool resource of SQL Analytics.
services: sql-data-warehouse
author: julieMSFT
manager: craigg
ms.service: sql-data-warehouse
ms.topic: conceptual
ms.subservice: security
ms.date: 04/17/2018
ms.author: jrasnick
ms.reviewer: igorstan
ms.custom: seo-lt-2019
tags: azure-synapse
---

# Secure a database in Azure Synapse
> [!div class="op_single_selector"]
> * [Security Overview](sql-data-warehouse-overview-manage-security.md)
> * [Authentication](sql-data-warehouse-authentication.md)
> * [Encryption (Portal)](sql-data-warehouse-encryption-tde.md)
> * [Encryption (T-SQL)](sql-data-warehouse-encryption-tde-tsql.md)
> 
> 

This article will walk you through the basics of securing your SQL pool within SQL Analytics. In particular, this article gets you started with resources for limiting access, protecting data, and monitoring activities on a database provisioned using SQL pool.

## Connection security
Connection Security refers to how you restrict and secure connections to your database using firewall rules and connection encryption.

Firewall rules are used by both the server and the database to reject connection attempts from IP addresses that haven't been explicitly whitelisted. To allow connections from your application or client machine's public IP address, you must first create a server-level firewall rule using the Azure portal, REST API, or PowerShell. 

As a best practice, you should restrict the IP address ranges allowed through your server firewall as much as possible.  To access SQL pool from your local computer, ensure the firewall on your network and local computer allows outgoing communication on TCP port 1433.  

Azure Synapse Analytics uses server-level IP firewall rules. It doesn't support database-level IP firewall rules. For more information, see see [Azure SQL Database firewall rules](../sql-database/sql-database-firewall-configure.md)

Connections to your SQL pool are encrypted by default.  Modifying connection settings to disable encryption are ignored.

## Authentication
Authentication refers to how you prove your identity when connecting to the database. SQL pool currently supports SQL Server Authentication with a username and password, and with Azure Active Directory. 

When you created the logical server for your database, you specified a "server admin" login with a username and password. Using these credentials, you can authenticate to any database on that server as the database owner, or "dbo" through SQL Server Authentication.

However, as a best practice, your organization’s users should use a different account to authenticate. This way you can limit the permissions granted to the application and reduce the risks of malicious activity in case your application code is vulnerable to a SQL injection attack. 

To create a SQL Server Authenticated user, connect to the **master** database on your server with your server admin login and create a new server login.  It's a good idea to also create a user in the master database. Creating a user in master allows a user to log in using tools like SSMS without specifying a database name.  It also allows them to use the object explorer to view all databases on a SQL server.

```sql
-- Connect to master database and create a login
CREATE LOGIN ApplicationLogin WITH PASSWORD = 'Str0ng_password';
CREATE USER ApplicationUser FOR LOGIN ApplicationLogin;
```

Then, connect to your **SQL pool database** with your server admin login and create a database user based on the server login you created.

```sql
-- Connect to the database and create a database user
CREATE USER ApplicationUser FOR LOGIN ApplicationLogin;
```

To give a user permission to perform additional operations such as creating logins or creating new databases, assign the user to the `Loginmanager` and `dbmanager` roles in the master database. 

For more information on these additional roles and authenticating to a SQL Database, see [Managing databases and logins in Azure SQL Database](../sql-database/sql-database-manage-logins.md).  For more information on connecting using Azure Active Directory, see [Connecting by using Azure Active Directory Authentication](sql-data-warehouse-authentication.md).

## Authorization
Authorization refers to what you can do within a database once you are authenticated and connected. Authorization privileges are determined by role memberships and permissions. As a best practice, you should grant users the least privileges necessary. To manage roles, you can use the following stored procedures:

```sql
EXEC sp_addrolemember 'db_datareader', 'ApplicationUser'; -- allows ApplicationUser to read data
EXEC sp_addrolemember 'db_datawriter', 'ApplicationUser'; -- allows ApplicationUser to write data
```

The server admin account you are connecting with is a member of db_owner, which has authority to do anything within the database. Save this account for deploying schema upgrades and other management operations. Use the "ApplicationUser" account with more limited permissions to connect from your application to the database with the least privileges needed by your application.

There are ways to further limit what a user can do within the database:

* Granular [Permissions](https://docs.microsoft.com/sql/relational-databases/security/permissions-database-engine?view=sql-server-ver15) let you control which operations you can do on individual columns, tables, views, schemas, procedures, and other objects in the database. Use granular permissions to have the most control and grant the minimum permissions necessary. 
* [Database roles](https://docs.microsoft.com/sql/relational-databases/security/authentication-access/database-level-roles?view=sql-server-ver15) other than db_datareader and db_datawriter can be used to create more powerful application user accounts or less powerful management accounts. The built-in fixed database roles provide an easy way to grant permissions, but can result in granting more permissions than are necessary.
* [Stored procedures](https://docs.microsoft.com/sql/relational-databases/stored-procedures/stored-procedures-database-engine?redirectedfrom=MSDN&view=sql-server-ver15) can be used to limit the actions that can be taken on the database.

The following example grants read access to a user-defined schema.
```sql
--CREATE SCHEMA Test
GRANT SELECT ON SCHEMA::Test to ApplicationUser
```

Managing databases and logical servers from the Azure portal or using the Azure Resource Manager API is controlled by your portal user account's role assignments. For more information, see [Role-based access control in Azure portal](https://azure.microsoft.com/documentation/articles/role-based-access-control-configure).

## Encryption
Transparent Data Encryption (TDE) helps protect against the threat of malicious activity by encrypting and decrypting your data at rest. When you encrypt your database, associated backups and transaction log files are encrypted without requiring any changes to your applications. TDE encrypts the storage of an entire database by using a symmetric key called the database encryption key. 

In SQL Database, the database encryption key is protected by a built-in server certificate. The built-in server certificate is unique for each SQL Database server. Microsoft automatically rotates these certificates at least every 90 days. The encryption algorithm used is AES-256. For a general description of TDE, see [Transparent Data Encryption](https://docs.microsoft.com/sql/relational-databases/security/encryption/transparent-data-encryption?view=sql-server-ver15).

You can encrypt your database using the [Azure portal](sql-data-warehouse-encryption-tde.md) or [T-SQL](sql-data-warehouse-encryption-tde-tsql.md).

## Next steps
For details and examples on connecting to your warehouse with different protocols, see [Connect to SQL pool](sql-data-warehouse-connect-overview.md).
