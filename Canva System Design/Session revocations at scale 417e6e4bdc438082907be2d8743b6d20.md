# Session revocations at scale

*How Canva keeps hundreds of millions of user sessions fast and secure*

Canva uses browser cookies to store everything we need to know about a user’s session, from their user ID to their permissions and roles. By encrypting these cookies, our gateways can trust these details without talking to a networked datastore on every request. However, if a user is logged out or their permissions change, we need to be able to revoke or update their cookies in near real time.

To enable this, each gateway needs a record of any revoked sessions. We use in-memory lookups because they’re faster and more reliable than checking a networked datastore. Since our session cookies refresh periodically, we store 12 hours of revocations in memory and rely on slower MySQL lookups during these refreshes. Tokens that need refreshing will always be checked against the database, so they don’t have to rely on the in-memory cache.

If we wanted to use S3 to cache our data, we needed to convert our sliding window of revocation records into a format suitable for object storage. We decided to partition our sliding window into 30-minute segments, each corresponding to one object in S3. We don’t need to worry about individually removing old revocations from S3, since each gateway instance can fetch only the most recent chunks. 

Each revocation contains two critical pieces of information: who it applies to (called the “principal”), and the login timestamp up to which it applies. We’re able to pack this into 16 bytes, with our revocation chunks just being a flat array of these 16-byte elements. By sorting this array, we can efficiently binary search each chunk to find revocations that apply to a specific principal. This makes the cache loading incredibly fast, since the only cost paid is the download from our S3 bucket. The gateway service can operate directly on the downloaded bytes. This also reduces the size of in-memory cache.

In practice, it turns out to be a bit more complicated, since we need to support multiple types of revocations. For example, we might want to invalidate some information cached in a user’s cookies without logging them out, or we might want to target an entire brand rather than a single user. To handle this, we needed to reserve a few bits for flags, but as long as we still sort by each revocation’s principal, we can store many types of revocations in the same flat array.

The worker continually scans our database and fetches large batches of revocations which haven’t yet been uploaded to S3. The worker then fetches the latest chunk (or creates a new one if sufficient time has passed), inserts the new revocations into the sorted array, and then re-uploads the chunk. To ensure high availability and to aid with deployments, we run multiple copies of this worker process at any given time leading to lost updates. To prevent this, we use conditional PUT requests to implement optimistic concurrency control. This includes adding a precondition on all chunk updates to assert that the chunk hasn’t changed since it was first read, and newly created chunks use similar checks to ensure another process hasn’t already created the same chunk. The PUT conditions guarantee that each read-modify-write operation only ever appends to the set of revocations in S3, ensuring interleaving executions never lose data. We also use a ZooKeeper leader election as an optimization to prevent continuous conflicts from adding load to our system. 

![image.png](Session%20revocations%20at%20scale/image.png)

How a worker task copies revocations from our database into S3

We also need to keep our in-memory caches continuously up to date with new revocations as they are created: we used S3 conditional GET requests to redownload the latest chunks only when they have actually changed.

![image.png](Session%20revocations%20at%20scale/image%201.png)

How revocations flow through our new system