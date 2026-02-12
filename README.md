# T-SQL Unique Constraints and Unique Indexes

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
