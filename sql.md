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

SQL statement to get size of table as well as the rows in each table
```

DECLARE @ShowDiskSizeByTable BIT = 1;
DECLARE @ShowNumberOfRowsByTable BIT = 1;
DECLARE @ShowNumberOfRowsByTable_ExcludeEmptyTables BIT = 1;

IF (@ShowDiskSizeByTable = 1)
BEGIN
    SELECT 
        CONCAT(s.[Name], '.', t.[NAME]) AS [SchemaTableName],
        SUM(a.[used_pages]) * 8.0 / 1024 AS [TableSizeMB]
    FROM 
        [sys].[tables] t
        JOIN [sys].[indexes] i ON t.[OBJECT_ID] = i.[object_id]
        JOIN [sys].[partitions] p ON i.[object_id] = p.[OBJECT_ID] AND i.[index_id] = p.[index_id]
        JOIN [sys].[allocation_units] a ON p.[partition_id] = a.[container_id]
        JOIN [sys].[schemas] s ON t.[schema_id] = s.[schema_id]
    WHERE   
        t.[NAME] NOT LIKE 'dt%' 
        AND t.[is_ms_shipped] = 0
        AND i.[OBJECT_ID] > 255 
    GROUP BY 
        s.[Name],
        t.[NAME]
    ORDER BY 
        SUM(a.[used_pages]) DESC;
END

IF (@ShowNumberOfRowsByTable = 1)
BEGIN
    DECLARE @TableName NVARCHAR(256), @SchemaName NVARCHAR(256), @SQL NVARCHAR(MAX);
    DECLARE @Results TABLE (SchemaTable NVARCHAR(512), NumberOfRows INT);

    DECLARE TableCursor CURSOR FOR
    SELECT TABLE_SCHEMA, TABLE_NAME
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE' AND TABLE_NAME NOT LIKE 'dt%';

    OPEN TableCursor;
    FETCH NEXT FROM TableCursor INTO @SchemaName, @TableName;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = N'SELECT ''' + @SchemaName + '.' + @TableName + ''' AS SchemaTable, COUNT(*) AS NumberOfRows FROM [' + @SchemaName + '].[' + @TableName + ']';
        INSERT INTO @Results (SchemaTable, NumberOfRows)
        EXEC sp_executesql @SQL;

        FETCH NEXT FROM TableCursor INTO @SchemaName, @TableName;
    END

    CLOSE TableCursor;
    DEALLOCATE TableCursor;

    SELECT 
        * 
    FROM 
        @Results r
    WHERE
        @ShowNumberOfRowsByTable_ExcludeEmptyTables = 0
        OR
        (
            @ShowNumberOfRowsByTable_ExcludeEmptyTables = 1
            AND
            r.[NumberOfRows] > 0
        )
    ORDER BY
        r.[SchemaTable];
END
```

Azure Data Studio User Settings
```
{
    "workbench.enablePreviewFeatures": true,
    "mssql.format.keywordCasing": "uppercase",
    "editor.wordSeparators": "`~!#$%^&*()=+{}\\|;:'\",.<>/?",
    "editor.dragAndDrop": false,
    "editor.emptySelectionClipboard": false,
    "workbench.editorAssociations": {
        "*.{sqlplan}": "workbench.editor.executionplan"
    },
    "editor.fontSize": 16,
    "editor.suggestSelection": "first",
    "vsintellicode.modify.editor.suggestSelection": "automaticallyOverrodeDefaultValue",
    "[sql]": {
        "editor.defaultFormatter": "Microsoft.mssql"
    },
    "editor.acceptSuggestionOnEnter": "off",
    "workbench.editor.wrapTabs": true,
    "editor.suggest.showColors": false,
    "editor.suggest.showCustomcolors": false,
    "editor.suggest.showDeprecated": false,
    "editor.suggest.showFunctions": false,
    "editor.suggest.showFolders": false,
    "editor.suggest.showEvents": false,
    "editor.suggest.showEnums": false,
    "editor.suggest.showEnumMembers": false,
    "editor.suggest.showMethods": false,
    "editor.suggest.showUsers": false,
    "editor.suggest.showValues": false,
    "editor.suggest.showWords": false,
    "window.zoomLevel": 1,
    "workbench.editor.pinnedTabSizing": "compact",
    "editor.unicodeHighlight.invisibleCharacters": false,
    "editor.unicodeHighlight.ambiguousCharacters": false,
    "mssql.tableDesigner.preloadDatabaseModel": true,
    "editor.detectIndentation": false,
    "dashboard.server.properties": true,
    "workbench.colorCustomizations": {
        "queryEditor.nullBackground": "#3e3430"
    },
    "workbench.colorTheme": "Default Dark Azure Data Studio",
    "queryEditor.results.showCopyCompletedNotification": false
}
```

Azure Data Studio Keybinding
```
// Place your key bindings in this file to override the defaultsauto[]
[
    {
        "key": "shift+win+r",
        "command": "workbench.action.openRecent"
    },
    {
        "key": "ctrl+r",
        "command": "-workbench.action.openRecent"
    },
    {
        "key": "shift+win+r",
        "command": "workbench.action.reloadWindow",
        "when": "isDevelopment"
    },
    {
        "key": "ctrl+r",
        "command": "-workbench.action.reloadWindow",
        "when": "isDevelopment"
    },
    {
        "key": "shift+win+r",
        "command": "workbench.action.quickOpenNavigateNextInRecentFilesPicker",
        "when": "inQuickOpen && inRecentFilesPicker"
    },
    {
        "key": "ctrl+r",
        "command": "-workbench.action.quickOpenNavigateNextInRecentFilesPicker",
        "when": "inQuickOpen && inRecentFilesPicker"
    },
    {
        "key": "ctrl+r",
        "command": "toggleQueryResultsKeyboardAction",
        "when": "queryEditorVisible"
    },
    {
        "key": "shift+win+r",
        "command": "-toggleQueryResultsKeyboardAction",
        "when": "queryEditorVisible"
    },
    {
        "key": "ctrl+e",
        "command": "editor.action.insertSnippet",
        "when": "editorTextFocus",
        "args": {
            "snippet": "BEGIN TRANSACTION;\n/**** <Your SQL> ****/\n/********************/\n\n\n${TM_SELECTED_TEXT}\n\n\n/*********************/\n/**** </Your SQL> ****/\nROLLBACK TRANSACTION;"
        }
    },
    {
        "key": "shift+delete",
        "command": "editor.action.deleteLines",
        "when": "textInputFocus && !editorReadonly"
    },
    {
        "key": "ctrl+shift+k",
        "command": "-editor.action.deleteLines",
        "when": "textInputFocus && !editorReadonly"
    },
    {
        "key": "shift+delete",
        "command": "-deleteFile",
        "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceReadonly && !inputFocus"
    },
    {
        "key": "shift+delete",
        "command": "-editor.action.clipboardCutAction"
    },
    {
        "key": "ctrl+d",
        "command": "-editor.action.addSelectionToNextFindMatch",
        "when": "editorFocus"
    },
    {
        "key": "ctrl+d",
        "command": "editor.action.copyLinesDownAction",
        "when": "editorTextFocus && !editorReadonly"
    },
    {
        "key": "shift+alt+down",
        "command": "-editor.action.copyLinesDownAction",
        "when": "editorTextFocus && !editorReadonly"
    }
]
```
