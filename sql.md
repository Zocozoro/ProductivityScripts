SQL to create an Extended Events session. Just change 'AppName' below to the Application Name in the connection string and this will capture basically all the SQL that you care about.

```
CREATE EVENT SESSION [AppName] ON SERVER 
ADD EVENT sqlserver.rpc_completed(
    ACTION(package0.collect_cpu_cycle_time,package0.event_sequence,sqlserver.client_app_name,sqlserver.client_pid,sqlserver.database_id,sqlserver.database_name,sqlserver.nt_username,sqlserver.query_hash,sqlserver.server_principal_name,sqlserver.session_id,sqlserver.sql_text,sqlserver.transaction_id)
    WHERE ([package0].[equal_boolean]([sqlserver].[is_system],(0)) AND [sqlserver].[client_app_name]=N'AppName')),
ADD EVENT sqlserver.sql_batch_completed(
    ACTION(package0.collect_cpu_cycle_time,package0.event_sequence,sqlserver.client_app_name,sqlserver.client_pid,sqlserver.database_id,sqlserver.database_name,sqlserver.nt_username,sqlserver.query_hash,sqlserver.server_principal_name,sqlserver.session_id,sqlserver.sql_text,sqlserver.transaction_id)
    WHERE ([package0].[equal_boolean]([sqlserver].[is_system],(0)) AND [sqlserver].[client_app_name]=N'AppName')),
ADD EVENT sqlserver.sql_batch_starting(
    ACTION(package0.collect_cpu_cycle_time,package0.event_sequence,sqlserver.client_app_name,sqlserver.client_pid,sqlserver.database_id,sqlserver.database_name,sqlserver.nt_username,sqlserver.query_hash,sqlserver.server_principal_name,sqlserver.session_id,sqlserver.sql_text,sqlserver.transaction_id)
    WHERE ([package0].[equal_boolean]([sqlserver].[is_system],(0)) AND [sqlserver].[client_app_name]=N'AppName'))
WITH (MAX_MEMORY=819200 KB,EVENT_RETENTION_MODE=NO_EVENT_LOSS,MAX_DISPATCH_LATENCY=5 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=PER_CPU,TRACK_CAUSALITY=ON,STARTUP_STATE=OFF)
GO
```

SQL statement to get missing indexes
```
DECLARE @OnlyCheckCurrentDatabase BIT = 1;

SELECT 
    [DatabaseName] = DB_NAME(mid.[database_id]),
    [SchemaName] = OBJECT_SCHEMA_NAME(mid.[object_id], mid.[database_id]),
    [ObjectName] = OBJECT_NAME(mid.[object_id], mid.[database_id]),
    migs.[avg_user_impact],
    mid.[equality_columns],
    mid.[inequality_columns],
    mid.[included_columns],
    [CreateIndexSQL] = 
		CONCAT('CREATE NONCLUSTERED INDEX [IX_', 
        OBJECT_NAME(mid.[object_id], mid.[database_id]), '_', 
        REPLACE(REPLACE(REPLACE(ISNULL(mid.[equality_columns], '') + 
                CASE 
                    WHEN mid.[inequality_columns] IS NOT NULL AND mid.[equality_columns] IS NOT NULL THEN '_' 
                    WHEN mid.[inequality_columns] IS NOT NULL THEN ''
                    ELSE '' 
                END + ISNULL(mid.[inequality_columns], ''), ', ', '_'), '[', ''), ']', ''),
        '] ON [', 
        OBJECT_SCHEMA_NAME(mid.[object_id], mid.[database_id]), '].[', 
        OBJECT_NAME(mid.[object_id], mid.[database_id]), '] (', 
        ISNULL(mid.[equality_columns], '') + 
        CASE 
            WHEN mid.[inequality_columns] IS NOT NULL AND mid.[equality_columns] IS NOT NULL THEN ', ' 
            ELSE '' 
        END + ISNULL(mid.[inequality_columns], ''), 
        ')',
        CASE 
            WHEN mid.[included_columns] IS NOT NULL THEN ' INCLUDE (' + mid.[included_columns] + ')' 
            ELSE '' 
        END)
FROM 
    sys.[dm_db_missing_index_groups] mig
    JOIN sys.[dm_db_missing_index_group_stats] migs ON migs.[group_handle] = mig.[index_group_handle]
    JOIN sys.[dm_db_missing_index_details] mid ON mig.[index_handle] = mid.[index_handle]
WHERE
    @OnlyCheckCurrentDatabase = 0
    OR
    DB_NAME(mid.[database_id]) = DB_NAME()
ORDER BY
    migs.[avg_user_impact] DESC;

```
