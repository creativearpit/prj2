# 03 — Failures, Edge Cases & System Limitations

> **Scope:** What happens when things break. Every failure mode, race condition, and scaling limit in the SnapGallery backend — analyzed against the exact code, with production-grade fixes and code samples.

---

## Table of Contents

1. [Orphaned Images: Cloudinary Success + MongoDB Failure](#1-orphaned-images-cloudinary-success--mongodb-failure)
2. [Atomicity Gaps: The Sequential `save()` Problem](#2-atomicity-gaps-the-sequential-save-problem)
3. [Out of Memory (OOM) Crashes: `memoryStorage` Under Load](#3-out-of-memory-oom-crashes-memorystorage-under-load)
4. [Race Conditions in `toggleLike`](#4-race-conditions-in-togglelike)
5. [Orphaned Cloudinary Images on Post Deletion](#5-orphaned-cloudinary-images-on-post-deletion)
6. [Unbounded Array Growth: The 16 MB Document Limit](#6-unbounded-array-growth-the-16-mb-document-limit)
7. [ReDoS Vulnerability in User Search](#7-redos-vulnerability-in-user-search)
8. [JWT Token Theft and No Revocation](#8-jwt-token-theft-and-no-revocation)
9. [Missing Input Validation & Sanitization](#9-missing-input-validation--sanitization)
10. [Missing Rate Limiting](#10-missing-rate-limiting)
11. [Database Connection Failure Handling](#11-database-connection-failure-handling)
12. [Missing MIME Type Validation](#12-missing-mime-type-validation)
13. [Comprehensive Fix Summary](#13-comprehensive-fix-summary)

---

## 1. Orphaned Images: Cloudinary Success + MongoDB Failure

### The Exact Code Path That Creates the Problem

**File:** [postController.js#L17-L35](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L17-L35)

```js
export const createPost = async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ message: 'Image is required' });
    }

    // STEP A: Upload to Cloudinary — THIS SUCCEEDS
    const result = await uploadToCloudinary(req.file.buffer);

    // STEP B: Create MongoDB document — THIS FAILS
    const post = await Post.create({
      author: req.user._id,
      imageUrl: result.secure_url,
      caption: req.body.caption || ''
    });

    const populated = await post.populate('author', 'username profilePic');
    res.status(201).json(populated);
  } catch (err) {
    // STEP C: Error handler — catches the DB error, but does NOT clean up Cloudinary
    res.status(500).json({ message: err.message });
  }
};
```

### The Failure Scenario

**What happens:**
1. **Line 23** — `uploadToCloudinary()` succeeds. The image is now permanently stored on Cloudinary. It has a `public_id` (e.g., `SnapGallery/abc123xyz`) and a `secure_url`. Cloudinary has accepted and stored the bytes.

2. **Line 25** — `Post.create()` **throws an error**. Possible reasons:
   - MongoDB is temporarily unreachable (network partition, Atlas maintenance window)
   - Mongoose validation fails (e.g., if `author` is somehow invalid)
   - MongoDB write concern timeout (if using `w: "majority"` on a replica set with degraded nodes)
   - Server crashes between Line 23 and Line 25 (process kill, OOM, unhandled rejection in another async operation)

3. **Line 33** — The `catch` block sends a 500 response to the client. The client sees an error.

4. **Result:** The image lives on Cloudinary **with no corresponding MongoDB document**. There is no reference to it in the database. It consumes Cloudinary storage quota indefinitely. If the image contains sensitive content, it remains accessible via its direct URL forever.

### Why the Current Code Cannot Roll Back

The `catch` block at Line 33 is a generic error handler:
```js
catch (err) {
  res.status(500).json({ message: err.message });
}
```

It has **no awareness** of the Cloudinary upload result. The `result` variable (which contains `result.public_id` needed for cleanup) is scoped inside the `try` block and is accessible in the `catch` block, but the code doesn't use it for cleanup.

### Fix 1: Compensating Transaction (Immediate Cleanup)

```js
export const createPost = async (req, res) => {
  let cloudinaryResult = null;  // Track the upload result for potential rollback

  try {
    if (!req.file) {
      return res.status(400).json({ message: 'Image is required' });
    }

    // Step 1: Upload to Cloudinary
    cloudinaryResult = await uploadToCloudinary(req.file.buffer);

    // Step 2: Attempt MongoDB save
    try {
      const post = await Post.create({
        author: req.user._id,
        imageUrl: cloudinaryResult.secure_url,
        cloudinaryPublicId: cloudinaryResult.public_id,  // STORE THIS!
        caption: req.body.caption || ''
      });

      const populated = await post.populate('author', 'username profilePic');
      res.status(201).json(populated);
    } catch (dbError) {
      // Step 3: MongoDB failed — roll back Cloudinary upload
      console.error('DB save failed, rolling back Cloudinary upload:', dbError.message);
      
      try {
        await cloudinary.uploader.destroy(cloudinaryResult.public_id);
        console.log('Cloudinary rollback successful:', cloudinaryResult.public_id);
      } catch (cleanupError) {
        // Even the cleanup failed — log for manual intervention
        console.error('CRITICAL: Failed to clean up Cloudinary image:', {
          public_id: cloudinaryResult.public_id,
          secure_url: cloudinaryResult.secure_url,
          cleanupError: cleanupError.message
        });
        // In production: Send alert to monitoring system (Datadog, PagerDuty)
      }

      res.status(500).json({ message: 'Failed to create post. Please try again.' });
    }
  } catch (uploadError) {
    // Cloudinary upload itself failed — no cleanup needed
    res.status(500).json({ message: 'Image upload failed. Please try again.' });
  }
};
```

**Key changes:**
1. `cloudinaryResult` is declared in the outer scope for cleanup access.
2. Nested try/catch separates Cloudinary failures from DB failures.
3. On DB failure, `cloudinary.uploader.destroy(public_id)` is called.
4. Even the cleanup has a try/catch — if Cloudinary delete also fails, it's logged for manual intervention.
5. The `public_id` is stored in the Post schema (requires schema update).

### Fix 2: Periodic Garbage Collection (Cron-Based Reconciliation)

```js
// reconciliation-job.js (run via node-cron or external scheduler)
import cloudinary from '../utils/cloudinary.js';
import Post from '../models/Post.js';

export const reconcileOrphanedImages = async () => {
  console.log('Starting orphaned image reconciliation...');
  
  // Get all images from Cloudinary's SnapGallery folder
  const { resources } = await cloudinary.api.resources({
    type: 'upload',
    prefix: 'SnapGallery/',
    max_results: 500
  });

  // Get all imageUrls stored in MongoDB
  const posts = await Post.find({}, 'imageUrl').lean();
  const dbUrls = new Set(posts.map(p => p.imageUrl));

  // Find orphans: on Cloudinary but not in MongoDB
  const orphans = resources.filter(r => !dbUrls.has(r.secure_url));

  console.log(`Found ${orphans.length} orphaned images out of ${resources.length} total`);

  // Delete orphans
  for (const orphan of orphans) {
    await cloudinary.uploader.destroy(orphan.public_id);
    console.log(`Deleted orphan: ${orphan.public_id}`);
  }

  console.log('Reconciliation complete');
};
```

**Trade-off:** This approach doesn't prevent orphans — it cleans them up periodically. There's a window between orphan creation and cleanup where storage is wasted. However, it catches orphans from ALL failure scenarios, including server crashes.

### Fix 3: Two-Phase Commit Pattern

```js
// Step 1: Create Post with pending status
const post = await Post.create({
  author: req.user._id,
  status: 'pending',
  caption: req.body.caption || ''
});

// Step 2: Upload to Cloudinary
const result = await uploadToCloudinary(req.file.buffer);

// Step 3: Update Post with URL and publish
post.imageUrl = result.secure_url;
post.cloudinaryPublicId = result.public_id;
post.status = 'published';
await post.save();

// Cleanup job: Delete posts with status 'pending' older than 10 minutes
// These represent uploads that were interrupted
```

---

## 2. Atomicity Gaps: The Sequential `save()` Problem

### The Exact Code That Is Not Atomic

**File:** [userController.js#L25-L26](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L25-L26)

```js
await currentUser.save();   // ← Write #1: Updates User A's document
await targetUser.save();    // ← Write #2: Updates User B's document
```

### Why This Is NOT Truly Atomic

**What "atomic" means:** An operation is atomic if it either **fully completes** or **fully rolls back**. There is no intermediate state observable by any other operation.

**The current code:** Two separate `save()` calls are two separate MongoDB operations. Between them, there exists a **window of inconsistency**.

### Failure Scenario: Corrupted Social Graph

**Timeline of events when User A follows User B:**

```
T=0ms  : currentUser.following = [B._id]  (push in memory)
T=0ms  : targetUser.followers = [A._id]   (push in memory)
T=1ms  : await currentUser.save()          → MongoDB acknowledges ✅
T=2ms  : ---- SERVER CRASH (process.exit, OOM, power failure) ----
T=???  : targetUser.save() NEVER EXECUTES  → MongoDB never updated ❌
```

**Resulting state:**
- User A's `following` array: `[B._id]` ← A thinks they follow B
- User B's `followers` array: `[]` ← B doesn't know A follows them

**Consequences:**
1. **Feed inconsistency:** A's feed includes B's posts (because A's `following` contains B). But B's profile shows 0 followers (or at least doesn't list A).
2. **Toggle logic breaks:** If A tries to "unfollow" B, `currentUser.following.includes(targetUser._id)` returns `true`, so the code calls `.pull()` on both arrays. A's following is fixed, but B's followers `.pull(A._id)` is a no-op (A isn't there). The inconsistency persists.
3. **UI glitch:** B's profile page shows a different follower count than reality.

### Concurrent Request Scenario (Race Condition)

Even without crashes, concurrent requests can cause inconsistencies:

```
--- User A follows User B ---
T=0ms  : Request 1 loads currentUser (A) with following=[]
T=1ms  : Request 1 loads targetUser (B) with followers=[]

--- User C also follows User B, simultaneously ---
T=2ms  : Request 2 loads targetUser (B) with followers=[]  ← STALE READ

T=3ms  : Request 1 saves A (following=[B]) ✅
T=4ms  : Request 1 saves B (followers=[A]) ✅

T=5ms  : Request 2 saves C (following=[B]) ✅
T=6ms  : Request 2 saves B (followers=[C]) ✅  ← OVERWRITES Request 1's change!
```

**Result:** B's `followers` = `[C]` instead of `[A, C]`. User A's follow of B is **silently lost** from B's perspective.

**Root cause:** The read-modify-write pattern without any concurrency control. Both requests read B's document before either writes, then the second write overwrites the first.

### Fix 1: MongoDB Multi-Document Transactions

```js
import mongoose from 'mongoose';

export const toggleFollow = async (req, res) => {
  // Start a session
  const session = await mongoose.startSession();
  
  try {
    // Begin transaction
    session.startTransaction();
    
    if (req.params.id === req.user._id.toString()) {
      return res.status(400).json({ message: 'Cannot follow yourself' });
    }

    // All reads within the transaction see a consistent snapshot
    const targetUser = await User.findById(req.params.id).session(session);
    if (!targetUser) {
      await session.abortTransaction();
      return res.status(404).json({ message: 'User not found' });
    }

    const currentUser = await User.findById(req.user._id).session(session);
    const isFollowing = currentUser.following.includes(targetUser._id);

    if (isFollowing) {
      currentUser.following.pull(targetUser._id);
      targetUser.followers.pull(currentUser._id);
    } else {
      currentUser.following.push(targetUser._id);
      targetUser.followers.push(currentUser._id);
    }

    // Both saves are within the transaction
    await currentUser.save({ session });
    await targetUser.save({ session });

    // Commit — both writes are applied atomically
    await session.commitTransaction();

    res.json({
      isFollowing: !isFollowing,
      followersCount: targetUser.followers.length,
      followingCount: targetUser.following.length
    });
  } catch (err) {
    // Abort — neither write is applied
    await session.abortTransaction();
    res.status(500).json({ message: err.message });
  } finally {
    // Always end the session
    session.endSession();
  }
};
```

**Requirements for transactions:**
- MongoDB must be running as a **replica set** (even a single-node replica set works). Standalone MongoDB does not support multi-document transactions.
- MongoDB Atlas (which SnapGallery likely uses based on the `MONGODB_URI` env var) always runs as a replica set, so this works out of the box.

**Performance impact:** Transactions add ~2-5ms overhead per operation due to the write-ahead log and distributed commit protocol. For a social network's follow/unfollow action (not a high-frequency operation), this is negligible.

### Fix 2: Atomic `$addToSet` / `$pull` with `bulkWrite`

```js
export const toggleFollow = async (req, res) => {
  try {
    if (req.params.id === req.user._id.toString()) {
      return res.status(400).json({ message: 'Cannot follow yourself' });
    }

    const targetUser = await User.findById(req.params.id);
    if (!targetUser) {
      return res.status(404).json({ message: 'User not found' });
    }

    const currentUser = await User.findById(req.user._id);
    const isFollowing = currentUser.following.includes(targetUser._id);

    if (isFollowing) {
      // UNFOLLOW using atomic operators
      await User.bulkWrite([
        {
          updateOne: {
            filter: { _id: currentUser._id },
            update: { $pull: { following: targetUser._id } }
          }
        },
        {
          updateOne: {
            filter: { _id: targetUser._id },
            update: { $pull: { followers: currentUser._id } }
          }
        }
      ]);
    } else {
      // FOLLOW using atomic operators
      await User.bulkWrite([
        {
          updateOne: {
            filter: { _id: currentUser._id },
            update: { $addToSet: { following: targetUser._id } }
          }
        },
        {
          updateOne: {
            filter: { _id: targetUser._id },
            update: { $addToSet: { followers: currentUser._id } }
          }
        }
      ]);
    }

    // Reload for response
    const updatedTarget = await User.findById(targetUser._id);
    
    res.json({
      isFollowing: !isFollowing,
      followersCount: updatedTarget.followers.length,
      followingCount: updatedTarget.following.length
    });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**Why `bulkWrite` is better than sequential `save()`:**
1. **Single network roundtrip.** Both operations are sent to MongoDB in one command. The server processes them sequentially but without a network hop between them, reducing the inconsistency window.
2. **`$addToSet` is idempotent.** If called twice (retry scenario), the same ObjectId won't be added twice. This prevents duplicate entries in the arrays.
3. **`$pull` is also idempotent.** Pulling a non-existent value is a no-op, not an error.
4. **No Mongoose middleware triggers.** Unlike `.save()`, `bulkWrite` bypasses Mongoose pre-save hooks. This is actually desired here — we don't want password re-hashing to trigger when modifying the `following` array.

**Why it's still not perfectly atomic:** `bulkWrite` without a transaction is NOT atomic across documents. If the server crashes between the two `updateOne` operations within `bulkWrite`, the inconsistency can still occur. However, the window is microseconds instead of milliseconds.

---

## 3. Out of Memory (OOM) Crashes: `memoryStorage` Under Load

### The Exact Mechanism

**File:** [posts.js (routes)#L7](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js#L7)

```js
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });
```

### What Happens to the Node.js Event Loop and RAM

**Scenario:** 50 users upload 10 MB images concurrently.

Wait — the current limit is 5 MB (`fileSize: 5 * 1024 * 1024`). So let's analyze two cases:

**Case A: 50 concurrent 5 MB uploads (within current limits)**

```
Memory calculation:
- Base Node.js process: ~50 MB (V8 runtime, loaded modules, Express middleware)
- 50 × 5 MB buffers: 250 MB
- Buffer duplication during stream.end(): Up to 50 × 5 MB additional (250 MB peak)
- Mongoose document overhead per request: ~1 KB × 50 = negligible
- Express request/response objects: ~2 KB × 50 = negligible

Peak memory: ~550 MB

On a 512 MB free-tier instance (Render, Heroku):
→ Process exceeds memory limit
→ Linux OOM killer sends SIGKILL to the Node.js process
→ ALL 50 in-flight requests are terminated without response
→ Any images already uploaded to Cloudinary become orphans
→ Container restarts, subsequent requests are served
```

**Case B: 50 concurrent 10 MB uploads (if the limit were raised)**

```
Memory calculation:
- 50 × 10 MB buffers: 500 MB
- Buffer duplication during stream: up to 500 MB additional
- Peak memory: ~1.05 GB

On a free-tier instance: Instant OOM
On a 1.5 GB instance (Node.js default V8 heap): 
→ V8 triggers aggressive garbage collection
→ GC pause: 200-500ms (stop-the-world)
→ ALL concurrent requests experience latency spikes during GC
→ Event loop is frozen during GC — no new requests processed
→ Clients may timeout and retry, adding more load (thundering herd)
```

### The Event Loop Impact (Detailed)

The Node.js event loop has 6 phases. During an OOM scenario:

```
┌──────────────────────────┐
│         timers           │ ← setTimeout/setInterval callbacks DELAYED
├──────────────────────────┤
│   pending callbacks      │ ← I/O callbacks from previous cycle DELAYED
├──────────────────────────┤
│     idle, prepare        │ ← Internal use
├──────────────────────────┤
│         poll             │ ← New I/O events BLOCKED during GC
├──────────────────────────┤
│        check             │ ← setImmediate callbacks DELAYED
├──────────────────────────┤
│    close callbacks       │ ← Socket cleanup DELAYED
└──────────────────────────┘
```

When V8's garbage collector runs, it **stops the world** — the event loop is completely paused. No callbacks execute, no I/O is processed, no new connections are accepted.

**For a single-threaded Node.js process, this means:**
- All WebSocket connections miss heartbeats → clients may disconnect
- All HTTP keep-alive connections timeout
- Database connection pool sockets may timeout (MongoDB default socket timeout: 30 seconds)
- Load balancer health checks may fail → instance marked as unhealthy → removed from rotation

### Mitigation Strategies (From Cheapest to Most Robust)

**Level 1: Enforce strict limits (current — partially implemented)**

The current 5 MB limit is a good start, but it's not sufficient alone:
```js
const upload = multer({ 
  storage: multer.memoryStorage(), 
  limits: { 
    fileSize: 5 * 1024 * 1024,  // 5 MB max ← CURRENT
    files: 1                     // Only accept 1 file per request ← ADD THIS
  }
});
```

**Level 2: Rate limiting per user**

```js
import rateLimit from 'express-rate-limit';

const uploadRateLimit = rateLimit({
  windowMs: 60 * 1000,  // 1 minute window
  max: 3,                // Max 3 uploads per minute per IP
  message: { message: 'Upload rate limit exceeded. Try again later.' }
});

router.post('/', uploadRateLimit, upload.single('image'), createPost);
```

This caps concurrent uploads. Even if 1000 users try to upload simultaneously, only 3 per IP per minute are accepted.

**Level 3: Concurrent upload semaphore (server-level cap)**

```js
let activeUploads = 0;
const MAX_CONCURRENT_UPLOADS = 10;

const concurrencyGuard = (req, res, next) => {
  if (activeUploads >= MAX_CONCURRENT_UPLOADS) {
    return res.status(503).json({ 
      message: 'Server busy. Please try again in a few seconds.',
      retryAfter: 5
    });
  }
  activeUploads++;
  res.on('finish', () => activeUploads--);
  res.on('close', () => activeUploads--);
  next();
};

router.post('/', concurrencyGuard, upload.single('image'), createPost);
```

This ensures that at most 10 uploads are in-flight at any time. With 5 MB each, peak buffer memory is capped at 50 MB.

**Level 4: Switch to `diskStorage` + streaming**

```js
import fs from 'fs';
import os from 'os';
import path from 'path';

const upload = multer({ 
  storage: multer.diskStorage({
    destination: os.tmpdir(),
    filename: (req, file, cb) => {
      cb(null, `upload-${Date.now()}-${Math.random().toString(36).slice(2)}`);
    }
  }),
  limits: { fileSize: 10 * 1024 * 1024 }  // Can now afford 10 MB
});

// In controller:
const uploadToCloudinaryFromDisk = (filePath) => {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'SnapGallery' },
      (error, result) => {
        // Clean up temp file regardless of outcome
        fs.unlink(filePath, () => {});
        if (error) reject(error);
        else resolve(result);
      }
    );
    // Pipe from disk to Cloudinary — constant memory usage (~64KB buffer)
    fs.createReadStream(filePath).pipe(stream);
  });
};
```

**Memory usage with `diskStorage`:** Constant ~64 KB per upload (the streaming buffer size), regardless of file size. 50 concurrent uploads = ~3.2 MB of buffer memory. This is a 99.4% reduction compared to `memoryStorage`.

**Level 5: Direct-to-Cloudinary uploads (bypass server entirely)**

The ultimate solution — the Node.js server never touches the image bytes:

```js
// Server-side: Generate a signed upload preset
app.get('/api/uploads/signature', auth, (req, res) => {
  const timestamp = Math.round(Date.now() / 1000);
  const signature = cloudinary.utils.api_sign_request(
    { timestamp, folder: 'SnapGallery' },
    process.env.CLOUD_API_SECRET
  );
  res.json({
    signature,
    timestamp,
    cloudName: process.env.CLOUD_NAME,
    apiKey: process.env.CLOUD_API_KEY
  });
});

// Client-side: Upload directly to Cloudinary
const { signature, timestamp, cloudName, apiKey } = await fetch('/api/uploads/signature', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(r => r.json());

const formData = new FormData();
formData.append('file', fileInput.files[0]);
formData.append('signature', signature);
formData.append('timestamp', timestamp);
formData.append('api_key', apiKey);
formData.append('folder', 'SnapGallery');

// This goes directly to Cloudinary, not your server
const cloudResult = await fetch(
  `https://api.cloudinary.com/v1_1/${cloudName}/image/upload`,
  { method: 'POST', body: formData }
).then(r => r.json());

// Send the URL back to your server to create the Post document
await fetch('/api/posts', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    imageUrl: cloudResult.secure_url,
    cloudinaryPublicId: cloudResult.public_id,
    caption: captionInput.value
  })
});
```

**Server memory usage for uploads: ZERO.** The server only handles a lightweight JSON request with the URL.

---

## 4. Race Conditions in `toggleLike`

### The Exact Problem

**File:** [postController.js#L84-L106](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L84-L106)

```js
const post = await Post.findById(req.params.id);  // READ
// ... modify post.likes array in memory ...
await post.save();  // WRITE
```

This is a **read-modify-write** cycle without any locking or atomic guarantees.

### Scenario: Two Concurrent Like Requests from the Same User

```
Request 1 (User A likes Post X):
  T=0ms : post = await Post.findById(X)  →  post.likes = []
  T=1ms : index = post.likes.indexOf(A)  →  -1
  T=2ms : post.likes.push(A)             →  post.likes = [A]

Request 2 (User A double-clicks like button):
  T=0ms : post = await Post.findById(X)  →  post.likes = []  ← STALE!
  T=1ms : index = post.likes.indexOf(A)  →  -1
  T=2ms : post.likes.push(A)             →  post.likes = [A]

  T=3ms : Request 1: post.save()         →  MongoDB: likes = [A] ✅
  T=4ms : Request 2: post.save()         →  MongoDB: likes = [A] ← overwrites, but same value (lucky!)
```

This specific scenario is benign (same result). But what about:

### Scenario: Two Different Users Like Simultaneously

```
Request 1 (User A likes Post X):
  T=0ms : post = Post.findById(X)  →  likes = [C, D]
  T=1ms : likes.push(A)            →  likes = [C, D, A]  (in memory)

Request 2 (User B likes Post X):
  T=0ms : post = Post.findById(X)  →  likes = [C, D]  ← STALE (doesn't see A)
  T=1ms : likes.push(B)            →  likes = [C, D, B]  (in memory)

  T=2ms : Request 1: save()  →  MongoDB: likes = [C, D, A] ✅
  T=3ms : Request 2: save()  →  MongoDB: likes = [C, D, B] ← OVERWRITES, loses A's like!
```

**Result:** User A's like is silently lost. The like count shows 3 instead of 4.

### The Fix: Atomic `$addToSet` and `$pull`

```js
export const toggleLike = async (req, res) => {
  try {
    const postId = req.params.id;
    const userId = req.user._id;

    const post = await Post.findById(postId);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    const isLiked = post.likes.includes(userId);

    if (isLiked) {
      // Atomic unlike — no race condition possible
      await Post.updateOne(
        { _id: postId },
        { $pull: { likes: userId } }
      );
    } else {
      // Atomic like — $addToSet prevents duplicates
      await Post.updateOne(
        { _id: postId },
        { $addToSet: { likes: userId } }
      );
    }

    // Reload for response
    const updated = await Post.findById(postId)
      .populate('author', 'username profilePic');
    res.json(updated);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**Why this fixes the race condition:**
- `$addToSet` and `$pull` are **document-level atomic operations** in MongoDB.
- MongoDB applies them directly to the stored document without a read-modify-write cycle.
- Two concurrent `$addToSet` operations will both succeed without overwriting each other.
- `$addToSet` is idempotent — adding the same userId twice results in only one entry.

---

## 5. Orphaned Cloudinary Images on Post Deletion

### The Problem

**File:** [postController.js#L66-L82](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L66-L82)

```js
export const deletePost = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    if (post.author.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    await post.deleteOne();                          // ← Deletes MongoDB document
    res.json({ message: 'Post deleted' });           // ← Image still on Cloudinary!
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**What's missing:** There is no call to `cloudinary.uploader.destroy()` to delete the image from Cloudinary.

### Why This Is a Problem

1. **Storage cost accumulation.** Deleted posts' images consume Cloudinary quota indefinitely.
2. **Privacy concern.** A user deletes a post expecting the image to be gone. But the Cloudinary URL (`result.secure_url`) is still accessible. Anyone who saved the URL can still access the image.
3. **Compliance risk.** GDPR "right to erasure" requires deleting user data upon request. If a user deletes their account and all posts, the images must also be purged from Cloudinary.

### The Fix

**Step 1: Update the Post schema to store `cloudinaryPublicId`:**

```js
// Post.js — add this field
cloudinaryPublicId: {
  type: String,
  required: true
}
```

**Step 2: Store the public_id during upload** ([postController.js#L25-L29](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L25-L29)):
```js
const post = await Post.create({
  author: req.user._id,
  imageUrl: result.secure_url,
  cloudinaryPublicId: result.public_id,  // ← ADD THIS
  caption: req.body.caption || ''
});
```

**Step 3: Delete from Cloudinary on post deletion:**
```js
export const deletePost = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    if (post.author.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    // Delete from Cloudinary first
    if (post.cloudinaryPublicId) {
      try {
        await cloudinary.uploader.destroy(post.cloudinaryPublicId);
      } catch (cloudErr) {
        console.error('Failed to delete Cloudinary image:', cloudErr.message);
        // Continue with MongoDB deletion even if Cloudinary fails
        // (better to have an orphaned image than a post the user can't delete)
      }
    }

    await post.deleteOne();
    res.json({ message: 'Post deleted' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

---

## 6. Unbounded Array Growth: The 16 MB Document Limit

### The Problem

Both `Post.likes` ([Post.js#L18-L21](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js#L18-L21)) and `User.followers/following` ([User.js#L28-L35](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L28-L35)) are **unbounded arrays** of ObjectId references.

**MongoDB document size limit:** 16 MB (16,777,216 bytes).

**Each ObjectId:** 12 bytes in BSON.

**Maximum array sizes before hitting the limit:**

| Field | Other Document Overhead | Available Space for Array | Max ObjectIds | What This Means |
|-------|------------------------|--------------------------|---------------|-----------------|
| `Post.likes` | ~200 bytes (author, imageUrl, caption, timestamps) | ~16.7 MB | ~1,398,101 ObjectIds | A post with 1.4M likes hits the limit |
| `User.followers` | ~500 bytes (username, email, password hash, profilePic, timestamps) + following array | ~8 MB (shared with following) | ~699,050 ObjectIds each | A user with 700K followers AND 700K following hits the limit |

**For SnapGallery's scale:** These limits are unlikely to be reached. But for a production social network (celebrities, viral posts), they're real constraints.

### Additional Performance Issues with Large Arrays

Even before hitting 16 MB, large arrays cause performance degradation:

1. **Network transfer:** Every `findById()` on a user with 100K followers transfers 1.2 MB of ObjectId data over the network, even if you only need the username.
2. **Memory overhead:** Mongoose hydrates each ObjectId in the array into a JavaScript object, consuming significantly more memory than the raw BSON.
3. **Array modification cost:** `$push` and `$pull` on large arrays require MongoDB to rewrite the entire BSON document. For a 1 MB likes array, every like/unlike rewrites 1 MB.
4. **Populate explosion:** `.populate('followers', 'username profilePic')` on a user with 100K followers executes `User.find({ _id: { $in: [100K IDs] } })`, which returns 100K user documents (~200 bytes each = 20 MB of result data).

### The Scalable Fix: Separate Collections

```js
// Follow.js — separate collection
const followSchema = new mongoose.Schema({
  follower: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  following: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: Date.now }
});

// Compound unique index prevents duplicate follows
followSchema.index({ follower: 1, following: 1 }, { unique: true });
// Index for "who follows me?" queries
followSchema.index({ following: 1 });

export default mongoose.model('Follow', followSchema);
```

```js
// Like.js — separate collection
const likeSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  post: { type: mongoose.Schema.Types.ObjectId, ref: 'Post', required: true },
  createdAt: { type: Date, default: Date.now }
});

// Compound unique index prevents duplicate likes (idempotent!)
likeSchema.index({ user: 1, post: 1 }, { unique: true });
// Index for "who liked this post?" queries
likeSchema.index({ post: 1 });

export default mongoose.model('Like', likeSchema);
```

---

## 7. ReDoS Vulnerability in User Search

### The Problem

**File:** [userController.js#L68-L69](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L68-L69)

```js
const users = await User.find({
  username: { $regex: q, $options: 'i' }
})
```

The user's search query `q` is passed **directly** into a MongoDB regex pattern without any sanitization.

### The Attack

A malicious user sends:
```
GET /api/users/search?q=(a%2B)%2Bb
```
URL-decoded: `q = (a+)+b`

This is a classic **ReDoS (Regular Expression Denial of Service)** pattern. The regex `(a+)+b` exhibits **catastrophic backtracking** when tested against a string of repeated `a`s (e.g., `aaaaaaaaaaaaaaaaaaa`). The regex engine explores exponentially many ways to match the repeated `a+` groups.

While MongoDB's regex engine (PCRE) is more resistant to ReDoS than JavaScript's built-in regex, certain patterns can still cause significant CPU usage on the MongoDB server.

### The Fix

```js
// Escape all regex special characters from user input
const escapeRegex = (string) => {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
};

export const searchUsers = async (req, res) => {
  try {
    const { q } = req.query;
    if (!q) {
      return res.status(400).json({ message: 'Search query is required' });
    }

    const sanitized = escapeRegex(q);

    const users = await User.find({
      username: { $regex: `^${sanitized}`, $options: 'i' }  // Anchored to start
    })
      .select('username profilePic followers following')
      .limit(20);

    res.json(users);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**Changes:**
1. `escapeRegex()` escapes all special regex characters (`.*+?^${}()|[]\`)
2. `^` prefix anchors the regex to the start of the string, allowing B-tree index usage

---

## 8. JWT Token Theft and No Revocation

### The Problem

**File:** [authController.js#L4-L6](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/authController.js#L4-L6)

```js
const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '7d' });
};
```

**Issue:** The token has a **7-day expiry** and there is **no revocation mechanism**. If a token is stolen:
- The attacker has 7 days of full account access.
- Changing the password does NOT invalidate existing tokens.
- Logging out on the client (clearing localStorage) does NOT invalidate the server-side token.

### The Fix: Token Versioning

```js
// User.js — add tokenVersion field
tokenVersion: {
  type: Number,
  default: 0
}

// authController.js — include version in JWT
const generateToken = (id, tokenVersion) => {
  return jwt.sign({ id, tokenVersion }, process.env.JWT_SECRET, { expiresIn: '7d' });
};

// middleware/auth.js — verify version
const decoded = jwt.verify(token, process.env.JWT_SECRET);
const user = await User.findById(decoded.id);
if (user.tokenVersion !== decoded.tokenVersion) {
  return res.status(401).json({ message: 'Token revoked' });
}

// On password change:
user.tokenVersion += 1;  // Invalidates all existing tokens
await user.save();
```

---

## 9. Missing Input Validation & Sanitization

### Current State

Registration validation ([authController.js#L12-L14](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/authController.js#L12-L14)):
```js
if (!username || !email || !password) {
  return res.status(400).json({ message: 'All fields are required' });
}
```

**What's NOT validated:**
- Email format (no regex or library validation)
- Password strength (6-char minimum via Mongoose `minlength`, but no complexity requirements)
- Username characters (could contain HTML, SQL injection attempts, or Unicode control characters)
- Caption length is validated by Mongoose (`maxlength: 500`) but not at the controller level

### The Fix

```js
import validator from 'validator'; // npm package

if (!validator.isEmail(email)) {
  return res.status(400).json({ message: 'Invalid email format' });
}

if (!validator.isAlphanumeric(username, 'en-US', { ignore: '_-' })) {
  return res.status(400).json({ message: 'Username can only contain letters, numbers, underscores, and hyphens' });
}

if (password.length < 8 || !/[A-Z]/.test(password) || !/[0-9]/.test(password)) {
  return res.status(400).json({ message: 'Password must be at least 8 characters with at least one uppercase letter and one number' });
}
```

---

## 10. Missing Rate Limiting

### Current State

There is **no rate limiting** anywhere in the codebase. Any endpoint can be called at any rate.

**Attack vectors:**
- **Brute-force login:** Unlimited password attempts against `/api/auth/login`
- **Upload DDoS:** Flood `/api/posts` to exhaust server memory and Cloudinary quota
- **Search abuse:** Spam `/api/users/search` to overload MongoDB with regex queries

### The Fix

```js
import rateLimit from 'express-rate-limit';

// Global rate limit
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                    // 100 requests per window per IP
  message: { message: 'Too many requests. Please try again later.' }
});

// Strict auth rate limit
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 10 login/register attempts per 15 minutes
  message: { message: 'Too many authentication attempts.' }
});

app.use(globalLimiter);
app.use('/api/auth', authLimiter);
```

---

## 11. Database Connection Failure Handling

### Current State

**File:** [config/db.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/config/db.js)

```js
const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI);
    console.log(`MongoDB connected: ${conn.connection.host}`);
  } catch (err) {
    console.error(`MongoDB connection error: ${err.message}`);
    process.exit(1);  // ← Hard exit on failure
  }
};
```

**Issue:** `process.exit(1)` kills the process immediately. On platforms with auto-restart (Heroku, Render), this causes a restart loop if MongoDB is temporarily unavailable.

**Missing:** No connection retry logic, no graceful degradation.

### The Fix

```js
const connectDB = async (retries = 5, delay = 5000) => {
  for (let i = 0; i < retries; i++) {
    try {
      const conn = await mongoose.connect(process.env.MONGODB_URI);
      console.log(`MongoDB connected: ${conn.connection.host}`);
      return;
    } catch (err) {
      console.error(`MongoDB connection attempt ${i + 1}/${retries} failed: ${err.message}`);
      if (i < retries - 1) {
        console.log(`Retrying in ${delay / 1000}s...`);
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }
  console.error('All MongoDB connection attempts failed. Exiting.');
  process.exit(1);
};
```

---

## 12. Missing MIME Type Validation

### Current State

**File:** [posts.js (routes)#L7](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js#L7)

```js
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });
```

**Issue:** There is no `fileFilter` configuration. Multer accepts **any file type** — PDFs, executables, HTML files, anything. The file is uploaded to Cloudinary regardless of type.

### The Fix

```js
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 },
  fileFilter: (req, file, cb) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new multer.MulterError('LIMIT_UNEXPECTED_FILE', 'Only JPEG, PNG, GIF, and WebP images are allowed'));
    }
  }
});
```

**Note:** Client-sent MIME types can be spoofed. For robust validation, use a library like `file-type` that inspects the file's magic bytes:
```js
import { fileTypeFromBuffer } from 'file-type';

// In controller, after multer:
const type = await fileTypeFromBuffer(req.file.buffer);
if (!type || !['image/jpeg', 'image/png', 'image/gif', 'image/webp'].includes(type.mime)) {
  return res.status(400).json({ message: 'Invalid image file' });
}
```

---

## 13. Comprehensive Fix Summary

| Issue | Severity | Current Code | Fix | Effort |
|-------|----------|-------------|-----|--------|
| Orphaned Cloudinary images (upload) | 🟡 Medium | No cleanup on DB failure | Compensating `destroy()` + cron reconciliation | Medium |
| Non-atomic follow/unfollow | 🟡 Medium | Sequential `save()` | MongoDB transactions or `bulkWrite` with `$addToSet/$pull` | Low |
| OOM under concurrent uploads | 🔴 High | `memoryStorage`, no rate limit | Rate limiting + concurrency guard + eventual direct-to-Cloudinary | High |
| Like race conditions | 🟡 Medium | Read-modify-write cycle | Atomic `$addToSet` / `$pull` | Low |
| Orphaned images on delete | 🟡 Medium | No Cloudinary cleanup | Store `public_id`, call `destroy()` on delete | Low |
| Unbounded arrays (16 MB limit) | 🟡 Medium (at scale) | Embedded arrays | Separate `Follow` and `Like` collections | High |
| ReDoS in search | 🔴 High | Unsanitized regex input | Escape regex + prefix anchor | Low |
| JWT no revocation | 🟡 Medium | 7-day token, no invalidation | Token versioning | Low |
| No input validation | 🟡 Medium | Presence check only | Validator library, format checks | Low |
| No rate limiting | 🔴 High | Unlimited requests | `express-rate-limit` | Low |
| DB connection no retry | 🟢 Low | `process.exit(1)` | Retry loop with backoff | Low |
| No MIME type validation | 🟡 Medium | Accepts any file type | `fileFilter` + magic byte check | Low |

---
