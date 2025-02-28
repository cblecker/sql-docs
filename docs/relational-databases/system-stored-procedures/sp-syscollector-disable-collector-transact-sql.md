---
title: "sp_syscollector_disable_collector (Transact-SQL)"
description: "sp_syscollector_disable_collector (Transact-SQL)"
author: markingmyname
ms.author: maghan
ms.date: "03/14/2017"
ms.service: sql
ms.subservice: system-objects
ms.topic: "reference"
f1_keywords:
  - "sp_syscollector_disable_collector_TSQL"
  - "sp_syscollector_disable_collector"
helpviewer_keywords:
  - "data collector [SQL Server], stored procedures"
  - "sp_syscollector_disable_collector"
dev_langs:
  - "TSQL"
---
# sp_syscollector_disable_collector (Transact-SQL)
[!INCLUDE [SQL Server](../../includes/applies-to-version/sqlserver.md)]

  Disables the data collector. Because there is only one data collector per server, no parameters are required.  
  
 :::image type="icon" source="../../includes/media/topic-link-icon.svg" border="false"::: [Transact-SQL syntax conventions](../../t-sql/language-elements/transact-sql-syntax-conventions-transact-sql.md)  
  
## Syntax  
  
```  
  
dbo.sp_syscollector_disable_collector   
```  
  
## Arguments  
 None  
  
## Return Code Values  
 **0** (success) or **1** (failure)  
  
## Remarks  
 Defaults to the data collector on the server.  
  
## Permissions  
 Requires membership in the **dc_admin** or **dc_operator** (with EXECUTE permission) fixed database role to execute this procedure.  
  
## Examples  
 The following example disables the data collector.  
  
```  
EXEC dbo.sp_syscollector_disable_collector;  
```  
  
## See Also  
 [System Stored Procedures &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/system-stored-procedures-transact-sql.md)   
 [Data Collection](../../relational-databases/data-collection/data-collection.md)  
  
  
