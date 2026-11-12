# SnapShare: Scaling Plan

SnapShare is a photo-sharing app. Users upload photos and scroll a feed of photos from people they follow.

## 1. Assumptions

| Assumption | Value |
|---|---|
| Registered users | 10,000,000 |
| Daily active users (DAU) | 10% of registered users |
| Uploads per active user per day | 1 photo |
| Feed page views per active user per day | 50 |
| Average photo size | 2 MB |
| Thumbnail size | 50 KB (one per photo) |
| Seconds in a day | 86,400 |
| Peak traffic | 5x the average |
| Storage units | Decimal (1 TB = 1,000,000 MB) |

Additional simplifications: traffic is spread evenly across the day for the average figures, the peak multiplier covers busy hours, and photos are kept forever (no deletion).

**Daily active users:** 10,000,000 x 10% = **1,000,000 DAU**.

## 2. Estimates

### Uploads per second

- Uploads per day: 1,000,000 x 1 = 1,000,000
- Average: 1,000,000 / 86,400 = **about 11.6 uploads/s**
- Peak (5x): **about 58 uploads/s**

### Feed views per second

- Feed views per day: 1,000,000 x 50 = 50,000,000
- Average: 50,000,000 / 86,400 = **about 579 views/s**
- Peak (5x): **about 2,894 views/s (roughly 2,900/s)**

### Photo storage per year

- Original photos per day: 1,000,000 x 2 MB = 2 TB
- Thumbnails per day: 1,000,000 x 50 KB = 0.05 TB
- Total per day: 2.05 TB
- Per year: 2.05 TB x 365 = **about 748 TB per year (about 0.75 PB)**
  - Originals: 730 TB
  - Thumbnails: 18.25 TB

Database metadata (user ids, captions, timestamps, follow relationships) is tiny compared with this, so the photo files dominate storage. Replication or backups would multiply the figure, which is why object storage is used (see section 4).

## 3. Read-heavy or write-heavy?

**The system is read-heavy.** There are about 50 feed views for every 1 upload (579 views/s versus 11.6 uploads/s on average, 2,894 versus 58 at peak). Each feed view also loads many images, so the real read load on photo files is higher still.

What this means for the design:

- **Optimise for reads first.** Use a CDN so images are served from locations near the user, and a cache so the feed does not hit the database every time.
- **Add a read replica.** Feed queries go to replicas, so the primary database only handles the (much smaller) write load.
- **Make writes cheap and asynchronous.** Uploads are only about 58/s at peak, so a single write path is fine. Slow work (thumbnail creation) is moved out of the request into a queue.
- **Accept slightly stale feeds.** Because reads dominate, it is worth caching feed data for seconds or minutes, even though a new photo may appear a little late.

## 4. Why photos do not belong in the database

Storing 2 MB image files as database blobs causes several problems:

- **Size and cost:** about 748 TB per year would sit on expensive database disks. Database storage is much pricier than object storage.
- **Performance:** large blobs fill the database cache and slow the queries that really need the database (feed, follows, likes).
- **Scaling and backups:** a database with hundreds of terabytes is slow to back up, replicate and restore, and replicas would each need a full copy.
- **Serving:** a database cannot hand files to browsers through a CDN efficiently, so every image view would load the database.

**Where photos go instead:** in **object storage** (for example Amazon S3 or an equivalent). It is cheap, effectively unlimited, durable (data is copied across machines automatically) and designed to serve files over HTTP, which makes it a natural source for the CDN. The database keeps only the *metadata*: a photo id, owner, caption, timestamp, and the **object key / URL** that points to the file in object storage.

## 5. Architecture diagram

```
                         +----------------------------+
                         |   Users (mobile / web)     |
                         +-------------+--------------+
                                       |
               images (GET)            |            API calls (feed, upload)
          +----------------------------+----------------------------+
          |                                                         |
          v                                                         v
  +---------------+                                       +-----------------+
  |      CDN      |                                       |  Load Balancer  |
  | (edge caches) |                                       +--------+--------+
  +-------+-------+                                                |
          | cache miss                         +-------------------+-------------------+
          |                                    |                   |                   |
          |                              +-----v-----+       +-----v-----+       +-----v-----+
          |                              | App Server|       | App Server|       | App Server|
          |                              |     1     |       |     2     |  ...  |     N     |
          |                              +--+--+--+--+       +-----------+       +-----------+
          |                                 |  |  |            (all servers talk to the same
          |                                 |  |  |             components shown below)
          |        +------------------------+  |  +----------------------------+
          |        | read feed/metadata        | write metadata                  | enqueue job
          |        v                           v                                 v
          |  +-----------+  miss        +--------------+                   +-----------+
          |  |   Cache   +------+       |  Primary DB  |                   |   Queue   |
          |  |  (Redis)  |      |       |   (writes)   |                   +-----+-----+
          |  +-----------+      |       +------+-------+                         |
          |                     v              | replication                     | job
          |              +--------------+      |                                 v
          |              | Read Replica |<-----+                       +-------------------+
          |              | (feed reads) |                              |  Thumbnail Worker |
          |              +--------------+                              |  (resize to 50KB) |
          |                                                            +---------+---------+
          |                                                                      |
          |    +-----------------------------------------------------+           |
          +--->|                   Object Storage                    |<----------+
               |      originals (2 MB)  +  thumbnails (50 KB)        |  read original,
               +-----------------------------------------------------+  save thumbnail
                          ^
                          | App Server saves the original here on upload
```

Simplified request paths:

```
View feed:   User -> Load Balancer -> App Server -> Cache -> (miss) Read Replica
             User -> CDN -> (miss) Object Storage          (images)

Upload:      User -> Load Balancer -> App Server -> Object Storage (original)
                                                 -> Primary DB (metadata)
                                                 -> Queue -> Thumbnail Worker
                                                          -> Object Storage (thumbnail)
```

## 6. Components: one sentence each

- **CDN:** serves popular photos and thumbnails from edge locations close to the user, which cuts latency and keeps the heavy image traffic away from our servers and storage.
- **Load balancer:** spreads incoming requests across many app servers so no single server is overloaded and a failed server can be skipped.
- **App servers:** run the SnapShare logic (login, upload, feed building) and are stateless, so we can add more of them to handle growing traffic.
- **Cache:** keeps hot data such as recent feeds and photo metadata in memory so most feed requests avoid the slower database.
- **Database (primary):** stores reliable structured metadata (users, follows, photo records, captions) and handles all writes.
- **Read replica:** holds a continuously updated copy of the database that serves feed reads, so the read-heavy load does not slow down writes on the primary.
- **Object storage:** stores the photo and thumbnail files cheaply and durably at hundreds of terabytes, which a database cannot do economically.
- **Queue:** holds thumbnail jobs so uploads can finish quickly while the slow resizing work happens in the background and survives traffic spikes.
- **Thumbnail worker:** takes jobs from the queue, creates the 50 KB thumbnail and saves it, so feeds can load small images instead of 2 MB originals.

## 7. Upload flow, step by step

1. The user picks a photo in the app and taps upload. The client sends the request to SnapShare.
2. The **load balancer** receives the request and forwards it to a healthy **app server**.
3. The app server authenticates the user and validates the file (type is an image, size is within limits).
4. The app server stores the original (about 2 MB) in **object storage** under a unique key, for example `photos/{user_id}/{photo_id}.jpg`. (A common variation is to have the client upload directly to object storage using a short-lived signed URL, which keeps large files off the app servers.)
5. The app server writes a metadata row to the **primary database**: photo id, user id, caption, timestamp, object key, and thumbnail status = `pending`.
6. The app server places a "create thumbnail" job (containing the photo id and object key) on the **queue**.
7. The app server returns "upload successful" to the user. The user does not wait for the thumbnail.
8. A **thumbnail worker** picks up the job from the queue, downloads the original from object storage, resizes it to a thumbnail of about 50 KB, and saves it to object storage (for example `thumbs/{photo_id}.jpg`).
9. The worker updates the database row: thumbnail status = `ready`, and the thumbnail key. If the job fails, it is retried; after repeated failure it goes to a dead-letter queue for investigation.
10. The **cache** entries for followers' feeds are invalidated or updated so the new photo appears. Followers then load the thumbnail through the **CDN**, which fetches it from object storage on the first request and caches it after that.

## 8. Trade-offs

1. **Cache freshness vs speed (eventual consistency).** Caching feeds and reading from a replica makes the common case very fast and cheap, but followers may see a new photo a few seconds late, and replicas lag slightly behind the primary. For a photo feed, this delay is acceptable. For something like a bank balance it would not be.
2. **Asynchronous thumbnails vs immediate availability.** Using a queue and worker keeps uploads fast and absorbs spikes (peak 58 uploads/s), but a thumbnail is not ready the instant the upload finishes. The app must handle a "pending" state, for example by showing a placeholder or the original briefly. It also adds more components to run and monitor.
3. **Object storage plus CDN vs cost and complexity.** Moving files out of the database is far cheaper and scales to about 748 TB per year, but it adds more moving parts, and the database and object storage can drift out of sync (for example a file saved but its database row failed). We handle this with careful ordering and cleanup jobs. CDN bandwidth also has a cost, so it is worth tuning cache lifetimes.
4. **Read replica vs operational overhead.** A replica protects the primary from read load, but it costs extra money, can serve slightly stale data, and needs failover planning. A single database would be simpler but would be the first thing to break at about 2,900 feed views/s at peak.
5. **Keeping originals and thumbnails vs storage cost.** Thumbnails add only about 18 TB per year on top of 730 TB of originals (about 2.5%), a small price for much faster feeds. Storing only originals would save that space but make feeds slow and bandwidth-hungry.

## 9. Summary

- 1,000,000 DAU; about 11.6 uploads/s (58 peak); about 579 feed views/s (2,894 peak); about 748 TB of photos per year.
- Read-heavy (about 50 to 1), so the design leans on the CDN, cache and read replica.
- Photos live in object storage; the database holds only metadata and links.
- Thumbnail creation is asynchronous through a queue and worker, so uploads stay fast.
