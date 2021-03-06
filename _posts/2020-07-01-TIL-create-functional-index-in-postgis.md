---
layout: post
title: Create a functional index in PostGIS
tags: [til, postgis]
categories: til postgis
date: 2020-07-01 09:37:00 +0200
toc: true
---


It possible to set an index on the return value of a function.

Say you're casting a `geometry` to `geography` a lot. You have an `GIST` on the `geometry`, but that index is not utilized when you cast the `geometry` to `geography`. 

What you can do to circumvent that without creating a `geography` column and index that, is to create the index on the return of `CAST(your_geometry AS geography)` or `geography(your_geometry_column`.

```sql
create index idx_cast_geography_on_your_geometry_column 
on your_table 
using GIST (
	geography(your_geometry_column)
);
```

source: [gis.stackexchange.com](https://gis.stackexchange.com/a/247131)