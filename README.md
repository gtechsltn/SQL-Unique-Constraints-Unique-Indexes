# T-SQL Unique Constraints and Unique Indexes
+ **Unique constraints** are logical rules (ALTER TABLE ADD CONSTRAINT ... UNIQUE)
+ **Unique indexes** enforce uniqueness but may not be declared as constraints
+ **Many databases use indexes instead of constraints.**

# View table definition (table structure) using T-SQL

## Cách nhanh nhất (giống SSMS)
```
EXEC sp_help 'dbo.sites';
```

Sẽ hiển thị:

+ Columns
+ Data types
+ Identity
+ Indexes
+ Constraints

## Chỉ xem danh sách cột + kiểu dữ liệu
```
SELECT 
    c.column_id,
    c.name AS column_name,
    t.name AS data_type,
    c.max_length,
    c.precision,
    c.scale,
    c.is_nullable,
    c.is_identity
FROM sys.columns c
JOIN sys.types t 
    ON c.user_type_id = t.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.sites')
ORDER BY c.column_id;
```

## Xem Primary Key
```
SELECT 
    kc.name AS pk_name,
    c.name AS column_name
FROM sys.key_constraints kc
JOIN sys.index_columns ic 
    ON kc.parent_object_id = ic.object_id
   AND kc.unique_index_id = ic.index_id
JOIN sys.columns c 
    ON ic.object_id = c.object_id
   AND ic.column_id = c.column_id
WHERE kc.type = 'PK'
  AND kc.parent_object_id = OBJECT_ID('dbo.sites');
```

## Xem Unique Constraints
```
SELECT 
    kc.name,
    c.name AS column_name
FROM sys.key_constraints kc
JOIN sys.index_columns ic 
    ON kc.parent_object_id = ic.object_id
   AND kc.unique_index_id = ic.index_id
JOIN sys.columns c 
    ON ic.object_id = c.object_id
   AND ic.column_id = c.column_id
WHERE kc.type = 'UQ'
  AND kc.parent_object_id = OBJECT_ID('dbo.sites');
```

## Generate CREATE TABLE script (rất hữu ích)
+ Cách chuẩn bằng SSMS (tốt nhất ngoài đời)
+ Nếu bạn dùng SQL Server Management Studio:
+ Right click table → Script Table as → CREATE To → New Query Editor Window
    + Chính xác 100%
    + Bao gồm PK, FK, indexes, defaults, identity, etc.

## Cách T-SQL thuần (reconstruct CREATE TABLE) để **Generate CREATE TABLE + PRIMARY KEY**
+ Columns
+ Data types (varchar/nvarchar/decimal/etc.)
+ NULL / NOT NULL
+ IDENTITY
+ Primary Key

(An toàn với lỗi **STRING_AGG aggregation result exceeded the limit of 8000 bytes. Use LOB types to avoid result truncation.**)

```
DECLARE @TableName SYSNAME = 'sites';
DECLARE @SchemaName SYSNAME = 'dbo';

DECLARE @ObjectId INT = OBJECT_ID(QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName));

-- Columns
WITH cols AS (
    SELECT 
        c.column_id,
        '    ' + QUOTENAME(c.name) + ' ' + 
        t.name +
        CASE 
            WHEN t.name IN ('varchar','char','varbinary','binary')
                THEN '(' + CASE WHEN c.max_length = -1 THEN 'MAX'
                                ELSE CAST(c.max_length AS VARCHAR(10)) END + ')'
            WHEN t.name IN ('nvarchar','nchar')
                THEN '(' + CASE WHEN c.max_length = -1 THEN 'MAX'
                                ELSE CAST(c.max_length / 2 AS VARCHAR(10)) END + ')'
            WHEN t.name IN ('decimal','numeric')
                THEN '(' + CAST(c.precision AS VARCHAR(10)) + ',' + CAST(c.scale AS VARCHAR(10)) + ')'
            ELSE ''
        END +
        CASE WHEN c.is_identity = 1
                THEN ' IDENTITY(' + 
                     CAST(ic.seed_value AS VARCHAR(10)) + ',' + 
                     CAST(ic.increment_value AS VARCHAR(10)) + ')'
             ELSE ''
        END +
        CASE WHEN c.is_nullable = 1 THEN ' NULL' ELSE ' NOT NULL' END
        AS column_def
    FROM sys.columns c
    JOIN sys.types t 
        ON c.user_type_id = t.user_type_id
    LEFT JOIN sys.identity_columns ic
        ON c.object_id = ic.object_id
       AND c.column_id = ic.column_id
    WHERE c.object_id = @ObjectId
),

pk AS (
    SELECT 
        kc.name AS pk_name,
        STRING_AGG(QUOTENAME(c.name), ', ') 
            WITHIN GROUP (ORDER BY ic.key_ordinal) AS pk_columns
    FROM sys.key_constraints kc
    JOIN sys.index_columns ic
        ON kc.parent_object_id = ic.object_id
       AND kc.unique_index_id = ic.index_id
    JOIN sys.columns c
        ON ic.object_id = c.object_id
       AND ic.column_id = c.column_id
    WHERE kc.parent_object_id = @ObjectId
      AND kc.type = 'PK'
    GROUP BY kc.name
)

SELECT 
    CAST('CREATE TABLE ' + QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName) + CHAR(10) +
         '(' + CHAR(10) AS NVARCHAR(MAX)) +

    (SELECT STRING_AGG(CAST(column_def AS NVARCHAR(MAX)), ',' + CHAR(10)) FROM cols) +

    ISNULL(
        (SELECT ',' + CHAR(10) + 
                '    CONSTRAINT ' + QUOTENAME(pk_name) +
                ' PRIMARY KEY (' + pk_columns + ')'
         FROM pk),
        ''
    ) +

    CHAR(10) + ');' AS CreateTableScript;
```

## Cách dùng view **INFORMATION_SCHEMA.COLUMNS**
```
SELECT *
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'sites'
  AND TABLE_SCHEMA = 'dbo';
```

## View clustered và nonclustered
+ Tên index
+ Có phải Primary Key hay không
+ Có unique hay không
+ Loại index (clustered / nonclustered)
+ Tên cột thuộc index
+ Thứ tự cột trong index (quan trọng với composite key)

```
SELECT 
    i.name AS index_name,
    i.is_primary_key,
    i.is_unique,
    i.type_desc,
    c.name AS column_name,
    ic.key_ordinal
FROM sys.indexes i
JOIN sys.index_columns ic
    ON i.object_id = ic.object_id
   AND i.index_id = ic.index_id
JOIN sys.columns c
    ON ic.object_id = c.object_id
   AND ic.column_id = c.column_id
WHERE i.object_id = OBJECT_ID('dbo.sites')
ORDER BY i.name, ic.key_ordinal;
```

# Unique Constraints
```
--Unique Constraints on sites
SELECT 
    kc.name AS constraint_name,
    c.name  AS column_name,
    ic.key_ordinal
FROM sys.key_constraints kc
JOIN sys.indexes i
    ON kc.parent_object_id = i.object_id
   AND kc.unique_index_id = i.index_id
JOIN sys.index_columns ic
    ON i.object_id = ic.object_id
   AND i.index_id = ic.index_id
JOIN sys.columns c
    ON ic.object_id = c.object_id
   AND ic.column_id = c.column_id
WHERE kc.type = 'UQ'
  AND kc.parent_object_id = OBJECT_ID('dbo.sites')   -- adjust schema if needed
ORDER BY kc.name, ic.key_ordinal;
```

# Unique Indexes
```
--Unique Indexes on sites
SELECT 
    i.name AS index_name,
    c.name AS column_name,
    ic.key_ordinal
FROM sys.indexes i
JOIN sys.index_columns ic
    ON i.object_id = ic.object_id
   AND i.index_id = ic.index_id
JOIN sys.columns c
    ON ic.object_id = c.object_id
   AND ic.column_id = c.column_id
WHERE i.is_unique = 1
  AND i.object_id = OBJECT_ID('dbo.sites')   -- adjust schema if needed
ORDER BY i.name, ic.key_ordinal;
```
