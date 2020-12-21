---
layout: post
title: Spatially cluster PostGIS tables
tags: [til, postgis]
categories: til postgis
date: 2020-08-05 09:37:00 +0200
toc: true
---


If your query pattern evolves around getting geometries that are often spatially close together it might make sense to `cluster` the table on a spatial correlated field.

```sql
-- first create a functional index on the geohash of your geometry
create index idx_polygon_geohash on zone (st_geohash(polygon));

-- vacuum and analyze your db
vacuum analyze zone;

-- cluster the table on that newly created index
cluster zone using idx_cl_polygon_geohash;
```

performance gains heavily depend on the size of the response payload and the query pattern.

see: [here](http://postgis.net/workshops/postgis-intro/clusterindex.html)