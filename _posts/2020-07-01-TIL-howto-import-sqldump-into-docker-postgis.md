---
layout: post
title: How to import an sql dump into a database that is running in a container
tags: [til, postgis]
categories: til postgis
date: 2020-07-01 09:37:00 +0200
toc: true
---

First you get the volume that is attached to the container your database is running in.

```
docker inspect -f '{{ json .Mounts }}' db-name | python -m json.tool

>>
[
    {
        "Type": "volume",
        "Name": "99b4bbceeb44156d1b8a800f1e1ba3bdd9d381bf46de383acccf92a6f813b7c7",
        "Source": "/var/lib/docker/volumes/99b4bbceeb44156d1b8a800f1e1ba3bdd9d381bf46de383acccf92a6f813b7c7/_data",
        "Destination": "/var/lib/postgresql/data",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
]
```

Afterwards you get the `id` of the container your db is running in

```
docker ps -aqf "name=^db-name$"
>> 
503ca8bf7a07
```

Now you can copy your `dump.sql` using this:

```
docker cp dump.sql <container-id>:/var/lib/postgresql/data
```

OR

do it in one go using this:

```
docker cp dump.sql $(docker ps -aqf "name=^db-name$"):/var/lib/postgresql/data
```

Then use 

```
docker-compose exec db-name bash
```

to get a bash console in you db container and exec 

```
psql -U username db-name < var/lib/postgresql/data/dump.sql
```