---
layout: post
title: pyle38 - An async python client for Tile38
tags: [tile38, database, geodata, pyle38, python, async]
categories: database go tile38
date: 2021-07-31 13:37:00 +0200
---

<p align="center">
<img src="/img/2021-07-21-pyle38/pyle38.png" alt="pyle38">
</p>

Back in May I wrote an introductory blog post about [Tile38](https://iwpnd.pw/articles/2021-04/Tile38-in-memory-geodatabase). Professionally I work with Tile38 in
a TypeScript eco-system. Togther with a colleague I authored a TypeScript client
that is now used internally - and hopefully soon to be released to the public.
As there was also no client for Python either yet and due to the pandemic there
was more free time than usual on my hand I wrote one myself.

## Introducing Pyle38

Pyle38 is a lazy asynchonous Python client for Tile38 that allows for fast and easy
interaction with the worlds fastest in-memory geodatabase [Tile38](https://tile38.com) build
on top of the re-designed [aioredis 2.0.0](https://aioredis.readthedocs.io/en/latest/migration/).
Even though the Python community is only slowly adapting typing into Python, I
made sure that Pyle38 is already fully typed and passes [mypy](http://mypy-lang.org/)
validation. [Pydantic](https://pydantic-docs.helpmanual.io/) is used to enforce
those type hints at runtime.

## Feature: Command chaining

Tile38 provides a lot of options for its queries, configs and insert commands.

```bash
WITHIN fleet LIMIT 10 NOFIELDS IDS CIRCLE 52.25 13.37 1000
```

Returns the ids of 10 objects in the `fleet` collection in a radius of 1000
around a point with latitude 52.25 and longitude 13.37. The order of the
the commands and options is fixed, putting the `LIMIT 10` at the end of the command
would result in an error.

The same query in pyle38 would look like this:

```python
await tile38.within('fleet').limit(10).nofields().circle(52.25, 13.37, 1000).asIds()
```

The order of the commands in pyle38 can, but does not have to be exactly as required in Tile38.
Commands can be chained to the users liking and pyle38 will take care that the order
is correct when the command is passed to Tile38.

## Feature: fully asynchronous

Pyle38 methods are almost entirely asynchronous. Why? For one I wanted to learn
how to do it. On the other hand the application I had in mind for Pyle38/Tile38
was as a backend for a [FastAPI](https://fastapi.tiangolo.com/) app.

## Feature: Leader/Follower replication

Tile38 supports [leader/follower replication](https://iwpnd.pw/articles/2021-04/Tile38-in-memory-geodatabase#leader-and-follower-replication)
and so does Pyle38. Pyle38 allows to instantiate a two clients at once, one for
the leader and one for a follower. That allows to let writes be handled by the
leader and reads by the follower.

## Basic Usage

```python
import asyncio
from pyle38 import Tile38


async def main():
    tile38 = Tile38(url="redis://localhost:9851", follower_url="redis://localhost:9851")

    await tile38.set("fleet", "truck").point(52.25,13.37).exec()

    response = await tile38.follower()
        .within("fleet")
        .circle(52.25, 13.37, 1000)
        .asObjects()

    assert response.ok

    print(response.dict())

asyncio.run(main())

> {
    "ok": True,
    "elapsed": "48.8µs",
    "objects": [
        {
            "object": {
                "type": "Point",
                "coordinates": [
                    13.37,
                    52.25
                ]
            },
            "id": "truck"
        }
    ],
    "count": 1,
    "cursor": 0
}
```

Let's go and dissect this example.

```python
tile38 = Tile38(url="redis://localhost:9851", follower_url="redis://localhost:9851")
```

First, an instance of Tile38 is created. Optionally a follower url can be passed
to Tile38 in the constructor, that will create a both a client for the leader,
as well as the follower.

```python
await tile38.set("fleet", "truck").point(52.25,13.37).exec()
```

Now we add an object with the name `truck` to the `fleet` collection as a point
with latitude 52.25 and longitude 13.37. `SET`'s have to be executed specifically
with `.exec()`.

```python
response = await tile38.follower()
        .within("fleet")
        .circle(52.25, 13.37, 1000)
        .asObjects()
```

As there is now an object in the fleet collection. The user can now send a query
to Tile38. The `.follower()` method indicates that the query should specifically
be send to the follower instance of Tile38.
`.within('fleet')` searches a collection for objects that are fully
contained within a given bounding area. The bounding area that is provided in
this query is a `.circle(52.25, 13.37, 1000)` with a radius of `1000` around the
coordinates `52.25` and `13.37`.

## Get it now

You can download pyle38 via `pip install pyle38` or if Python is not your thing, you can
check our other officially supported [Tile38 clients](https://tile38.com/topics/client-libraries).
