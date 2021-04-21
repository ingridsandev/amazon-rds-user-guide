# Using extended events with Amazon RDS for Microsoft SQL Server<a name="SQLServer.ExtendedEvents"></a>

You can use extended events in Microsoft SQL Server to capture debugging and troubleshooting information for Amazon RDS for SQL Server\. Extended events replace SQL Trace and Server Profiler, which have been deprecated by Microsoft\. Extended events are similar to profiler traces but with more granular control on the events being traced\. Extended events are supported for SQL Server versions 2012 and later on Amazon RDS\. For more information, see [Extended events overview](https://docs.microsoft.com/en-us/sql/relational-databases/extended-events/extended-events) in the Microsoft documentation\.

Extended events are turned on automatically for users with master user privileges in Amazon RDS for SQL Server\.

**Topics**
+ [Limitations and recommendations](#SQLServer.ExtendedEvents.Limits)
+ [Configuring extended events on RDS for SQL Server](#SQLServer.ExtendedEvents.Config)
+ [Considerations for Multi\-AZ deployments](#SQLServer.ExtendedEvents.MAZ)
+ [Querying extended event files](#SQLServer.ExtendedEvents.Querying)

## Limitations and recommendations<a name="SQLServer.ExtendedEvents.Limits"></a>

When using extended events on RDS for SQL Server, the following limitations apply:
+ Extended events are supported only for the Enterprise and Standard Editions\.
+ You can't alter default extended event sessions\.
+ Make sure to set the session memory partition mode to `NONE`\.
+ Session event retention mode can be either `ALLOW_SINGLE_EVENT_LOSS` or `ALLOW_MULTIPLE_EVENT_LOSS`\.
+ Event Tracing for Windows \(ETW\) targets aren't supported\.
+ Make sure that file targets are in the `D:\rdsdbdata\log` directory\.
+ For pair matching targets, set the `respond_to_memory_pressure` property to `1`\.
+ Ring buffer target memory can't be greater than 4 MB\.
+ The following actions aren't supported:
  + `debug_break`
  + `create_dump_all_threads`
  + `create_dump_single_threads`

## Configuring extended events on RDS for SQL Server<a name="SQLServer.ExtendedEvents.Config"></a>

On RDS for SQL Server, you can configure the values of certain parameters of extended event sessions\. The following table describes the configurable parameters\.


| Parameter name | Description | RDS default value | Minimum value | Maximum value | 
| --- | --- | --- | --- | --- | 
| xe\_session\_max\_memory | Specifies the maximum amount of memory to allocate to the session for event buffering\. This value corresponds to the max\_memory setting of the event session\. | 4 MB | 4 MB | 8 MB | 
| xe\_session\_max\_event\_size | Specifies the maximum memory size allowed for large events\. This value corresponds to the max\_event\_size setting of the event session\. | 4 MB | 4 MB | 8 MB | 
| xe\_session\_max\_dispatch\_latency | Specifies the amount of time that events are buffered in memory before being dispatched to extended event session targets\. This value corresponds to the max\_dispatch\_latency setting of the event session\. | 30 seconds | 1 second | 30 seconds | 
| xe\_file\_target\_size | Specifies the maximum size of the file target\. This value corresponds to the max\_file\_size setting of the file target\. | 100 MB | 10 MB | 1 GB | 
| xe\_file\_retention | Specifies the retention time in days for files generated by the file targets of event sessions\. | 7 days | 0 days | 7 days | 

**Note**  
Setting `xe_file_retention` to zero causes \.xel files to be removed automatically after the lock on these files is released by SQL Server\. The lock is released whenever an \.xel file reaches the size limit set in `xe_file_target_size`\.

You can use the `rdsadmin.dbo.rds_show_configuration` stored procedure to show the current values of these parameters\. For example, use the following SQL statement to view the current setting of `xe_session_max_memory`\.

```
exec rdsadmin..rds_show_configuration 'xe_session_max_memory'
```

You can use the `rdsadmin.dbo.rds_set_configuration` stored procedure to modify them\. For example, use the following SQL statement to set `xe_session_max_memory` to 4 MB\.

```
exec rdsadmin..rds_set_configuration 'xe_session_max_memory', 4
```

## Considerations for Multi\-AZ deployments<a name="SQLServer.ExtendedEvents.MAZ"></a>

When you create an extended event session on a primary DB instance, it doesn't propagate to the standby replica\. You can fail over and create the extended event session on the new primary DB instance\. Or you can remove and then readd the Multi\-AZ configuration to propagate the extended event session to the standby replica\. RDS stops all nondefault extended event sessions on the standby replica, so that these sessions don't consume resources on the standby\. Because of this, after a standby replica becomes the primary DB instance, make sure to manually start the extended event sessions on the new primary\.

**Note**  
This approach applies to both Always On Availability Groups and Database Mirroring\.

You can also use a SQL Server Agent job to track the standby replica and start the sessions if the standby becomes the primary\. For example, use the following query in your SQL Server Agent job step to restart event sessions on a primary DB instance\.

```
BEGIN
    IF (DATABASEPROPERTYEX('rdsadmin','Updateability')='READ_WRITE'
    AND DATABASEPROPERTYEX('rdsadmin','status')='ONLINE'
    AND (DATABASEPROPERTYEX('rdsadmin','Collation') IS NOT NULL OR DATABASEPROPERTYEX('rdsadmin','IsAutoClose')=1)
    )
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM sys.dm_xe_sessions WHERE name='xe1')
            ALTER EVENT SESSION xe1 ON SERVER STATE=START
        IF NOT EXISTS (SELECT 1 FROM sys.dm_xe_sessions WHERE name='xe2')
            ALTER EVENT SESSION xe2 ON SERVER STATE=START
    END
END
```

This query restarts the event sessions `xe1` and `xe2` on a primary DB instance if these sessions are in a stopped state\. You can also add a schedule with a convenient interval to this query\.

## Querying extended event files<a name="SQLServer.ExtendedEvents.Querying"></a>

You can either use SQL Server Management Studio or the `sys.fn_xe_file_target_read_file` function to view data from extended events that use file targets\. For more information on this function, see [sys\.fn\_xe\_file\_target\_read\_file \(Transact\-SQL\)](https://docs.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-xe-file-target-read-file-transact-sql) in the Microsoft documentation\.

Extended event file targets can only write files to the `D:\rdsdbdata\log` directory on RDS for SQL Server\.

As an example, use the following SQL query to list the contents of all files of extended event sessions whose names start with `xe`\.

```
SELECT * FROM sys.fn_xe_file_target_read_file('d:\rdsdbdata\log\xe*', null,null,null);
```