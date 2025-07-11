---
title: "DragonflyDB vs Redis"
subtitle: ""
date: 2023-08-22T08:35:52+07:00
lastmod: 2023-08-22T08:35:53+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["DevOps", "Database", "DragonflyDB", "Redis"]
categories: ["DevOps"]

lightgallery: true
---

Normally, when we think of in-memory databases, Redis and Memcached often come to mind. Today, I'd like to introduce another interesting option: Dragonfly, which claims to be a **multi-threaded Redis replacement**. So, let's test how Dragonfly compares to Redis 7.

<!--more-->

## Redis test compose

First, let's create a compose file for testing Redis.

```yaml
services:
  redis:
    image: redis
    ulimits:
      memlock: -1
    volumes:
      - redis-data:/data
    hostname: redis
    command:
      ["redis-server", "--loglevel warning", "--unixsocket", "/data/redis.sock"]

  redis_tcp:
    depends_on:
      - redis
    image: redislabs/memtier_benchmark
    volumes:
      - redis-data:/redis-data
      - ./tmp:/data
    command: ["-h", "redis", "--out-file=/data/redis-tcp.log"]

  redis_sock:
    depends_on:
      - redis
    image: redislabs/memtier_benchmark
    volumes:
      - redis-data:/redis-data
      - ./tmp:/data
    command: ["-S", "/redis-data/redis.sock", "--out-file=/data/redis-sock.log"]

volumes:
  redis-data:
```

## Dragonfly test compose

Next, the compose file for Dragonfly.

{{< admonition note "Note" >}}
Dragonfly typically requires a minimum RAM limit of 256MB per logical CPU.
{{< /admonition >}}

```yaml
services:
  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly
    ulimits:
      memlock: -1
    volumes:
      - dragonfly-data:/data
    hostname: dragonfly
    command:
      ["dragonfly", "--logtostderr", "--unixsocket", "/data/dragonfly.sock"]

  dragonfly_tcp:
    depends_on:
      - dragonfly
    image: redislabs/memtier_benchmark
    volumes:
      - dragonfly-data:/dragonfly-data
      - ./tmp:/data
    command: ["-h", "dragonfly", "--out-file=/data/dragonfly-tcp.log"]

  dragonfly_sock:
    depends_on:
      - dragonfly
    image: redislabs/memtier_benchmark
    volumes:
      - dragonfly-data:/dragonfly-data
      - ./tmp:/data
    command: ["-S", "/dragonfly-data/dragonfly.sock", "--out-file=/data/dragonfly-sock.log"]

volumes:
  dragonfly-data:
```
## TCP

Let's start with the common scenario where everyone uses TCP/IP between containers.

### Redis

```bash
docker compose -f docker-compose.redis.yml up redis_tcp 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 6446.59    | ---        | ---           | 2.82053      | 2.68700     | 5.40700     | 10.36700     | 496.50   |
| Gets      | 64395.05   | 0.00       | 64395.05      | 2.82098      | 2.68700     | 5.40700     | 10.68700     | 2508.47  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 70841.64   | 0.00       | 64395.05      | 2.82093      | 2.68700     | 5.40700     | 10.62300     | 3004.97  |

### Dragonfly

```bash
docker compose -f docker-compose.dragonfly.yml up dragonfly_tcp 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 12683.03   | ---        | ---           | 1.53450      | 0.66300     | 10.87900    | 16.12700     | 976.82   |
| Gets      | 126690.88  | 0.00       | 126690.88     | 1.51768      | 0.65500     | 10.94300    | 16.31900     | 4935.16  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 139373.91  | 0.00       | 126690.88     | 1.51921      | 0.65500     | 10.94300    | 16.31900     | 5911.97  |

## UNIX socket

Now let's try testing with UNIX sockets between containers.

### Redis

```bash
docker compose -f docker-compose.redis.yml up redis_sock 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 17540.93   | ---        | ---           | 1.04733      | 0.99900     | 1.82300     | 4.99100      | 1350.96  |
| Gets      | 175216.52  | 0.00       | 175216.52     | 1.04597      | 0.99900     | 1.82300     | 3.87100      | 6825.44  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 192757.45  | 0.00       | 175216.52     | 1.04610      | 0.99900     | 1.82300     | 4.03100      | 8176.40  |

### Dragonfly

```bash
docker compose -f docker-compose.dragonfly.yml up dragonfly_sock 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 23136.83   | ---        | ---           | 0.80438      | 0.27900     | 7.74300     | 14.39900     | 1781.94  |
| Gets      | 231114.05  | 0.00       | 231114.05     | 0.78618      | 0.27900     | 7.39100     | 12.67100     | 9002.89  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 254250.88  | 0.00       | 231114.05     | 0.78783      | 0.27900     | 7.42300     | 12.79900     | 10784.83 |

## Conclusion
With the power of multi-threading, Dragonfly performs very well (especially through UNIX sockets, where the difference from TCP/IP is noticeable). However, it's worth noting that Dragonfly doesn't currently support 100% of the Redis API. If you're using advanced or special features, you'll need to check if it can be used as a direct replacement. You can find more information in the [command-reference](https://www.dragonflydb.io/docs/command-reference/compatibility). From my testing, if you normally use Redis with only a single instance, and don't use `Graph` or `Geo location` features, primarily using `Set`, `Get`, and occasionally `PubSub` or `Stream`, you can replace it with Dragonfly. The code and connection methods remain the same.
