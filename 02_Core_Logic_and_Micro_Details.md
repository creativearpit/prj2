# 02 — Core Logic & Micro-Details: "How It Works Internally"

> **Scope:** Line-by-line breakdown of every critical backend feature. Each data flow is traced from the HTTP request through middleware, controller logic, database operations, and back to the response. All code references point to exact files and line numbers.

---

## Table of Contents

1. [The Upload Pipeline: Client → Multer → Cloudinary → MongoDB](#1-the-upload-pipeline)
2. [Personalized Feed Generation: The `$in` Query Deep Dive](#2-personalized-feed-generation)
3. [Social Graph: Bidirectional Follow/Unfollow Logic](#3-social-graph-bidirectional-followunfollow-logic)
4. [Authentication Flow: Registration → Login → Token Verification](#4-authentication-flow)
5. [The Like System: Toggle Mechanics & Array Manipulation](#5-the-like-system)
6. [User Search: Regex Query Internals](#6-user-search-regex-query-internals)
7. [Profile Data Retrieval: Population & Privacy Controls](#7-profile-data-retrieval)
8. [Post Deletion: Authorization & Cleanup](#8-post-deletion)

---

## 1. The Upload Pipeline

### Complete Data Flow: Client `FormData` → Stored MongoDB Document

This is the most complex data flow in SnapGallery, spanning 4 files and involving 3 external services (browser, Cloudinary, MongoDB).

---

### Stage 1: Client-Side — Constructing the `FormData`

**File:** [upload.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/public/js/upload.js) (client-side)

The browser constructs a `multipart/form-data` request:
```js
const formData = new FormData();
formData.append('image', fileInput.files[0]);  // Binary file data
formData.append('caption', captionInput.value); // Text field

fetch('/api/posts', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  // NOTE: Content-Type is NOT set manually — the browser auto-generates
  // the multipart boundary string (e.g., "----WebKitFormBoundary7MA4YWxk...")
  body: formData
});
```

**Critical detail:** The `Content-Type` header MUST NOT be set explicitly when sending `FormData`. If you set it to `multipart/form-data`, the browser won't append the boundary string, and Multer's `Busboy` parser will fail with `"Multipart: Boundary not found"`.

**What the raw HTTP request looks like on the wire:**
```
POST /api/posts HTTP/1.1
Host: localhost:3000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxk

------WebKitFormBoundary7MA4YWxk
Content-Disposition: form-data; name="image"; filename="sunset.jpg"
Content-Type: image/jpeg

<binary JPEG data>
------WebKitFormBoundary7MA4YWxk
Content-Disposition: form-data; name="caption"

Beautiful sunset at the beach
------WebKitFormBoundary7MA4YWxk--
```

---

### Stage 2: Express Middleware Chain — Auth + Multer

**File:** [posts.js (routes)](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js)

```js
// Line 7: Configure Multer
const upload = multer({ 
  storage: multer.memoryStorage(),        // Store in V8 heap, not disk
  limits: { fileSize: 5 * 1024 * 1024 }   // 5 MB hard cap
});

// Line 9: Apply auth to ALL post routes
router.use(auth);

// Line 12: Chain auth → multer → controller
router.post('/', upload.single('image'), createPost);
```

**Execution order when a POST request arrives:**

**Step 2a: `auth` middleware** ([middleware/auth.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/middleware/auth.js))

```js
const auth = async (req, res, next) => {
  // Line 6: Extract Authorization header
  const header = req.headers.authorization;
  
  // Line 7: Validate format "Bearer <token>"
  if (!header || !header.startsWith('Bearer ')) {
    return res.status(401).json({ message: 'No token provided' });
  }

  // Line 11: Split "Bearer eyJ..." → extract just "eyJ..."
  const token = header.split(' ')[1];
  
  // Line 12: Verify HMAC-SHA256 signature AND check expiration
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  // jwt.verify() throws TokenExpiredError or JsonWebTokenError on failure
  
  // Line 13: Load user from DB (ensures user still exists and isn't deleted)
  const user = await User.findById(decoded.id).select('-password');
  // .select('-password') EXCLUDES the password field at the MongoDB query level
  // This is more secure than stripping it after retrieval
  
  // Line 19: Attach user to request object for downstream handlers
  req.user = user;
  next();
};
```

**Key micro-detail:** The `.select('-password')` on Line 13 is a MongoDB **projection**. It translates to the following MongoDB query:
```json
db.users.findOne({ "_id": ObjectId("...") }, { "password": 0 })
```
The `password` field is never transferred from MongoDB to the Node.js process. This is defense-in-depth — even if `toJSON()` (which also strips password) were accidentally removed from the schema, the password would never leave the database layer for authenticated routes.

**Step 2b: `multer` middleware** (`upload.single('image')`)

After `auth` calls `next()`, Multer executes:

1. **Busboy initialization:** Multer creates a `Busboy` instance configured to parse `multipart/form-data`. It reads the `Content-Type` header to extract the boundary string.

2. **Streaming parse:** As the request body streams in (chunk by chunk from the TCP socket), Busboy identifies:
   - **File parts** (the `image` field) → chunks are accumulated into a `Buffer` in V8's heap
   - **Field parts** (the `caption` field) → stored as `req.body.caption`

3. **File size enforcement:** If the accumulated buffer exceeds `5 * 1024 * 1024` bytes (5 MB), Multer emits a `MulterError` with code `LIMIT_FILE_SIZE` and aborts the request. The partially accumulated buffer is released.

4. **File object construction:** On successful parsing, Multer creates:
   ```js
   req.file = {
     fieldname: 'image',
     originalname: 'sunset.jpg',
     encoding: '7bit',
     mimetype: 'image/jpeg',
     buffer: <Buffer ff d8 ff e0 ...>,  // Complete file in memory
     size: 2345678                        // Byte count
   };
   ```

5. **Control passes to `createPost`** via `next()`.

**What happens if no file is sent:** Multer will not set `req.file` (it will be `undefined`). The controller catches this at [postController.js#L19-L21](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L19-L21):
```js
if (!req.file) {
  return res.status(400).json({ message: 'Image is required' });
}
```

---

### Stage 3: Buffer → Cloudinary Stream

**File:** [postController.js#L4-L15](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L4-L15)

```js
const uploadToCloudinary = (buffer) => {
  return new Promise((resolve, reject) => {
    // Line 6: Create a Writable stream connected to Cloudinary's upload API
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'SnapGallery' },  // All images stored in this folder on Cloudinary
      (error, result) => {         // Callback when upload completes or fails
        if (error) reject(error);
        else resolve(result);
      }
    );
    // Line 13: Push the entire buffer into the stream and signal end-of-data
    stream.end(buffer);
  });
};
```

**Line-by-line micro-analysis:**

**Line 6 — `cloudinary.uploader.upload_stream()`:**
- This creates a **Node.js `Writable` stream** that connects to Cloudinary's HTTP upload API (`https://api.cloudinary.com/v1_1/<cloud_name>/image/upload`).
- Internally, the Cloudinary SDK opens an HTTPS connection to their ingest endpoint and streams data through it.
- The `{ folder: 'SnapGallery' }` option namespaces the image, resulting in a `public_id` like `SnapGallery/abc123xyz`.

**Line 8-10 — The callback:**
- This is invoked once Cloudinary finishes processing the upload (or if it fails).
- On success, `result` contains:
  ```json
  {
    "public_id": "SnapGallery/abc123xyz",
    "version": 1691234567,
    "signature": "...",
    "width": 1920,
    "height": 1080,
    "format": "jpg",
    "resource_type": "image",
    "created_at": "2026-08-13T16:45:00Z",
    "bytes": 2345678,
    "url": "http://res.cloudinary.com/...",
    "secure_url": "https://res.cloudinary.com/demo/image/upload/v1691234567/SnapGallery/abc123xyz.jpg"
  }
  ```
- The `secure_url` is the HTTPS CDN link that will be stored in MongoDB.
- The `public_id` (`SnapGallery/abc123xyz`) is what you'd use to delete the image later via `cloudinary.uploader.destroy(public_id)`. **Note: This `public_id` is NOT stored in the current codebase — see the orphaned images section in 03_Failures_and_Edge_Cases.md.**

**Line 13 — `stream.end(buffer)`:**
- `.end(buffer)` is equivalent to `.write(buffer)` followed by `.end()`.
- It pushes the entire buffer into the writable stream in a single write operation.
- The stream then flushes this data over HTTPS to Cloudinary.
- **Memory lifecycle:** At this point, both `req.file.buffer` AND the stream's internal buffer reference the same data. The buffer is effectively duplicated in memory briefly. After `stream.end()` completes and the Promise resolves, both references become eligible for GC.

**Why wrap in a Promise:** The Cloudinary SDK's `upload_stream` uses a callback pattern (not Promises). Wrapping it in `new Promise()` allows the controller to use `await`:
```js
const result = await uploadToCloudinary(req.file.buffer);
```

---

### Stage 4: MongoDB Document Creation

**File:** [postController.js#L25-L32](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L25-L32)

```js
// Line 25: Create the Post document in MongoDB
const post = await Post.create({
  author: req.user._id,            // ObjectId from the auth middleware
  imageUrl: result.secure_url,      // Cloudinary CDN URL
  caption: req.body.caption || ''   // Text from the form, default empty string
});

// Line 31: Populate the author field for the response
const populated = await post.populate('author', 'username profilePic');

// Line 32: Send the complete post back to the client
res.status(201).json(populated);
```

**What `Post.create()` does internally:**

1. **Schema validation** ([Post.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js)):
   - `author` — Verified as a valid `ObjectId`, `required: true` passes.
   - `imageUrl` — Verified as a `String`, `required: true` passes.
   - `caption` — Validated against `maxlength: 500`, defaults to `''`.
   - `likes` — Not provided, defaults to `[]`.

2. **Timestamp injection** — `{ timestamps: true }` ([Post.js#L22](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js#L22)) auto-adds:
   - `createdAt: new Date()` — used for feed sorting
   - `updatedAt: new Date()` — updated on every `.save()`

3. **MongoDB write** — Mongoose sends an `insertOne` command to the MongoDB driver, which writes the BSON document to the `posts` collection.

4. **The returned `post` object** is a Mongoose document instance with methods like `.populate()`, `.save()`, `.toJSON()`.

**What `.populate('author', 'username profilePic')` does:**

1. Takes the `author` field value (an `ObjectId`).
2. Executes `User.findById(author).select('username profilePic')` internally.
3. Replaces the `ObjectId` with the actual user data in the document.

**Before populate:**
```json
{ "author": "ObjectId('64a1b2c3...')", "imageUrl": "https://...", ... }
```

**After populate:**
```json
{ "author": { "_id": "64a1b2c3...", "username": "arpit", "profilePic": "https://..." }, "imageUrl": "https://...", ... }
```

This is essentially a **client-side JOIN** — Mongoose issues a second query to MongoDB to resolve the reference. In SQL terms, this is equivalent to:
```sql
SELECT p.*, u.username, u.profile_pic 
FROM posts p 
LEFT JOIN users u ON p.author_id = u.id 
WHERE p.id = <newly_created_id>;
```

**Total number of external I/O operations in the upload pipeline:**
1. MongoDB query: `User.findById()` in auth middleware
2. HTTPS request: Buffer → Cloudinary upload stream
3. MongoDB write: `Post.create()`
4. MongoDB query: `.populate('author')` — second `User.findById()`

**4 external I/O operations per upload request.**

---

### Complete Flow Diagram

```
Client (Browser)
    │
    ├─── FormData { image: File, caption: String }
    ├─── Authorization: Bearer <JWT>
    │
    ▼
Express Server
    │
    ├─── [1] cors() middleware (allow cross-origin)
    ├─── [2] express.json() middleware (parse JSON bodies — NOT used here, but present globally)
    ├─── [3] auth middleware
    │        ├── Extract JWT from Authorization header
    │        ├── jwt.verify(token, JWT_SECRET) → { id: "..." }
    │        ├── User.findById(decoded.id).select('-password') → MongoDB Query #1
    │        ├── req.user = user
    │        └── next()
    ├─── [4] multer middleware (upload.single('image'))
    │        ├── Parse multipart boundary from Content-Type header
    │        ├── Create Busboy stream parser
    │        ├── Accumulate file chunks into Buffer (V8 heap)
    │        ├── Enforce 5 MB file size limit
    │        ├── Set req.file = { buffer, mimetype, originalname, size }
    │        ├── Set req.body.caption = "..."
    │        └── next()
    ├─── [5] createPost controller
    │        ├── Validate req.file exists
    │        ├── uploadToCloudinary(req.file.buffer)
    │        │      ├── Create Writable stream to Cloudinary API
    │        │      ├── stream.end(buffer) → HTTPS upload → External I/O #2
    │        │      └── resolve(result) → { secure_url, public_id, ... }
    │        ├── Post.create({ author, imageUrl, caption }) → MongoDB Write #3
    │        ├── post.populate('author') → MongoDB Query #4
    │        └── res.status(201).json(populated)
    │
    ▼
Client receives:
{
  "_id": "...",
  "author": { "_id": "...", "username": "arpit", "profilePic": "..." },
  "imageUrl": "https://res.cloudinary.com/.../SnapGallery/abc123.jpg",
  "caption": "Beautiful sunset",
  "likes": [],
  "createdAt": "2026-08-13T...",
  "updatedAt": "2026-08-13T..."
}
```

---

## 2. Personalized Feed Generation

### The `$in` Query Deep Dive

**File:** [postController.js#L38-L51](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L38-L51)

```js
export const getFeed = async (req, res) => {
  try {
    const user = req.user;
    // Line 41: Spread following IDs + add self to see own posts in feed
    const following = [...user.following, user._id];

    // Line 43-46: The core feed query
    const posts = await Post.find({ author: { $in: following } })
      .sort({ createdAt: -1 })
      .populate('author', 'username profilePic')
      .limit(50);

    res.json(posts);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

### Line-by-Line Breakdown

**Line 40 — `const user = req.user;`**
- `req.user` was set by the auth middleware ([middleware/auth.js#L19](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/middleware/auth.js#L19)).
- This is the **complete Mongoose document** for the authenticated user, including the `following` array (an array of `ObjectId` values).
- **Important:** The user was loaded with `.select('-password')`, so the password hash is not in memory.

**Line 41 — `const following = [...user.following, user._id];`**
- The spread operator `...` creates a **new array** containing all ObjectIds from `user.following`.
- `user._id` is appended so the user sees **their own posts** in the feed.
- If User A follows users B, C, D, then `following` = `[B._id, C._id, D._id, A._id]`.
- **Edge case:** If the user follows nobody, `following` = `[user._id]` — they only see their own posts.
- **Memory implication:** For a user following 10,000 accounts, this creates an array of 10,001 ObjectIds in memory (each ObjectId is 12 bytes = ~120 KB). This is negligible.

**Line 43 — `Post.find({ author: { $in: following } })`**

This is the core query. Let's break down exactly what MongoDB does:

**The `$in` operator internally:**

1. **Query plan selection:** MongoDB's query optimizer looks for an index on the `author` field. Since `author` is defined as an `ObjectId` with `ref: 'User'` in the schema ([Post.js#L4-L8](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js#L4-L8)), Mongoose does NOT automatically create an index on it (only `_id` gets an auto-index). However, if the collection is small, MongoDB will use a collection scan (COLLSCAN).

2. **With an index on `author`:** MongoDB converts the `$in` query into multiple **bounded index seeks** on the B-tree. For `$in: [A, B, C, D]`, it performs:
   - Seek to key `A` → collect matching document pointers
   - Seek to key `B` → collect matching document pointers
   - Seek to key `C` → collect matching document pointers
   - Seek to key `D` → collect matching document pointers
   - Merge all pointers → fetch documents

3. **Without an index (current state):** MongoDB performs a **full collection scan (COLLSCAN)**, checking every document's `author` field against the `$in` array. For N posts and F followed users:
   - Time complexity: O(N × F) for the membership check (though MongoDB optimizes this with hash-based comparison)
   - This is acceptable for small collections but degrades linearly with collection size.

**What the generated MongoDB query looks like:**
```json
db.posts.find({ 
  "author": { "$in": [ObjectId("A"), ObjectId("B"), ObjectId("C"), ObjectId("D")] } 
}).sort({ "createdAt": -1 }).limit(50)
```

**Line 44 — `.sort({ createdAt: -1 })`**

- `-1` means **descending** (newest first).
- MongoDB must sort the result set. Without an index on `createdAt`, this is an **in-memory sort**.
- MongoDB's in-memory sort limit is **100 MB**. If the intermediate result set (posts from followed users) exceeds 100 MB, the query fails with `QueryExceededMemoryLimitNoDiskUseAllowed`.
- **Optimization:** A compound index `{ author: 1, createdAt: -1 }` would allow MongoDB to traverse the index in sorted order, eliminating the in-memory sort entirely.

**Line 45 — `.populate('author', 'username profilePic')`**

- After the main query returns (let's say) 50 post documents, Mongoose collects all unique `author` ObjectIds from those 50 posts.
- It then executes a **single** `User.find({ _id: { $in: [unique_author_ids] } }).select('username profilePic')` query.
- This is Mongoose's N+1 optimization — instead of 50 separate `User.findById()` calls, it batches them into one `$in` query.
- The results are mapped back to each post document, replacing the ObjectId with the user data.

**Total MongoDB queries for `getFeed`:**
1. `User.findById()` in auth middleware (to get `req.user` with `following` array)
2. `Post.find({ author: { $in: [...] } }).sort().limit()` — the main feed query
3. `User.find({ _id: { $in: [unique_authors] } }).select('username profilePic')` — populate batch query

**3 queries per feed request.**

**Line 46 — `.limit(50)`**

- Caps the result set to 50 documents maximum.
- This acts as **pagination without a cursor**. The current implementation is "load latest 50" only.
- **Missing: cursor-based pagination.** To implement infinite scroll, the client would need to send the `createdAt` timestamp (or `_id`) of the last loaded post, and the query would become:
  ```js
  Post.find({ 
    author: { $in: following },
    createdAt: { $lt: lastPostTimestamp }
  }).sort({ createdAt: -1 }).limit(50);
  ```

---

### Fan-Out-On-Read vs. Fan-Out-On-Write: Deep Comparison

The current architecture uses **fan-out-on-read** — the feed is computed at read time by querying across multiple users' posts.

| Aspect | Fan-Out-On-Read (Current) | Fan-Out-On-Write (Twitter/Instagram-style) |
|--------|---------------------------|-------------------------------------------|
| **What happens on POST** | Single `Post.create()` — O(1) write | Create post + write N timeline entries (one per follower) — O(N) write |
| **What happens on GET /feed** | `Post.find({ author: { $in: following } })` — O(F) query where F = following count | `Timeline.find({ userId: me }).sort().limit(50)` — O(1) read from pre-computed timeline |
| **Data structure** | `posts` collection only | `posts` collection + `timelines` collection (or Redis sorted set) |
| **Write amplification** | None | User with 1M followers → 1M timeline writes per post |
| **Read latency** | Depends on following count + post volume | Constant — pre-computed, indexed by userId |
| **Consistency** | **Always consistent** — query returns latest data | **Eventually consistent** — fan-out is async, new followers may miss old posts |
| **Celebrity problem** | N/A (reads are per-user) | User with 10M followers → 10M writes per post → write storm |
| **Storage** | 1 copy of each post | N copies of each post reference (one per follower's timeline) |
| **Unfollow handling** | Automatic — unfollowed user's posts disappear from `$in` | Must purge unfollowed user's posts from timeline (complex) |
| **Infrastructure** | MongoDB only | MongoDB + message queue (RabbitMQ/Kafka) + worker processes |

**Why fan-out-on-read is correct for SnapGallery:**

1. **Small following counts.** A typical SnapGallery user follows 10-200 accounts. The `$in` array is small, and the query is fast.
2. **No celebrity problem.** SnapGallery doesn't have users with millions of followers, so write amplification isn't a concern.
3. **Perfect consistency.** The feed always reflects the current follow graph. If you unfollow someone, their posts vanish from your next feed request instantly.
4. **Zero infrastructure overhead.** No message queue, no worker processes, no separate timeline collection.

**When to switch to fan-out-on-write:**
- When a user follows 50,000+ accounts and feed reads become slow.
- When read volume massively exceeds write volume (e.g., 1,000 reads per 1 write).
- When you need sub-100ms feed response times guaranteed (pre-computed timelines).

---

## 3. Social Graph: Bidirectional Follow/Unfollow Logic

### The Complete `toggleFollow` Function Breakdown

**File:** [userController.js#L3-L36](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L3-L36)

```js
export const toggleFollow = async (req, res) => {
  try {
    // STEP 1: Self-follow guard
    if (req.params.id === req.user._id.toString()) {
      return res.status(400).json({ message: 'Cannot follow yourself' });
    }

    // STEP 2: Load target user
    const targetUser = await User.findById(req.params.id);
    if (!targetUser) {
      return res.status(404).json({ message: 'User not found' });
    }

    // STEP 3: Load current user (fresh from DB)
    const currentUser = await User.findById(req.user._id);
    
    // STEP 4: Determine current follow state
    const isFollowing = currentUser.following.includes(targetUser._id);

    // STEP 5: Toggle the bidirectional relationship
    if (isFollowing) {
      // UNFOLLOW: Remove from both arrays
      currentUser.following.pull(targetUser._id);
      targetUser.followers.pull(currentUser._id);
    } else {
      // FOLLOW: Add to both arrays
      currentUser.following.push(targetUser._id);
      targetUser.followers.push(currentUser._id);
    }

    // STEP 6: Persist both changes
    await currentUser.save();
    await targetUser.save();

    // STEP 7: Return updated state
    res.json({
      isFollowing: !isFollowing,
      followersCount: targetUser.followers.length,
      followingCount: targetUser.following.length
    });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

### Step-by-Step Micro-Analysis

**Step 1 — Self-Follow Guard (Line 5-7):**
```js
if (req.params.id === req.user._id.toString()) {
```
- `req.params.id` is a **string** from the URL (e.g., `"64a1b2c3..."`).
- `req.user._id` is an **ObjectId** object.
- `.toString()` converts the ObjectId to its hex string representation for comparison.
- Without `.toString()`, the comparison would always be `false` (different types), and users could follow themselves.

**Step 2 — Target User Load (Line 9-12):**
```js
const targetUser = await User.findById(req.params.id);
```
- This is a fresh database load, not from the request context.
- **No `.select('-password')`** here — the password field IS loaded. This is technically unnecessary since we only need `followers`. However, the `toJSON()` method on the User schema ([User.js#L47-L51](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L47-L51)) strips it before serialization.

**Step 3 — Current User Re-Load (Line 14):**
```js
const currentUser = await User.findById(req.user._id);
```
- **Why re-load the current user?** `req.user` was loaded in the auth middleware, but it may be stale if another concurrent request modified the user's data between the auth middleware execution and this point.
- More importantly, `req.user` was loaded with `.select('-password')`, which means `save()` on it might unintentionally blank out the password field. Reloading ensures a complete Mongoose document.

**Step 4 — Follow State Check (Line 15):**
```js
const isFollowing = currentUser.following.includes(targetUser._id);
```
- **How `.includes()` works on Mongoose arrays:** Mongoose `MongooseArray` overrides the native `.includes()` method to perform **ObjectId-aware comparison**. It internally uses `ObjectId.equals()` (Buffer comparison) rather than JavaScript's `===` (reference comparison).
- Without this override, `includes()` would always return `false` because two different ObjectId objects with the same value are not `===` equal.
- **Time complexity:** O(N) where N = length of the `following` array. For a user following 1000 accounts, this iterates 1000 ObjectId comparisons in the worst case.

**Step 5 — Array Manipulation (Lines 17-23):**

For **unfollow** (Lines 18-19):
```js
currentUser.following.pull(targetUser._id);
targetUser.followers.pull(currentUser._id);
```
- `.pull()` is a Mongoose array method that marks the element for removal.
- It does NOT immediately modify the database — it only modifies the in-memory Mongoose document.
- Internally, `.pull()` calls `MongooseArray.prototype.pull()`, which uses `$pull` semantics (removes by value, not index).

For **follow** (Lines 21-22):
```js
currentUser.following.push(targetUser._id);
targetUser.followers.push(currentUser._id);
```
- `.push()` adds the ObjectId to the in-memory array.
- Mongoose tracks this as a `$push` operation for the eventual `save()`.

**Step 6 — Sequential Saves (Lines 25-26):**
```js
await currentUser.save();
await targetUser.save();
```
- **Two separate database write operations.** Each `save()` triggers:
  1. Mongoose validation (schema validation for all modified fields)
  2. Pre-save middleware execution (password hashing if password is modified — see [User.js#L38-L41](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L38-L41))
  3. MongoDB `updateOne` or `replaceOne` command

- **The atomicity gap:** These are **NOT** wrapped in a transaction. If `currentUser.save()` succeeds but `targetUser.save()` fails (network error, validation error, server crash between the two calls), the social graph becomes inconsistent. This is analyzed in detail in **03_Failures_and_Edge_Cases.md**.

**Step 7 — Response (Lines 28-32):**
```js
res.json({
  isFollowing: !isFollowing,        // Toggled state
  followersCount: targetUser.followers.length,
  followingCount: targetUser.following.length
});
```
- `!isFollowing` inverts the previous state (if they were following, now they aren't, and vice versa).
- `.length` reads from the **in-memory** Mongoose array, which has already been modified in Step 5.
- **Note:** This returns the target user's `followingCount`, which seems like a UI display detail (the profile page shows both counts).

**Total MongoDB queries in `toggleFollow`:**
1. `User.findById()` in auth middleware → load authenticated user
2. `User.findById(req.params.id)` → load target user (Line 9)
3. `User.findById(req.user._id)` → reload current user (Line 14)
4. `currentUser.save()` → write to current user's document (Line 25)
5. `targetUser.save()` → write to target user's document (Line 26)

**5 MongoDB operations per follow/unfollow action.**

---

## 4. Authentication Flow

### Registration Flow

**File:** [authController.js#L8-L28](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/authController.js#L8-L28)

**Step-by-step:**

1. **Input extraction** (Line 10):
   ```js
   const { username, email, password } = req.body;
   ```
   `express.json()` middleware ([server.js#L13](file:///c:/Users/ARPRIT/Desktop/SnapGallery/server.js#L13)) has already parsed the JSON body.

2. **Input validation** (Lines 12-14):
   ```js
   if (!username || !email || !password) {
     return res.status(400).json({ message: 'All fields are required' });
   }
   ```
   Basic presence check. **Not validated:** email format, password complexity, username character restrictions.

3. **Duplicate check** (Lines 16-19):
   ```js
   const existingUser = await User.findOne({ $or: [{ email }, { username }] });
   ```
   The `$or` operator checks both `email` AND `username` uniqueness in a single query. This is important because the User schema defines `unique: true` on both fields ([User.js#L8](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L8), [User.js#L15](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L15)), which creates unique indexes in MongoDB. However, catching the duplicate via `findOne` first provides a cleaner error message than catching Mongoose's `MongoError: E11000 duplicate key error`.

4. **Dynamic profile picture** (Line 21):
   ```js
   const dynamicProfilePic = `https://ui-avatars.com/api/?background=random&size=200&name=${encodeURIComponent(username)}`;
   ```
   Uses the `ui-avatars.com` API to generate a letter-based avatar based on the username. `encodeURIComponent` handles special characters in usernames.

5. **User creation** (Line 22):
   ```js
   const user = await User.create({ username, email, password, profilePic: dynamicProfilePic });
   ```
   `User.create()` triggers the pre-save hook ([User.js#L38-L41](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L38-L41)):
   ```js
   userSchema.pre('save', async function () {
     if (!this.isModified('password')) return;
     this.password = await bcrypt.hash(this.password, 10);
   });
   ```
   The plaintext password is hashed with bcrypt (cost factor 10 = 1024 iterations of Eksblowfish). The hash replaces the plaintext before it reaches MongoDB.

6. **Token generation** (Line 23):
   ```js
   const token = generateToken(user._id);
   ```
   JWT payload: `{ id: user._id }`. Signed with HMAC-SHA256. Expires in 7 days.

7. **Response** (Line 25):
   ```js
   res.status(201).json({ token, user });
   ```
   `user` is serialized via `toJSON()` ([User.js#L47-L51](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L47-L51)) which strips the password hash.

---

## 5. The Like System

### Toggle Mechanics & Array Manipulation

**File:** [postController.js#L84-L106](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L84-L106)

```js
export const toggleLike = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    const userId = req.user._id;
    const index = post.likes.indexOf(userId);

    if (index === -1) {
      // User has NOT liked → add like
      post.likes.push(userId);
    } else {
      // User HAS liked → remove like
      post.likes.splice(index, 1);
    }

    await post.save();
    const populated = await post.populate('author', 'username profilePic');
    res.json(populated);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**Line 92 — `post.likes.indexOf(userId)`:**
- Mongoose's `indexOf` on a `MongooseArray` uses ObjectId-aware comparison.
- Returns the integer index of the first matching ObjectId, or `-1` if not found.
- **Time complexity:** O(N) where N = number of likes on the post.

**Lines 94-98 — The Toggle Logic:**
- **Like:** `post.likes.push(userId)` — appends the userId to the array.
- **Unlike:** `post.likes.splice(index, 1)` — removes exactly 1 element at the found index.
- `splice(index, 1)` modifies the array in-place, shifting all subsequent elements left.

**Line 100 — `await post.save()`:**
- This sends the modified `likes` array to MongoDB.
- Mongoose generates a `$set` operation for the entire `likes` array.
- For a post with 10,000 likes, this rewrites all 10,000 ObjectIds even if only 1 was added/removed.

**Line 101 — Second populate:**
- After saving, the `author` field is repopulated for the response.

**Total operations per like/unlike:**
1. `User.findById()` in auth middleware
2. `Post.findById(req.params.id)` — load post with likes array
3. `post.save()` — write modified likes array
4. `User.findById()` via `.populate('author')` — load author for response

**4 MongoDB operations per like/unlike action.**

---

## 6. User Search: Regex Query Internals

**File:** [userController.js#L61-L78](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L61-L78)

```js
export const searchUsers = async (req, res) => {
  try {
    const { q } = req.query;
    if (!q) {
      return res.status(400).json({ message: 'Search query is required' });
    }

    const users = await User.find({
      username: { $regex: q, $options: 'i' }
    })
      .select('username profilePic followers following')
      .limit(20);

    res.json(users);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**How `$regex` works under the hood:**

1. MongoDB compiles the regex string `q` into a PCRE (Perl-Compatible Regular Expression) pattern.
2. The `$options: 'i'` flag makes it case-insensitive.
3. **Without prefix anchoring (`^`)**, MongoDB cannot use a B-tree index efficiently — it must examine every document in the collection (COLLSCAN).
4. The regex `q` is applied as a **substring match** — it matches anywhere in the `username` string.

**What the MongoDB query looks like:**
```json
db.users.find(
  { "username": { "$regex": "arp", "$options": "i" } },
  { "username": 1, "profilePic": 1, "followers": 1, "following": 1 }
).limit(20)
```

**`.select('username profilePic followers following')`** translates to a MongoDB projection:
```json
{ "username": 1, "profilePic": 1, "followers": 1, "following": 1 }
```
This excludes `email` and `password` from the results at the database level.

---

## 7. Profile Data Retrieval

**File:** [userController.js#L38-L58](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L38-L58)

```js
export const getUser = async (req, res) => {
  try {
    const user = await User.findById(req.params.id)
      .select('-password')
      .populate('followers', 'username profilePic')
      .populate('following', 'username profilePic');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    // Privacy control: Hide email from non-owners
    const userObj = user.toObject();
    if (req.user._id.toString() !== req.params.id) {
      delete userObj.email;
    }

    res.json(userObj);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**The double `.populate()` chain:**

1. **First populate** — `.populate('followers', 'username profilePic')`:
   - Collects all ObjectIds in the `followers` array.
   - Executes: `User.find({ _id: { $in: follower_ids } }).select('username profilePic')`
   - Replaces each ObjectId with `{ _id, username, profilePic }`.

2. **Second populate** — `.populate('following', 'username profilePic')`:
   - Same process for the `following` array.

**Total MongoDB queries for `getUser`:**
1. `User.findById()` in auth middleware
2. `User.findById(req.params.id)` — load target user
3. `User.find({ $in: follower_ids })` — populate followers
4. `User.find({ $in: following_ids })` — populate following

**4 queries per profile view.**

**The privacy control (Lines 50-53):**
```js
const userObj = user.toObject();  // Convert Mongoose doc to plain JS object
if (req.user._id.toString() !== req.params.id) {
  delete userObj.email;  // Remove email for non-owners
}
```
- `.toObject()` creates a raw JavaScript object (not a Mongoose document). This allows `delete` to work on it.
- `delete` on a Mongoose document would not work as expected due to getter/setter proxy behavior.
- The comparison uses `.toString()` on `_id` to convert ObjectId to string for a simple string equality check.

---

## 8. Post Deletion

**File:** [postController.js#L66-L82](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L66-L82)

```js
export const deletePost = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    // Authorization: Only the author can delete
    if (post.author.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    await post.deleteOne();
    res.json({ message: 'Post deleted' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
};
```

**Authorization logic (Line 73):**
```js
if (post.author.toString() !== req.user._id.toString()) {
```
- Compares the post's `author` ObjectId with the authenticated user's `_id`.
- Both are converted to strings for comparison.
- Returns 403 Forbidden if they don't match — only the original author can delete their post.

**`post.deleteOne()` (Line 77):**
- This is a Mongoose document method that translates to:
  ```json
  db.posts.deleteOne({ "_id": ObjectId("...") })
  ```
- **What it does NOT do:** Delete the Cloudinary image. The image persists on Cloudinary indefinitely. See **03_Failures_and_Edge_Cases.md** for the orphaned image analysis.

**Total operations per delete:**
1. `User.findById()` in auth middleware
2. `Post.findById(req.params.id)` — load post
3. `post.deleteOne()` — delete from MongoDB

**3 MongoDB operations per delete.**

---
