---
layout: post
title: Degree Precision vs. Length
tags: [til, postgis]
categories: til postgis
date: 2020-09-10 09:37:00 +0200
toc: true
---

Trust me. It's okay to truncate your GPS updates coordinates.

```jsx
var Point = [24.9322965651688, 60.1921775614599]
```

```
decimal
places   degrees          distance
-------  -------          --------
0        1                111  km
1        0.1              11.1 km
2        0.01             1.11 km
3        0.001            111  m
4        0.0001           11.1 m
5        0.00001          1.11 m
6        0.000001         11.1 cm
7        0.0000001        1.11 cm
8        0.00000001       1.11 mm
9        0.000000001      111  μm
10       0.0000000001     11.1 μm
11       0.00000000001    1.11 μm
12       0.000000000001   111  nm
13       0.0000000000001  11.1 nm

```