## 1. Read-Heavy Systems

► Design a URL Shortener
 - ∟ Focus on: key generation, collisions, DB/caching, sharding

► Design an Image Hosting Service
 - ∟ Focus on: object storage, CDN, deduplication, resizing

► Design a Social Platform(Twitter/Facebook)
 -  ∟ Focus on: posts, timelines, denormalized storage, sharding

► Design a NewsFeed System
- ∟ Focus on: push vs pull, fanout, ranking, pagination

## 2. Write-Heavy Systems

► Design a Rate Limiter
- ∟ Focus on: token/leaky bucket, Redis, TTL

► Design a Log Collection/Analysis System
- ∟ Focus on: Kafka, ELK, partitioning, real-time query

► Design a Voting System
- ∟ Focus on: idempotency, fraud prevention, real-time aggregation

► Design a Trending Topics System
- ∟ Focus on: count-min sketch, sliding window, ranking

## 3. User-Facing Apps & Social Systems

► Design a Blogging Platform
- ∟ Focus on: data modeling, feed generation, access control

► Design OnePic(PhotoApp)
- ∟ Focus on: media storage, timelines, user-generated content

► Design Photo Tagging
- ∟ Focus on: graph relationships, image search

► Design HashTag Service
- ∟ Focus on: real-time indexing, trending detection

► Design User Affinity
- ∟ Focus on: collaborative filtering, scoring models

## 4. Search, Messaging, and Delivery

► Design a Word Dictionary
- ∟ Focus on: trie, prefix lookups

► Design a Text-Based Search Engine
- ∟ Focus on: tokenization, inverted index, ranking

► Design SQL-backed Message Broker
- ∟ Focus on: durability, ordering, delivery semantics

► Design a Distributed Task Scheduler
- ∟ Focus on: idempotency, retries, time-based triggering

► Design Recent Searches Service
- ∟ Focus on: LRU cache, user isolation

## 5. Streaming, Sync, and Media

► Design a Video Processing Pipeline
- ∟ Focus on: batch vs real-time, encoding

► Design Airline Check-in
- ∟ Focus on: concurrency, seat allocation, time-based locking

► Design Remote File Sync Service
- ∟ Focus on: delta sync, conflict detection, versioning

► Design Text-based Cricket Commentary
- ∟ Focus on: event streaming, real-time engagement

► Design “Who’s Near Me” Service
- ∟ Focus on: location sharding, frequent geo updates


## 6. Strong Consistency Systems

► Design Online Ticket Booking
- ∟ Focus on: locking, concurrency, seat/payment flows

► Design an E-Commerce Website
- ∟ Focus on: catalog, cart, order, consistency, idempotency

► Design an Online Messaging App
- ∟ Focus on: message queue, receipts, scaling chat infra

► Design a Task Management Tool
- ∟ Focus on: CRUD, auth, assignment, background jobs

## 7. Core Infrastructure & Storage

► Design a SQL-backed KV Store
- ∟ Focus on: relational schema modeling, CRUD latency tradeoffs

► Design a Superfast KV Store
- ∟ Focus on: in-memory caching, fast persistence

► Design a Faster Superfast KV Store
- ∟ Focus on: write-heavy workloads at scale

► Design S3 (Object Storage)
- ∟ Focus on: chunking, metadata, eventual consistency

► Design a Distributed Cache
- ∟ Focus on: eviction, replication, cache invalidation

## 8. Realtime & Event-Driven Systems

► Design Online/Offline Indicator
- ∟ Focus on: heartbeat, stale state detection

► Design a Realtime Database
- ∟ Focus on: websocket handling, conflict resolution

► Design Synchronized Queue Consumers
- ∟ Focus on: concurrency, message ordering, coordination

► Design Flash Sale
- ∟ Focus on: load shedding, queuing, atomic inventory

► Design Realtime Claps
- ∟ Focus on: low-latency counters, no write bottlenecks

## 9. Scheduler & Notification Services

► Design a Web Crawler
- ∟ Focus on: BFS/DFS, politeness, distributed queues

► Design a Task Scheduler
- ∟ Focus on: job queues, retry logic, cron triggers

► Design a Real-Time Notification System
- ∟ Focus on: push vs polling, webhooks, device tokens, scale

## 10. Trie / Proximity / Location Systems

► Design a Search Autocomplete
- ∟ Focus on: trie/ternary trees, frequency, typo-tolerance

► Design a Ride-Sharing App (Uber/Lyft)
- ∟ Focus on: matchmaking, real-time location, ETA, surge
