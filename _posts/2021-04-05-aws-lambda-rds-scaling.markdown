---
layout: post
title: "Lessons learned on concurrency limits of AWS Lambda backed by RDS database"
date: 2021-04-05 19:42:09 +0200
categories: engineering
tags:
  [programming, database, lambda, rds, postgres, serverless, connection pool]
---

## The problem

One of the services I'm maintaining is built on AWS Lambda and uses PostgreSQL database hosted on AWS RDS. A while ago the database started to run out of available connections during load spikes. Since the service was a critical part of the platform, I dived into debugging.

Simple load testing on staging environment with Apache Bench revealed that even after load abates the number of database connections only goes down after 4-5 more minutes, and the server is not able to process new requests during that time either.

## Possible solutions

Soon the cause started to seem obvious - spikes of load result in concurrent executions, forcing Lambda to create new instances. In order to foster database connections reuse each instance maintained its own pool of connections, storing a reference in the global function scope. That lead to database connection exhaustion at a certain concurrency level.

I saw 2 ways to address the issue: either using reserved Lambda concurrency to prevent it from going higher than the exhaustion limit or somehow making an instance to wait for a connection to become available rather than to through connection error right away.

The former was swept aside: such behavior would mean making clients aware that they should possibly retry. Also that didn't seem to being able to address the entirely - as mentioned above, the service was unresponsible for several minutes after the load abates when concurrency is essentially zero.

## A hope for RDS Proxy

While looking for a way to implement the second approach I found out that there is a AWS service exactly for that - RDS Proxy.

As stated in the description, it takes care of database connection entirely - maintains a pool of connections to the database and essentially multiplexes incoming connections to that pool. If no db connection is available to handle a request, it waits for up to a specified timeout and responds with an error.

I quickly set up the proxy, adjusted connection string and ran tests again, hoping for the best. Unfortunately it did not change anything in the results.

That did not make any sense until I found the following in the proxy logs:

```
The client session was pinned to the database connection
[dbConnection=<N>] for the remainder of the session.
The proxy can't reuse this connection until the session ends.
Reason: A parse message was detected.
```

In other words that means that each incoming connection is associated with exactly one database connection and that one-to-one relation makes proxy useless.

I tried to play with timeouts settings - both proxy and pooling library. No outcome.

## Disabling connection reuse to the rescue

Such results combined with the fact that the service was unresponsible even after the load spike gave me an idea. It looks like Lambda creates a set of instances according to concurrency and keeps them alive for several minutes, but apparently puts them to sleep much earlier. I don't have another explanation why it refuses to process new events with existing instances.

Having assumed that I had no other option that to try disabling connection reuse: as the presence of useless stale instances is out of our control, the only way to avoid connection pool exhaustion is to avoid having an active connection in them.

Once the change was deployed cheery load testing results followed: processing latency increased by around 20ms - apparently a penalty for TCP handshake on each request, but there were no failed requests. Neither there was a spike in database connections even with increased several times concurrency during load testing.
