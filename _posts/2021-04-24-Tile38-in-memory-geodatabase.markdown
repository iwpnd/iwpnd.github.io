---
layout: post
title: Tile38 in-memory geodatabase
tags: [tile38, database, geodata]
categories: database go tile38
date: 2021-04-24 13:37:00 +0200
toc: true
---

<p align="center">
<img src="/img/2021-04-tile38/tile38banner.png" alt="tile38banner">
</p>

For the longest time, it was everything about PostgreSQL and PostGIS. Building something that only remotely has to handle geo-data I had PostGIS on speed-dial. Requirement analysis of what I needed from the vastness of functionality that PostGIS packs? Nope. Never. Really.
What it mostly boiled down to were point-in-polygon, radius searches and if generous a KNN nearby search for either points or polygons. Do you need to use PostGIS for that? Well, if you have a bunch of points or a bunch of polygons I would do best to consider PostGIS for geo-indexed searches. That is until a friend introduced me to Tile38.

## Tile38
[Tile38](https://tile38.com) is an open-source, in-memory geo data store, spatial index, and real-time geofence framework written by Josh Baker ([@tidwall](https://github.com/tidwall))
There are some things to unpack here.

## In-memory geo datastore
Yes, Tile38 works in memory like Redis, without Redis but with broadly understood Redis Protocol. So while PostGIS stores and reads data from disk, Tile38 reads and writes to and from memory. This makes it blazingly fast, as Disk I/O is removed almost entirely. Almost? Well, Tile38 by default stores every command it receives in an append-only file that can be used to restore state in case of failure.
As Tile38 uses the Redis Protocol, using Tile38 comes naturally to everyone that previously worked with Redis.
```
SET fleet truck1 POINT 33.5123 -112.2693
```

This adds `truck1` to a `fleet` collection as a `POINT` with latitude `33.4626` and longitude `-112.1695`.

```
GET fleet truck1
```

And `GET` returns `truck1` from the `fleet` collection.

## Rtree spatial index
Spatial indexes are what make a spatial database. Without an index, any search would be a sequential scan - something everybody wants to avoid at all costs when querying a database. Put simply what a spatial index does, is it groups geometries (Points, LineStrings, Polygons) using a minimum bounding rectangle. If you now have a point database and want to know how many points are within a request polygon - instead of checking every possible point, it would compare if the minimum bounding rectangles overlap and only check those that are overlapping. Overly simplified, but that’s the gist (ha!) of it.
Now let’s put this index to use.
```
WITHIN fleet CIRCLE 33.462 -112.268 6000

> {
    "ok": True,
    "elapsed": "48.8µs",
    "objects": [
        {
            "object": {
                "type": "Point",
                "coordinates": [
                    -112.2693,
                    33.5123
                ]
            },
            "id": "truck"
        }
    ],
    "count": 1,
    "cursor": 0
}
```
This will check the Tile38 collection `fleet` for every item, that is within a `CIRCLE` with a center point of latitude `33.462`, longitude `-112.268`, and a radius of `6000` meters.

## Realtime geofence framework
Performing geo-searches is mind-boggling. When I was first able to show a manager of PostGIS capabilities and speed, it blew their mind. And even today, some fellow developers stand in awe, when they request a feature from me, and I can wrap something up with PostGIS, in an instance that speeds up their services. But now imagine that instead of requesting whether a point is in a polygon from a database, the database would instead tell you when the point enters and exits the area. That is what a geofence is, and this is what Tile38 provides. This is just next-level and opens up so many possible applications.
Adding a geofenced search to Tile38 is as easy as:
```
SETHOOK warehouse http://10.1.3.37/endpoint NEARBY fleet FENCE POINT 33.5123, -112.2693 500
```
This will make Tile38 send events to the given endpoint, whenever an object is in the collection `fleet` enters the area of a `500`-meter radius around latitude `33.5123` and longitude `-112.2693`.

## Leader and follower replication
Leader-follower replication is something that you want for every database where reliability and availability are key. In most setups like that (e.g. AWS Aurora PostgreSQL) you will have a writer (leader) that will handle incoming CREATE and UPDATE statements. And on the other hand, you will have a variable amount of reader instances that share the load of incoming requests among each other. Tile38 offers a similar setup. Say you start two Tile38 instances and call FOLLOW leader-uri:9581, then one will become a leader, the other will be a follower. The follower will request the current state of the leader and once caught up, will be good to receive queries. A leader will still be able to handle incoming query commands, but a follower will reject SET commands going forward. If the leader dies, the followers will become stale until the leader is back up, but are still able to complete requests sent to them so your application will continue to work. Something you will have to make sure is that data sent to the leader during downtime is stored somewhere - something like Apache Kafka is a good candidate for that. This setup allows for a pretty fault-tolerant setup as it works like a charm with horizontal auto-scalers for example in Kubernetes. The auto-scaler will just add pods to your service when the load on all followers exceeds a set threshold. And trust me, this happens fast. Compare this to something like AWS Aurora with PostgreSQL, and you’re not only more reliable but also waaaay cheaper off using Tile38.

## Why is PostGIS used in the application?
Good question, with the most obvious answer being that it is plain awesome still, well understood, highly customizable, and offers functionality that is unrivaled by any other geodatabase out there. But Tile38 brought a viable alternative to PostGIS into the mix that for most applications is not only sufficient enough but also offers completely new use-cases in today's modern event-driven architectures. The possibility to track IoT devices in space and being notified about their whereabouts is mind-bogglingly awesome. I build and will continue building applications with Tile38 going forward. But don’t worry PostGIS, there’s not a kid on the block that rivals you completely.