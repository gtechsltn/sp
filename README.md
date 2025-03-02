# Stored Procedure Caller Generator

https://github.com/gtechsltn/sp/

https://github.com/gtechsltn/DapperHelper

## How to get stored procedure parameters details?

https://stackoverflow.com/questions/20115881/how-to-get-stored-procedure-parameters-details

```
select  
   'Parameter_name' = name,  
   'Type'   = type_name(user_type_id),  
   'Length'   = max_length,  
   'Prec'   = case when type_name(system_type_id) = 'uniqueidentifier' 
              then precision  
              else OdbcPrec(system_type_id, max_length, precision) end,  
   'Scale'   = OdbcScale(system_type_id, scale),  
   'Param_order'  = parameter_id,  
   'Collation'   = convert(sysname, 
                   case when system_type_id in (35, 99, 167, 175, 231, 239)  
                   then ServerProperty('collation') end)
  from sys.parameters where object_id = object_id('dbo.core_insertEventLog')
```

## Passing Output parameters to stored procedure using dapper in c# code

https://stackoverflow.com/questions/22353881/passing-output-parameters-to-stored-procedure-using-dapper-in-c-sharp-code

```
public void TestProcSupport()
{
    var p = new DynamicParameters();
    p.Add("a", 11);
    p.Add("b", dbType: DbType.Int32, direction: ParameterDirection.Output);
    p.Add("c", dbType: DbType.Int32, direction: ParameterDirection.ReturnValue);
    connection.Execute(@"create proc #TestProc 
                         @a int,
                         @b int output
                         as 
                         begin
                             set @b = 999
                             select 1111
                             return @a
                         end");
    connection.Query<int>("#TestProc", p, commandType: CommandType.StoredProcedure).First().IsEqualTo(1111);
    p.Get<int>("c").IsEqualTo(11);
    p.Get<int>("b").IsEqualTo(999);
}
```

```
public void InsertData()
{
    var p = new DynamicParameters();
    p.Add("VAR1", "John");
    p.Add("VAR2", "McEnroe");
    p.Add("BASEID", 1);
    p.Add("NEWID", dbType: DbType.Int32, direction: ParameterDirection.Output);
    connection.Query<int>("SP_MYTESTpROC", p, commandType: CommandType.StoredProcedure);
    int newID =  p.Get<int>("NEWID");
}
```

###  Gen Code
+ Stored Procedure Caller Generator
+ https://github.com/gtechsltn/DapperHelper/

### What is the equivalent of 'describe table' in SQL Server?

https://stackoverflow.com/questions/319354/what-is-the-equivalent-of-describe-table-in-sql-server

+ Get [name], [type], [default value] and also [comment] of [column] in [table]
+ Lấy [tên], [kiểu], [giá trị mặc định] và cả [comments] của [cột] trong [bảng]

```
IF OBJECT_ID('sp_desc', 'P') IS NOT NULL
  DROP PROCEDURE sp_desc
GO

CREATE PROCEDURE [dbo].[sp_desc] (
  @tableName  nvarchar(128)
) AS

-- EXEC [dbo].[sp_desc] N'Assets'

BEGIN
  DECLARE @dbName       sysname;
  DECLARE @schemaName   sysname;
  DECLARE @objectName   sysname;
  DECLARE @objectID     int;
  DECLARE @tmpTableName varchar(100);
  DECLARE @sqlCmd       nvarchar(4000);

  SELECT @dbName = PARSENAME(@tableName, 3);
  IF @dbName IS NULL SELECT @dbName = DB_NAME();

  SELECT @schemaName = PARSENAME(@tableName, 2);
  IF @schemaName IS NULL SELECT @schemaName = SCHEMA_NAME();

  SELECT @objectName = PARSENAME(@tableName, 1);
  IF @objectName IS NULL
    BEGIN
      PRINT 'Object is missing from your function call!';
      RETURN;
    END;

  SELECT @objectID = OBJECT_ID(@dbName + '.' + @schemaName + '.' + @objectName);
  IF @objectID IS NULL
    BEGIN
      PRINT 'Object [' + @dbName + '].[' + @schemaName + '].[' + @objectName + '] does not exist!';
      RETURN;
    END;

  SELECT @tmpTableName = '#tmp_DESC_' + CAST(@@SPID AS VARCHAR) + REPLACE(REPLACE(REPLACE(REPLACE(CAST(CONVERT(CHAR, GETDATE(), 121) AS VARCHAR), '-', ''), ' ', ''), ':', ''), '.', '');
  --PRINT @tmpTableName;
  SET @sqlCmd = '
    USE ' + @dbName + '
    CREATE TABLE ' + @tmpTableName + ' (
      [NAME]              nvarchar(128) NOT NULL
     ,[TYPE]              varchar(50)
     ,[CHARSET]           varchar(50)
     ,[COLLATION]         varchar(50)
     ,[NULLABLE]          varchar(3)
     ,[DEFAULT]           nvarchar(4000)
     ,[COMMENTS]          nvarchar(3750));

    INSERT INTO ' + @tmpTableName + '
    SELECT
      a.[NAME]
     ,a.[TYPE]
     ,a.[CHARSET]
     ,a.[COLLATION]
     ,a.[NULLABLE]
     ,a.[DEFAULT]
     ,b.[COMMENTS]
    FROM
      (
        SELECT
          COLUMN_NAME                                     AS [NAME]
         ,CASE DATA_TYPE
            WHEN ''char''      THEN DATA_TYPE + ''('' + CAST(CHARACTER_MAXIMUM_LENGTH AS VARCHAR) + '')''
            WHEN ''numeric''   THEN DATA_TYPE + ''('' + CAST(NUMERIC_PRECISION AS VARCHAR) + '', '' + CAST(NUMERIC_SCALE AS VARCHAR) + '')''
            WHEN ''nvarchar''  THEN DATA_TYPE + ''('' + CAST(CHARACTER_MAXIMUM_LENGTH AS VARCHAR) + '')''
            WHEN ''varbinary'' THEN DATA_TYPE + ''('' + CAST(CHARACTER_MAXIMUM_LENGTH AS VARCHAR) + '')''
            WHEN ''varchar''   THEN DATA_TYPE + ''('' + CAST(CHARACTER_MAXIMUM_LENGTH AS VARCHAR) + '')''
            ELSE DATA_TYPE
          END                                             AS [TYPE]
         ,CHARACTER_SET_NAME                              AS [CHARSET]
         ,COLLATION_NAME                                  AS [COLLATION]
         ,IS_NULLABLE                                     AS [NULLABLE]
         ,COLUMN_DEFAULT                                  AS [DEFAULT]
         ,ORDINAL_POSITION
        FROM   
          INFORMATION_SCHEMA.COLUMNS
        WHERE   
          TABLE_NAME = ''' + @objectName + '''
      ) a
      FULL JOIN
      (
         SELECT
           CAST(value AS NVARCHAR)                        AS [COMMENTS]
          ,CAST(objname AS NVARCHAR)                      AS [NAME]
         FROM
           ::fn_listextendedproperty (''MS_Description'', ''user'', ''' + @schemaName + ''', ''table'', ''' + @objectName + ''', ''column'', default)
      ) b
      ON a.NAME COLLATE Hungarian_CI_AS = b.NAME COLLATE Hungarian_CI_AS
    ORDER BY
      a.[ORDINAL_POSITION];

    SELECT * FROM ' + @tmpTableName + ';'

    PRINT @sqlCmd;

    EXEC sp_executesql @sqlCmd;
    RETURN;
END;
GO
```

```
EXEC [dbo].[sp_help] N'Assets'
EXEC [dbo].[sp_columns] N'Assets'
EXEC [dbo].[sp_desc] N'Assets'
```

## How to describe table in SQL Server

Follow steps:

Write Table Name,

Select it, and press Alt + F1

It will show detailed information/ description of mentioned table as,

1) Table created date,
2) Columns Description,
3) Identity,
4) Indexes,
5) Constraints,
6) References etc. As shown Below

https://stackoverflow.com/questions/11078106/how-to-describe-table-in-sql-server-2008

https://github.com/gtechsltn/Sybase-ASE-shell-Script

https://github.com/kimls2/Sybase-ASE-shell-Script
