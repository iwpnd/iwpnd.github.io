---
layout: post
title: Apache Kafka producer and consumer with FastAPI and aiokafka
tags: [python, kafka, FastAPI]
categories: python kafka FastAPI
date: 2020-03-11 13:37:00 +0200
toc: true
---

After I finally published [flashgeotext](https://iwpnd.pw/articles/2020-02/flashgeotext-library) last month, it was time for the next item on the to-do list - Apache Kafka. After I somehow avoided the topic completely for the last years, aside from that one friend that wouldn't shut up about it, I noticed that more often than not Apache Kafka knowledge is a requirement for a lot of positions in data engineering nowadays. So I did what I recommand everybody starting out - check out [Tim Berglund's introduction on Apache Kafka](https://www.youtube.com/watch?v=06iRM1Ghr1k) on YouTube and skinny-dip into the rabbit hole from there.

# tl;dr

Find the finished project on GitHub: [geo-stream-kafka](https://github.com/iwpnd/geo-stream-kafka)

# What is Apache Kafka?

Always great to read an introduction into a complex topic from somebody who just recently started out with it, right? What could go wrong. So in its core, Apache Kafka is a messaging system with somebody/something producing a message on the one side and a somebody/something consuming the message on the other side, and a lot of magic in between. 

## Kafka core principles

To zoom in on the magic part, when a producer sends a message, the message is pushed into Kafka topics. When a consumer consumes a message it is pulling the message from a Kafka topic. Kafka topics reside within a so-called broker (eg. Zookeeper). Zookeeper provides synchronization within distributed systems and in the case of Apache Kafka keeps track of the status of Kafka cluster nodes and Kafka topics.


<p align="center">
<img src="/img/2020-03-geostream-kafka/kafka.png" alt="setup geostream FastAPI aiokafka">
</p>


You can have multiple producers pushing messages into one topic, or you can have them push to different topics. Messages within topics can be retained indefinitly or be discarded after a certain time, depending on the needs. When a consumer starts consuming a message, it starts from the first message that has been pushed to the topic and continues from thereon. Kafka stores the so-called _offset_, basically a pointer telling the consumer which messages have been consumed and what is still left to indulge. Messages will stay within the topic, yet when the same consumer pulls messages from the topic it will only receive messages from the offset onwards.

There is actually a lot more to dissect here, but if you want to understand as much Apache Kafka to be dangerous, I'd say we leave it at that and I direct you to more competent, no less handsome, people to tell you more.

# master builders assemble

Okay, we have the theory let's build something with it. This time, instead of a [niche NLP problem](https://iwpnd.pw/articles/2020-02/flashgeotext-library) I wanted to go back to my origins as a geospatial engineer. I figured I would build a producer/consumer architecture for geodata. We will have a FastAPI endpoint that will wrap the producer logic and a consumer FastAPI WebSocket endpoint that a [leaflet.js](https://leafletjs.com/) map-application then can tap into. So far I have used Leaflet.js to display static data, so why not try and use it with dynamic data instead. What I had in mind was an architecture like that:


<p align="center">
<img src="/img/2020-03-geostream-kafka/geostream-FastAPI-kafka.png" alt="setup geostream FastAPI aiokafka">
</p>

## Setup an Apache Kafka cluster

Hell no, I would never dare to even try to comprehend what's necessary to build that up from scratch. BUT, we live in different times now and there is that handsome technology called Docker. We also can be thankful that respective members of the interwebs community like [wurstmeister](https://github.com/wurstmeister/) (german for Saugagemaster) prepare containers like [Kafka-docker](https://github.com/wurstmeister/kafka-docker) that are ready to be used for you local (maybe more?) development entertainment. Now I will say that even that container didn't come without troubles for me. I just wasn't able to connect to the Kafka broker from outside the container. There is [wurstmeisters kafka connectivity guide](https://github.com/wurstmeister/kafka-docker/wiki) that helped me to shed some light on the networking inside, but it didn't help me to get a connection from the outside. The docs suggest that you add 

```bash
environment:
      KAFKA_ADVERTISED_HOST_NAME: hostname
      KAFKA_CREATE_TOPICS: "geostream:1:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```

to your docker-compose file. There are three things to unpack. We can create the Kafka Topic right here and give it 1 partition and 1 replica. We connect Kafka container to the zookeeper container. Now to the connection problem. So I thought that I wanted to advertise Kafka to `localhost`. Yet it took me some time to understand why that would not be possible. See, what I learned is that Docker on Mac sets up a VM (yaya old news blabla), so `localhost` for Kafka inside its container is not your localhost but the IP of the VM your Kafka is running in. So I added this to my `.zshenv` file:

```bash
export DOCKER_KAFKA_HOST=$(ipconfig getifaddr en0)
```

and was able to advertise the hostname like so:

```bash
environment:
      KAFKA_ADVERTISED_HOST_NAME: ${DOCKER_KAFKA_HOST}
```

et voila, it's alive!

## Test the Apache Kafka cluster

Wait, how do we know it's alive and taking messages? We create a minimum viable Kafka producer and a minimum viable Kafka consumer, spin up to terminals and fire the up the producer and the consumer.

### Setup a minimum viable producer

```python
from pykafka import KafkaClient
import time

client = KafkaClient("127.0.0.1:9092")
geostream = client.topcis["geostream"]

with geostream.get_sync_producer() as producer:
    i = 0
    for _ in range(10):
        producer.produce(("Kafka is not just an author " + str(i).encode("ascii"))
        i += 1
        time.sleep(1)
```

### Setup a minimum viable consumer

```python
from pykafka import KafkaClient
client = KafkaClient(hosts="127.0.0.1:9092")

def get_messages(topicname):
    def events():
        for message in client.topics[topicname].get_simple_consumer():
            yield f"i.value.decode()"
            
    return events()

for x in get_messages("geostream"):
    print(x)
```

If you see your consumer printing out what the producer is pushing, you're good to go. If you set up your networking incorrectly, then you will notice it at this stage at the latest.

## FastAPI Apache Kafka producer

As shown in my sketch I want to wrap the producer into a [FastAPI](https://FastAPI.tiangolo.com/) endpoint. This allows for more than one entity at a time to produce messages to a topic, but also enables me to flexibly change topics that I want to produce messages to with [FastAPI](https://FastAPI.tiangolo.com/tutorial/path-params-numeric-validations/) endpoint path parameters. I opted to use [aiokafka](https://github.com/aio-libs/aiokafka) instead of pykafka to make use of FastAPIs async capabilities, but also to plague myself and get a better understanding of async programming (still pending).

FastAPIs `on_event("startup)` and `on_event("shutdown")` make the use of a aiokafka producer easy.

```python
loop = asyncio.get_event_loop()
aioproducer = AIOKafkaProducer(
    loop=loop, client_id=PROJECT_NAME, bootstrap_servers=KAFKA_INSTANCE
)

@app.on_event("startup")
async def startup_event():
    await aioproducer.start()


@app.on_event("shutdown")
async def shutdown_event():
    await aioproducer.stop()
```

Afterwards we can use the `aioproducer` in our application.

```python
@app.post("/producer/{topicname}")
async def kafka_produce(msg: ProducerMessage, topicname: str):
    await aioproducer.send(topicname, json.dumps(msg.dict()).encode("ascii"))
    response = ProducerResponse(
        name=msg.name, message_id=msg.message_id, topic=topicname
    )

    return response
```

## FastAPI Apache Kafka consumer

The Apache Kafka consumer endpoint with FastAPI turned out to be a completely different beast. I wasn't able to setup up something similar to the `pykafka` MVP. In [Flask]() you can do something like and push server sent events to whoever is calling the `/consumer/<topicname>` endpoint:

```python
@app.route('/consumer/<topicname>')
def get_messages(topicname):
    def events():
        for message in client.topics[topicname].get_simple_consumer():
            yield f"i.value.decode()"
    
    return Response(events(), mimetype="text/event-stream")
```

This way we can in the Leaflet application we can add an [EventSource and an EventListener](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) that would take incoming events and do something with it - exactly as we wanted it to be in the first place (see architecture).
I was not able to do the same with FastAPI and an async generator. That is not to say it is not possible, but my dumb ass wasn't able to figure out how and didn't bother to open an issue with FastAPI.

What I did figure out, however, is that there is a beautiful thing called [Websockets](https://FastAPI.tiangolo.com/advanced/websockets/) and that FastAPI happily supports the likes. Since FastAPI is built on-top of starlette we can use class-basedd endpoints and especially the [WebsocketEndpoint](https://www.starlette.io/endpoints/#websocketendpoint) to handle incoming `Websocket` Sessions.

```python
@app.websocket_route("/consumer/{topicname}")
class WebsocketConsumer(WebSocketEndpoint):
    async def on_connect(self, websocket: WebSocket) -> None:

        # get topicname from path until I have an alternative
        topicname = websocket["path"].split("/")[2] 

        await websocket.accept()
        await websocket.send_json({"Message": "connected"})

        loop = asyncio.get_event_loop()
        self.consumer = AIOKafkaConsumer(
            topicname,
            loop=loop,
            client_id=PROJECT_NAME,
            bootstrap_servers=KAFKA_INSTANCE,
            enable_auto_commit=False,
        )

        await self.consumer.start()

        self.consumer_task = asyncio.create_task(
            self.send_consumer_message(websocket=websocket, topicname=topicname)
        )

        logger.info("connected")

    async def on_disconnect(self, websocket: WebSocket, close_code: int) -> None:
        self.consumer_task.cancel()
        await self.consumer.stop()
        logger.info(f"counter: {self.counter}")
        logger.info("disconnected")
        logger.info("consumer stopped")

    async def on_receive(self, websocket: WebSocket, data: typing.Any) -> None:
        await websocket.send_json({"Message": data})

    async def send_consumer_message(self, websocket: WebSocket, topicname: str) -> None:
        self.counter = 0
        while True:
            data = await consume(self.consumer, topicname)
            response = ConsumerResponse(topic=topicname, **json.loads(data))
            logger.info(response)
            await websocket.send_text(f"{response.json()}")
            self.counter = self.counter + 1
```

When an application starts a websocket connection with out websocket endpoint we grab the event loop, use that to build and start the aiokafka consumer, start it and start a consumer task in the loop. Once this is set, everytime the consumer pulls a new message it is forwarded to the application through the websocket. When the leafletjs application either specifically closes the websocket or the browser is closed, we close the websocket, cancel the consumer_task and stop the consumer.

## Leaflet application

I have to admit that it's been some time that I have touched JavaScript at all, so bare with me here.

First we will declare our map and websocket connection to the `/consumer/{topicname}` endpoint.

```javascript
var map = new L.Map('map');
var linesLayer = new L.LayerGroup();
var ws = new WebSocket("ws://127.0.0.1:8003/consumer/geostream");

var osmUrl = 'https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png',
    osmAttribution = '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
    osm = new L.tileLayer(osmUrl, {maxZoom: 18, attribution: osmAttribution});

var colors = ["#8be9fd", "#50fa7b", "#ffb86c", "#ff79c6", "#bd93f9", "#ff5555", "#f1fa8c"];
lines = {};
```

When client and backend established the silent agreement to use WebSockets, we can declare what we want to do, whenever the [websocket receives a new message](https://javascript.info/websocket). Every message is an `event`. And every `event` consists of metadata and `event.data` that can be parsed with `JSON.parse()`.

```javascript
ws.onmessage = function(event) {
    console.log(event.data)
    obj = JSON.parse(event.data)

    [...]
```

One of the requirements was to display more than one entity that pushes messages through the producer, Kafka and the consumer on the map as a live-event. For the leaflet application to associate an event to an entity, I hash events by the name of the entity that is sending them. If there is a new `name` in an event, it'll be hashed into a dictionary and added as a new layer on the map. As I wanted every entity to be represented with a different color, the color will be randomly grabbed from the list of `colors` and hashed alongside the position of the event/entity.

```javascript
    [...]

    if(!(obj.name in lines)) {
      lines[obj.name] = {"latlon": []};
      lines[obj.name]["latlon"].push([obj.lat, obj.lon]);
      lines[obj.name]["config"] = {"color": colors[Math.floor(Math.random()*colors.length)]};
    }
    else {
      lines[obj.name]["latlon"].push([obj.lat, obj.lon]);
    }

    line = L.polyline(lines[obj.name]["latlon"], {color: lines[obj.name]["config"]["color"]})
    linesLayer.addLayer(line)
    map.addLayer(linesLayer);

};
```

On thing to keep in mind is, that when you zoom on the map the `bounds` will be messed up and the events will not properly draw polylines along the trajectory of the entity. To fix this I added an `zoomend` trigger:

```javascript
map.on("zoomend", function (e) { linesLayer.clearLayers() });
```

that will clear the layers until the next event arrives and then continues to draw the trajectories.

## Result

<p align="center">
<img src="/img/2020-03-geostream-kafka/geostream.gif" alt="geostream leaflet">
</p>


