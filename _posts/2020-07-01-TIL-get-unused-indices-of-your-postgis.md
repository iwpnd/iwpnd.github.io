---
layout: post
title: Get unused indices of your PostGIS database
tags: [til, postgis]
categories: til postgis
date: 2020-07-01 09:37:00 +0200
toc: true
---

Unused indices are the bane of your production database as even though they are unused, they have to be rebuild and cleaned along the lifetime of a table.

To quickly validate if the index you've created on design time are actually used at run time, you can simply use this handy little snipped to clean up your indices.

```sql
SELECT 	ui.schemaname,
       	ui.relname 		AS table_name,
       	ui.indexrelname 	AS index_name,
       	pg_relation_size(ui.indexrelid) AS index_size
FROM 	pg_catalog.pg_stat_user_indexes ui,
	pg_catalog.pg_index pi
WHERE 	ui.idx_scan = 0      -- has never been scanned
AND    	ui.indexrelid = pi.indexrelid  
AND 	0 <>ALL (pi.indkey)  -- no index column is an expression
AND NOT pi.indisunique   -- is not a UNIQUE index
AND NOT EXISTS          -- does not enforce a constraint
         	(
	SELECT 1 
	FROM pg_catalog.pg_constraint pc
        WHERE pc.conindid = ui.indexrelid
)
ORDER BY table_name, pg_relation_size(ui.indexrelid) DESC
```

source: [AWS](https://aws.amazon.com/blogs/database/reducing-aurora-postgresql-storage-i-o-costs/)