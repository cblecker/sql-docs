---
title: "sp_foreignkeys (Transact-SQL)"
description: "sp_foreignkeys (Transact-SQL)"
author: markingmyname
ms.author: maghan
ms.date: "03/14/2017"
ms.service: sql
ms.subservice: system-objects
ms.topic: "reference"
f1_keywords:
  - "sp_foreignkeys_TSQL"
  - "sp_foreignkeys"
helpviewer_keywords:
  - "sp_foreignkeys"
dev_langs:
  - "TSQL"
---
# sp_foreignkeys (Transact-SQL)
[!INCLUDE [SQL Server](../../includes/applies-to-version/sqlserver.md)]

  Returns the foreign keys that reference primary keys on the table in the linked server.  
  
 :::image type="icon" source="../../includes/media/topic-link-icon.svg" border="false"::: [Transact-SQL syntax conventions](../../t-sql/language-elements/transact-sql-syntax-conventions-transact-sql.md)  
  
## Syntax  
  
```  
  
sp_foreignkeys [ @table_server = ] 'table_server'   
     [ , [ @pktab_name = ] 'pktab_name' ]   
     [ , [ @pktab_schema = ] 'pktab_schema' ]   
     [ , [ @pktab_catalog = ] 'pktab_catalog' ]   
     [ , [ @fktab_name = ] 'fktab_name' ]   
     [ , [ @fktab_schema = ] 'fktab_schema' ]   
     [ , [ @fktab_catalog = ] 'fktab_catalog' ]  
```  
  
## Arguments  
`[ @table_server = ] 'table_server'`
 Is the name of the linked server for which to return table information. *table_server* is **sysname**, with no default.  
  
`[ @pktab_name = ] 'pktab_name'`
 Is the name of the table with a primary key. *pktab_name* is **sysname**, with a default of NULL.  
  
`[ @pktab_schema = ] 'pktab_schema'`
 Is the name of the schema with a primary key. *pktab_schema*is **sysname**, with a default of NULL. In [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)], this contains the owner name.  
  
`[ @pktab_catalog = ] 'pktab_catalog'`
 Is the name of the catalog with a primary key. *pktab_catalog*is **sysname**, with a default of NULL. In [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)], this contains the database name.  
  
`[ @fktab_name = ] 'fktab_name'`
 Is the name of the table with a foreign key. *fktab_name*is **sysname**, with a default of NULL.  
  
`[ @fktab_schema = ] 'fktab_schema'`
 Is the name of the schema with a foreign key. *fktab_schema*is **sysname**, with a default of NULL.  
  
`[ @fktab_catalog = ] 'fktab_catalog'`
 Is the name of the catalog with a foreign key. *fktab_catalog*is **sysname**, with a default of NULL.  
  
## Return Code Values  
 None  
  
## Result Sets  
 Various DBMS products support three-part naming for tables (_catalog_**.**_schema_**.**_table_), which is represented in the result set.  
  
|Column name|Data type|Description|  
|-----------------|---------------|-----------------|  
|**PKTABLE_CAT**|**sysname**|Catalog for the table in which the primary key resides.|  
|**PKTABLE_SCHEM**|**sysname**|Schema for the table in which the primary key resides.|  
|**PKTABLE_NAME**|**sysname**|Name of the table (with the primary key). This field always returns a value.|  
|**PKCOLUMN_NAME**|**sysname**|Name of the primary key column or columns, for each column of the **TABLE_NAME** returned. This field always returns a value.|  
|**FKTABLE_CAT**|**sysname**|Catalog for the table in which the foreign key resides.|  
|**FKTABLE_SCHEM**|**sysname**|Schema for the table in which the foreign key resides.|  
|**FKTABLE_NAME**|**sysname**|Name of the table (with a foreign key). This field always returns a value.|  
|**FKCOLUMN_NAME**|**sysname**|Name of the foreign key columns, for each column of the TABLE_NAME returned. This field always returns a value.|  
|**KEY_SEQ**|**smallint**|Sequence number of the column in a multicolumn primary key. This field always returns a value.|  
|**UPDATE_RULE**|**smallint**|Action applied to the foreign key when the SQL operation is an update. [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] returns 0, 1, or 2 for these columns:<br /><br /> 0=CASCADE changes to foreign key.<br /><br /> 1=NO ACTION changes if foreign key is present.<br /><br /> 2=SET_NULL; set foreign key to NULL.|  
|**DELETE_RULE**|**smallint**|Action applied to the foreign key when the SQL operation is a deletion. [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] returns 0, 1, or 2 for these columns:<br /><br /> 0=CASCADE changes to foreign key.<br /><br /> 1=NO ACTION changes if foreign key is present.<br /><br /> 2=SET_NULL; set foreign key to NULL.|  
|**FK_NAME**|**sysname**|Foreign key identifier. It is NULL if not applicable to the data source. [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] returns the FOREIGN KEY constraint name.|  
|**PK_NAME**|**sysname**|Primary key identifier. It is NULL if not applicable to the data source. [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] returns the PRIMARY KEY constraint name.|  
|**DEFERRABILITY**|**smallint**|Indicates whether constraint checking is deferrable.|  
  
 In the result set, the FK_NAME and PK_NAME columns always return NULL.  
  
## Remarks  
 **sp_foreignkeys** queries the FOREIGN_KEYS rowset of the **IDBSchemaRowset** interface of the OLE DB provider that corresponds to *table_server*. The *table_name*, *table_schema*, *table_catalog*, and *column* parameters are passed to this interface to restrict the rows returned.  
  
## Permissions  
 Requires SELECT permission on the schema.  
  
## Examples  
 The following example returns foreign key information about the `Department` table in the [!INCLUDE[ssSampleDBobject](../../includes/sssampledbobject-md.md)] database on the linked server, `Seattle1`.  
  
```  
EXEC sp_foreignkeys @table_server = N'Seattle1',   
   @pktab_name = N'Department',   
   @pktab_catalog = N'AdventureWorks2012';  
```  
  
## See Also  
 [sp_catalogs &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-catalogs-transact-sql.md)   
 [sp_column_privileges &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-column-privileges-transact-sql.md)   
 [sp_indexes &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-indexes-transact-sql.md)   
 [sp_linkedservers &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-linkedservers-transact-sql.md)   
 [sp_primarykeys &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-primarykeys-transact-sql.md)   
 [sp_tables_ex &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-tables-ex-transact-sql.md)   
 [sp_table_privileges &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-table-privileges-transact-sql.md)   
 [System Stored Procedures &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/system-stored-procedures-transact-sql.md)  
  
  
