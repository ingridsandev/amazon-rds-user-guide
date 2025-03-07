# Multi\-AZ deployments for Amazon RDS for Microsoft SQL Server<a name="USER_SQLServerMultiAZ"></a>

Multi\-AZ deployments provide increased availability, data durability, and fault tolerance for DB instances\. In the event of planned database maintenance or unplanned service disruption, Amazon RDS automatically fails over to the up\-to\-date secondary DB instance\. This functionality lets database operations resume quickly without manual intervention\. The primary and standby instances use the same endpoint, whose physical network address transitions to the secondary replica as part of the failover process\. You don't have to reconfigure your application when a failover occurs\.

Amazon RDS supports Multi\-AZ deployments for Microsoft SQL Server by using either SQL Server Database Mirroring \(DBM\) or Always On Availability Groups \(AGs\)\. Amazon RDS monitors and maintains the health of your Multi\-AZ deployment\. If problems occur, RDS automatically repairs unhealthy DB instances, reestablishes synchronization, and initiates failovers\. Failover only occurs if the standby and primary are fully in sync\. You don't have to manage anything\.

When you set up SQL Server Multi\-AZ, RDS automatically configures all databases on the instance to use DBM or AGs\. Amazon RDS handles the primary, the witness, and the secondary DB instance for you\. Because configuration is automatic, RDS selects DBM or Always On AGs based on the version of SQL Server that you deploy\.

Amazon RDS supports Multi\-AZ with Always On AGs for the following SQL Server versions and editions:
+ SQL Server 2019:
  + Standard Edition 15\.00\.4073\.23 and higher
  + Enterprise Edition
+ SQL Server 2017:
  + Standard Edition 14\.00\.3401\.7 and higher
  + Enterprise Edition 14\.00\.3049\.1 and higher
+ SQL Server 2016: Enterprise Edition 13\.00\.5216\.0 and higher

Amazon RDS supports Multi\-AZ with DBM for the following SQL Server versions and editions, except for the versions noted previously:
+ SQL Server 2019: Standard Edition 15\.00\.4043\.16
+ SQL Server 2017: Standard and Enterprise Editions
+ SQL Server 2016: Standard and Enterprise Editions 
+ SQL Server 2014: Standard and Enterprise Editions
+ SQL Server 2012: Standard and Enterprise Editions

You can use the following SQL query to determine whether your SQL Server DB instance is Single\-AZ, Multi\-AZ with DBM, or Multi\-AZ with Always On AGs:

```
SELECT CASE WHEN dm.mirroring_state_desc IS NOT NULL THEN 'Multi-AZ (Mirroring)'
    WHEN dhdrs.group_database_id IS NOT NULL THEN 'Multi-AZ (AlwaysOn)'
    ELSE 'Single-AZ'
    END 'high_availability'
FROM sys.databases sd
LEFT JOIN sys.database_mirroring dm ON sd.database_id = dm.database_id
LEFT JOIN sys.dm_hadr_database_replica_states dhdrs ON sd.database_id = dhdrs.database_id AND dhdrs.is_local = 1
WHERE DB_NAME(sd.database_id) = 'rdsadmin';
```

The output resembles the following:

```
high_availability
Multi-AZ (AlwaysOn)
```

## Adding Multi\-AZ to a Microsoft SQL Server DB instance<a name="USER_SQLServerMultiAZ.Adding"></a>

When you create a new SQL Server DB instance using the AWS Management Console, you can add Multi\-AZ with Database Mirroring \(DBM\) or Always On AGs\. You do so by choosing **Yes \(Mirroring / Always On\)** from **Multi\-AZ deployment**\. For more information, see [Creating an Amazon RDS DB instance](USER_CreateDBInstance.md)\.

When you modify an existing SQL Server DB instance using the AWS Management Console, you can add Multi\-AZ with DBM or AGs by choosing **Yes \(Mirroring / Always On\)** from **Multi\-AZ deployment** on the **Modify DB instance** page\. For more information, see [Modifying an Amazon RDS DB instance](Overview.DBInstance.Modifying.md)\.

**Note**  
If your DB instance is running Database Mirroring \(DBM\)—not Always On Availability Groups \(AGs\)—you might need to disable in\-memory optimization before you add Multi\-AZ\. Disable in\-memory optimization with DBM before you add Multi\-AZ if your DB instance runs SQL Server 2014, 2016, or 2017 Enterprise Edition and has in\-memory optimization enabled\.   
If your DB instance is running AGs, it doesn't require this step\. 

## Microsoft SQL Server Multi\-AZ deployment limitations, notes, and recommendations<a name="USER_SQLServerMultiAZ.Recommendations"></a>

The following are some limitations when working with Multi\-AZ deployments on RDS for Microsoft SQL Server DB instances:
+ Cross\-Region Multi\-AZ isn't supported\.
+ You can't configure the secondary DB instance to accept database read activity\.
+ Multi\-AZ with Always On Availability Groups \(AGs\) supports in\-memory optimization\.
+ Multi\-AZ with Always On Availability Groups \(AGs\) doesn't support Kerberos authentication for the availability group listener\. This is because the listener has no Service Principal Name \(SPN\)\.
+ You can't rename a database on a SQL Server DB instance that is in a SQL Server Multi\-AZ deployment\. If you need to rename a database on such an instance, first turn off Multi\-AZ for the DB instance, then rename the database\. Finally, turn Multi\-AZ back on for the DB instance\. 
+ You can only restore Multi\-AZ DB instances that are backed up using the full recovery model\.
+ Multi\-AZ deployments have a limit of 100 SQL Server Agent jobs\.

  If you need a higher limit, request an increase by contacting AWS Support\. Open the [AWS Support Center](https://console.aws.amazon.com/support/home#/) page, sign in if necessary, and choose **Create case**\. Choose **Service limit increase**\. Complete and submit the form\.

The following are some notes about working with Multi\-AZ deployments on RDS for Microsoft SQL Server DB instances:
+ Amazon RDS exposes the Always On AGs [availability group listener endpoint](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover)\. The endpoint is visible in the console, and is returned by the `DescribeDBInstances` API as an entry in the endpoints field\.
+ Amazon RDS supports [availability group multisubnet failovers](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover)\.
+ To use SQL Server Multi\-AZ with a SQL Server DB instance in a VPC, first create a DB subnet group that has subnets in at least two distinct Availability Zones\. Then assign the DB subnet group to the primary replica of the SQL Server DB instance\. 
+ When a DB instance is modified to be a Multi\-AZ deployment, during the modification it has a status of **modifying**\. Amazon RDS creates the standby, and makes a backup of the primary DB instance\. After the process is complete, the status of the primary DB instance becomes **available**\.
+ Multi\-AZ deployments maintain all databases on the same node\. If a database on the primary host fails over, all your SQL Server databases fail over as one atomic unit to your standby host\. Amazon RDS provisions a new healthy host, and replaces the unhealthy host\.
+ Multi\-AZ with DBM or AGs supports a single standby replica\.
+ Users, logins, and permissions are automatically replicated for you on the secondary\. You don't need to recreate them\. User\-defined server roles \(a SQL Server 2012 feature\) are only replicated in Multi\-AZ instances for AGs instances\.
+ In Multi\-AZ deployments, SQL Server Agent jobs are replicated to the secondary\.
+ You might observe elevated latencies compared to a standard DB instance deployment \(in a single Availability Zone\) because of the synchronous data replication\.
+ Failover times are affected by the time it takes to complete the recovery process\. Large transactions increase the failover time\.
+ In SQL Server Multi\-AZ deployments, reboot with failover reboots only the primary DB instance\. After the failover, the primary DB instance becomes the new secondary DB instance\. Parameters might not be updated for Multi\-AZ instances\. For reboot without failover, both the primary and secondary DB instances reboot, and parameters are updated after the reboot\. If the DB instance is unresponsive, we recommend reboot without failover\.

The following are some recommendations for working with Multi\-AZ deployments on RDS for Microsoft SQL Server DB instances:
+ For databases used in production or preproduction, we recommend the following options:
  + Multi\-AZ deployments for high availability
  + "Provisioned IOPS" for fast, consistent performance
  + "Memory optimized" rather than "General purpose"
+ You can't select the Availability Zone \(AZ\) for the secondary instance, so when you deploy application hosts, take this into account\. Your database might fail over to another AZ, and the application hosts might not be in the same AZ as the database\. For this reason, we recommend that you balance your application hosts across all AZs in the given AWS Region\.
+ For best performance, don't enable Database Mirroring or Always On AGs during a large data load operation\. If you want your data load to be as fast as possible, finish loading data before you convert your DB instance to a Multi\-AZ deployment\. 
+ Applications that access the SQL Server databases should have exception handling that catches connection errors\. The following code sample shows a try/catch block that catches a communication error\. In this example, the `break` statement exits the `while` loop if the connection is successful, but retries up to 10 times if an exception is thrown\.

  ```
  int RetryMaxAttempts = 10;
  int RetryIntervalPeriodInSeconds = 1;
  int iRetryCount = 0;
  while (iRetryCount < RetryMaxAttempts)
  {
     using (SqlConnection connection = new SqlConnection(DatabaseConnString))
     {
        using (SqlCommand command = connection.CreateCommand())
        {
           command.CommandText = "INSERT INTO SOME_TABLE VALUES ('SomeValue');";
           try
           {
              connection.Open();
              command.ExecuteNonQuery();
              break;
           }
           catch (Exception ex) 
           {
              Logger(ex.Message);
              iRetryCount++;
           }
           finally {
              connection.Close();
           }
        }
     }
     Thread.Sleep(RetryIntervalPeriodInSeconds * 1000);
  }
  ```
+ Don't use the `Set Partner Off` command when working with Multi\-AZ instances\. For example, don't do the following\. 

  ```
  --Don't do this
  ALTER DATABASE db1 SET PARTNER off
  ```
+ Don't set the recovery mode to `simple`\. For example, don't do the following\. 

  ```
  --Don't do this
  ALTER DATABASE db1 SET RECOVERY simple
  ```
+ Don't use the `DEFAULT_DATABASE` parameter when creating new logins on Multi\-AZ DB instances, because these settings can't be applied to the standby mirror\. For example, don't do the following\. 

  ```
  --Don't do this
  CREATE LOGIN [test_dba] WITH PASSWORD=foo, DEFAULT_DATABASE=[db2]
  ```

  Also, don't do the following\.

  ```
  --Don't do this
  ALTER LOGIN [test_dba] SET DEFAULT_DATABASE=[db3]
  ```

## Determining the location of the secondary<a name="USER_SQLServerMultiAZ.Location"></a>

You can determine the location of the secondary replica by using the AWS Management Console\. You need to know the location of the secondary if you are setting up your primary DB instance in a VPC\. 

![\[Secondary AZ\]](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/SQLSvr-MultiAZ.png)

You can also view the Availability Zone of the secondary using the AWS CLI command `describe-db-instances` or RDS API operation `DescribeDBInstances`\. The output shows the secondary AZ where the standby mirror is located\. 

## Migrating from Database Mirroring to Always On Availability Groups<a name="USER_SQLServerMultiAZ.Migration"></a>

In version 14\.00\.3049\.1 of Microsoft SQL Server Enterprise Edition, Always On Availability Groups \(AGs\) are enabled by default\.

To migrate from Database Mirroring \(DBM\) to AGs, first check your version\. If you are using a DB instance with a version prior to Enterprise Edition 13\.00\.5216\.0, modify the instance to patch it to 13\.00\.5216\.0 or later\. If you are using a DB instance with a version prior to Enterprise Edition 14\.00\.3049\.1, modify the instance to patch it to 14\.00\.3049\.1 or later\.

If you want to upgrade a mirrored DB instance to use AGs, run the upgrade first, modify the instance to remove Multi\-AZ, and then modify it again to add Multi\-AZ\. This converts your instance to use Always On AGs\.