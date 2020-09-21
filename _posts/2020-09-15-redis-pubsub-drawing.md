---
title: Redis PubSub + WebRTC Signaling
published: true
description: Where we update our realtime drawing app in canvas to any number of signaling servers by using Redis PubSub.
tags: [javascript,tutorial,webdev,nodejs]
series: Realtime collaborative drawing with canvas and WebRTC
---

* [Part 1: Realtime collaborative drawing with canvas and WebRTC](/2020/09/05/collaborative-drawing-webrtc-canvas/)
* [Part 2: Server Sent Events + WebRTC Mesh Networks](/2020/09/08/collaborative-drawing-sse-webrtc/)
* [Part 3: Simulating webkit force, canvas color swatches](/2020/09/10/color-swatch-webkit-force/)
* **Part 4: Redis PubSub + WebRTC Signaling**

In any system that involves real-time communication, open connections, and messages that need to be routed among peers - you tend to run into the problem that not all of your connections will be able to run on a single server. What we need to do, instead, is setup a system that can route messages to any number of servers maintaining any number of connections.

In a previous article, our [drawing program](/2020/09/08/collaborative-drawing-sse-webrtc/) was recently refactored to use [server sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#Event_stream_format) by maintaining a connection open. However, if we introduced another web server to create some load balancing, we run into the problem that the client connections may not be accessible across servers.

We can solve this by having a shared communication server/cluster that can handle routing all these messages for us. To do this we will be using the [publisher-subscriber](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) pattern and we will leverage [redis](https://redis.io/) to get this job done.

## Redis

> Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. It supports data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs, geospatial indexes with radius queries and streams. Redis has built-in replication, Lua scripting, LRU eviction, transactions and different levels of on-disk persistence, and provides high availability via Redis Sentinel and automatic partitioning with Redis Cluster

Redis is an incredible project, it's [extremely fast](https://redis.io/topics/benchmarks) and uses minimal cpu. It is a work of art that the project has also remained backwards compatible since version 1. The maintainer [antirez](https://twitter.com/antirez) (who recently has decided to move on) has created this project over a number of years and built it into something truly incredible. Redis supports just about all the data structures as first-class features and operations.

- [string manipulation](https://redis.io/commands#string)
- [hashes](https://redis.io/commands#hash)
- [hyperloglog](https://redis.io/commands#hyperloglog)
- [sets](https://redis.io/commands#set)
- [sorted sets](https://redis.io/commands#sorted_set)
- [geospatial indexes](https://redis.io/commands#geo)
- [streams](https://redis.io/commands#stream)
- [pubsub](https://redis.io/commands#pubsub)

And it even supports [cluster](https://redis.io/commands#cluster). You can even use it as a [last-write-wins](https://github.com/soundcloud/roshi) CRDT using [roshi](https://github.com/soundcloud/roshi). At my work, we've used just about all of these features from queues, hyperloglogs, sorted sets, caching. In a previous project, I once used redis to build a [click stream](https://en.wikipedia.org/wiki/Click_path) system using an sort of [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) model.

## Redis PubSub

We're going to use a small feature of redis called [pubsub](https://redis.io/commands#pubsub) to route our messages between connections on our server. Assuming you have a redis-server setup. You'll need to add `redis` as a dependency to our drawing app.

```
npm install --save redis bluebird
```

We're going to use `bluebird` to be able to [promisifyAll](http://bluebirdjs.com/docs/api/promise.promisifyall.html) to all the redis client functions. This will help us write our code with [async/await](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Async_await) instead of a number of callbacks.

## /connect to SSE and Subscribe to Redis

Recall that our express server was simply keeping an in-memory cache of both connections and channels. We're going to first update our `/connect` function to instead **subscribe** to messages received from a redis *pubsub* client. To do this, we'll update the client creation code and add a `redis.createClient`. Then subscribe to messages received to our particular client id via `redis.subscribe('messages:' + client.id)`. Whenever we receive messages via `redis.on('message', (channel, message) => ...)` we can simply emit them back to the server sent event stream.

```javascript
app.get('/connect', auth, (req,res) => {
    if (req.headers.accept !== 'text/event-stream') {
        return res.sendStatus(404);
    }

    // write the event stream headers
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.flushHeaders();

    // setup a client
    let client = {
        id: req.user.id,
        user: req.user,
        redis: redis.createClient(),
        emit: (event, data) => {
            res.write(`id: ${uuid.v4()}\n`);
            res.write(`event: ${event}\n`);
            res.write(`data: ${JSON.stringify(data)}\n\n`);
        }
    };

    // cache the current connection until it disconnects
    clients[client.id] = client;

    // subscribe to redis events for user
    client.redis.on('message', (channel, message) => {
        let msg = JSON.parse(message);
        client.emit(msg.event, msg.data);
    });
    client.redis.subscribe(`messages:${client.id}`);

    // emit the connected state
    client.emit('connected', { user: req.user });

    // ping to the client every so often
    setInterval(() => {
        client.emit('ping');
    }, 10000);

    req.on('close', () => {
        disconnected(client);
    });
});
```

Also notice that I've added an interval to `ping` the client once every 10 seconds or so. This may not be completely necessary, but I add it to make sure
our connection state doesn't inadvertently get cut off for whatever reason.

## Peer Join, Peer Signaling

The only other functions we need to change are *when a peer joins the room*, *when a peer is sending a signal message to another peer*, and *when a peer disconnects from the server*. The other functions like *auth*, *:roomId* remain the same. Let's update the join function below. Note, we will need to keep track of a redis client that the server for general purpose redis communication.

```javascript
const redisClient = redis.createClient();

app.post('/:roomId/join', auth, async (req, res) => {
    let roomId = req.params.roomId;

    await redisClient.saddAsync(`${req.user.id}:channels`, roomId);

    let peerIds = await redisClient.smembersAsync(`channels:${roomId}`);
    peerIds.forEach(peerId => {
        redisClient.publish(`messages:${peerId}`, JSON.stringify({
            event: 'add-peer',
            data: {
                peer: req.user,
                roomId,
                offer: false
            }
        }));
        redisClient.publish(`messages:${req.user.id}`, JSON.stringify({
            event: 'add-peer',
            data: {
                peer: { id: peerId },
                roomId,
                offer: true
            }
        }));
    });

    await redisClient.saddAsync(`channels:${roomId}`, req.user.id);
    return res.sendStatus(200);
});
```

In order to keep track of who is in a particular **roomId**, we will make use of [redis sets](https://redis.io/commands#set) and add the room id to the current user's set of channels. Following this, we lookup what members are in the `channels:{roomId}` and iterate over the peer ids. For each peer id, we effectively will be routing a message to that peer that the current user has joined, and we will route the peer id to the *request.user*. Finally, we add our *request.user* to the `channels:{roomId}` set in redis.

Next, let's update the relay code. This will be even simpler since all we have to do is just **publish** the message to that peer id.

```javascript
app.post('/relay/:peerId/:event', auth, (req, res) => {
    let peerId = req.params.peerId;
    let msg = {
        event: req.params.event,
        data: {
            peer: req.user,
            data: req.body
        }
    };
    redisClient.publish(`messages:${peerId}`, JSON.stringify(msg));
    return res.sendStatus(200);
});
```

## Disconnect

Disconnect is a bit more involved, since we have to *clean up the rooms that the user is in*, then iterate over those rooms to *get the list of peers in those rooms*, then we must *signal to each peer in those rooms that the peer has disconnected*.

```javascript
async function disconnected(client) {
    delete clients[client.id];
    await redisClient.delAsync(`messages:${client.id}`);

    let roomIds = await redisClient.smembersAsync(`${client.id}:channels`);
    await redisClient.delAsync(`${client.id}:channels`);

    await Promise.all(roomIds.map(async roomId => {
        await redisClient.sremAsync(`channels:${roomId}`, client.id);
        let peerIds = await redisClient.smembersAsync(`channels:${roomId}`);
        let msg = JSON.stringify({
            event: 'remove-peer',
            data: {
                peer: client.user,
                roomId: roomId
            }
        });
        await Promise.all(peerIds.forEach(async peerId => {
            if (peerId !== client.id) {
                await redisClient.publish(`messages:${peerId}`, msg);
            }
        }));
    }));
}
```

![SSE Redis PubSub Drawing](/assets/draw-19xtftv.gif)

Success!

## Conclusion

Now that we've added support for Redis PubSub, we can scale our service to any number of server nodes (so long as we have a redis server that we can communicate between). Connections will remain open per node process while the messages and channel communication will be routed through redis to ensure that every message is delivered through the proper server sent event stream.

Thanks for following along!

Cheers! 🍻

Feel free to download the code for this series here [https://github.com/nyxtom/drawing-webrtc](https://github.com/nyxtom/drawing-webrtc)
