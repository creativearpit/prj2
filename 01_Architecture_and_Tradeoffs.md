# 01 — Architecture & Tradeoffs: "Why This and Not That"

> **Scope:** Every architectural decision in SnapGallery's backend, dissected against the alternatives. Each answer cites exact files, line numbers, and function signatures from the codebase.

---

## Table of Contents

1. [Media Storage: Cloudinary vs. S3 vs. Local Disk](#1-media-storage-cloudinary-vs-s3-vs-local-disk)
2. [Upload Pipeline: `memoryStorage` vs. `diskStorage`](#2-upload-pipeline-memorystorage-vs-diskstorage)
3. [Database: MongoDB vs. Neo4j vs. PostgreSQL for a Social Graph](#3-database-mongodb-vs-neo4j-vs-postgresql-for-a-social-graph)
4. [Framework: Express 5 vs. Fastify vs. NestJS](#4-framework-express-5-vs-fastify-vs-nestjs)
5. [Authentication: JWT (Stateless) vs. Sessions (Stateful)](#5-authentication-jwt-stateless-vs-sessions-stateful)
6. [Module System: ES Modules vs. CommonJS](#6-module-system-es-modules-vs-commonjs)
7. [Password Hashing: bcryptjs vs. argon2 vs. scrypt](#7-password-hashing-bcryptjs-vs-argon2-vs-scrypt)
8. [Deployment Architecture: Monolith vs. Microservices](#8-deployment-architecture-monolith-vs-microservices)

---

## 1. Media Storage: Cloudinary vs. S3 vs. Local Disk

### Q: "Why stream directly to Cloudinary instead of using AWS S3 or storing files on the local server disk?"

**The Decision in Code:**

The entire media storage strategy is configured in just two files:
- [cloudinary.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/utils/cloudinary.js) (10 lines) — Cloudinary SDK initialization with three env vars (`CLOUD_NAME`, `CLOUD_API_KEY`, `CLOUD_API_SECRET`).
- [postController.js#L4-L15](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L4-L15) — The `uploadToCloudinary()` function that wraps `cloudinary.uploader.upload_stream` in a Promise, streaming the in-memory buffer directly to Cloudinary's ingest API.

**Answer — The Reasoning (in depth):**

**Reason 1: Unified Storage + CDN + Transformation in One Service**

Cloudinary is not just a storage bucket — it is a **media intelligence platform** that combines:
- **Object storage** (equivalent to S3)
- **Global CDN delivery** (equivalent to CloudFront)
- **Real-time image transformations** (equivalent to running Sharp/ImageMagick on a Lambda@Edge function)

When `uploadToCloudinary()` resolves (Line 8-10 of [postController.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L8-L10)), the `result.secure_url` returned is already a CDN-backed URL. It is immediately servable worldwide without any additional configuration.

If I had used **S3 alone**, I would need to:
1. Configure an S3 bucket with proper CORS headers for browser access.
2. Set up IAM policies (PutObject for the server, GetObject for public reads or pre-signed URLs).
3. Provision a CloudFront distribution pointing to the S3 origin.
4. Configure TLS certificates via ACM (AWS Certificate Manager) for HTTPS on the CloudFront distribution.
5. Set Cache-Control headers on S3 objects for CDN caching behavior.
6. Implement cache invalidation strategies for updated/deleted images.

That's **6 separate configuration steps** just to match what Cloudinary gives me with a single `upload_stream` call.

**Reason 2: Zero-Overhead Image Optimization**

Cloudinary URLs support dynamic transformation parameters. Given the stored `secure_url`, I can derive optimized variants purely through URL manipulation:
```
// Original (stored in MongoDB)
https://res.cloudinary.com/demo/image/upload/v1234/SnapGallery/abc123.jpg

// Thumbnail (400px wide, auto-quality, auto-format)
https://res.cloudinary.com/demo/image/upload/w_400,q_auto,f_auto/v1234/SnapGallery/abc123.jpg

// WebP conversion with face-aware crop
https://res.cloudinary.com/demo/image/upload/w_200,h_200,c_fill,g_face,f_webp/v1234/SnapGallery/abc123.jpg
```

With S3, achieving this would require:
- A Lambda function triggered on S3 PutObject to generate multiple resized variants (thumbnail, medium, large).
- Or a Lambda@Edge function on CloudFront that intercepts requests and transforms on-the-fly using Sharp.
- Additional S3 storage for each derived variant.
- Logic in the application layer to construct the correct variant URL.

This is an entire infrastructure subsystem that Cloudinary eliminates.

**Reason 3: Credential Simplicity**

The Cloudinary configuration ([cloudinary.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/utils/cloudinary.js)) requires exactly 3 environment variables:
```js
cloudinary.config({
  cloud_name: process.env.CLOUD_NAME,    // public identifier
  api_key: process.env.CLOUD_API_KEY,     // public API key
  api_secret: process.env.CLOUD_API_SECRET // private secret
});
```

With AWS S3, the equivalent setup would require:
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (or an IAM role if on EC2/ECS)
- `AWS_REGION`
- `S3_BUCKET_NAME`
- CloudFront distribution ID (for cache invalidation)
- CloudFront domain name (for serving URLs)

And critically, AWS IAM policies are JSON documents that require careful scoping. A misconfigured policy can either lock out the application or, worse, expose the bucket publicly (which has caused countless high-profile data breaches).

**Reason 4: Cost at Portfolio/Project Scale**

Cloudinary's free tier provides 25 credits/month, which translates to approximately:
- 25,000 transformations or
- 25 GB of managed storage or
- 25 GB of bandwidth

For a portfolio project or interview demo, this is more than sufficient. S3's pricing model (per-GB storage + per-request + per-GB egress) would be negligible in cost, but the **operational overhead** of managing IAM, CloudFront, and bucket policies for a demo project is disproportionate to the value.

---

### Counter-Argument: When S3 + CloudFront Is the Superior Choice

| Factor | Cloudinary Wins | S3 + CloudFront Wins |
|--------|----------------|---------------------|
| Setup complexity | ✅ 3 env vars, done | ❌ IAM, bucket, CDN, certs |
| Image transformations | ✅ URL-based, zero code | ❌ Requires Lambda/Sharp |
| Cost at scale (1M+ images) | ❌ Credit-based pricing escalates | ✅ Per-GB pricing is cheaper |
| Storage lifecycle (archival) | ❌ No tiered storage | ✅ S3 Glacier, Intelligent-Tiering |
| Cross-region replication | ❌ Vendor-managed, no control | ✅ Full CRR control |
| Vendor lock-in | ❌ Cloudinary-specific URLs | ✅ S3 is nearly an industry standard |
| Regulatory compliance | ❌ Less granular audit controls | ✅ Full CloudTrail, VPC endpoints |
| Egress cost control | ❌ Bundle pricing | ✅ Reserve pricing, PrivateLink |

**Key Interview Phrase:**
> "At this scale, Cloudinary was the pragmatic choice — it eliminates the entire CDN and image transformation infrastructure. At scale, I'd migrate to S3 + CloudFront for cost efficiency and use pre-signed URLs for direct-to-S3 uploads, bypassing the Node server entirely."

---

### What About Local Disk Storage?

**Why local disk was never a viable option:**

1. **Ephemeral file systems on modern PaaS.** The project is deployed on Vercel (see [README.md#L3](file:///c:/Users/ARPRIT/Desktop/SnapGallery/README.md#L3) — live demo link). Vercel, Render, and Heroku all use ephemeral containers — any file written to disk is lost on the next deploy or dyno restart. Storing images locally would mean permanent data loss.

2. **No CDN delivery.** Local disk serves files from the application server. Every image request consumes the same bandwidth and CPU as API requests. Under load, image serving competes with API responsiveness.

3. **Vertical scaling ceiling.** A single server has finite disk space. Cloud storage (Cloudinary or S3) scales to petabytes without the application being aware.

4. **No redundancy.** A disk failure = permanent data loss. Cloud storage providers replicate across multiple availability zones by default.

5. **CORS and static serving complexity.** Express would need `express.static()` middleware for the uploads directory, proper Cache-Control headers, and potentially reverse-proxy configuration (nginx) for production efficiency.

**The only scenario where local disk makes sense:** Development/testing environments where you want zero external dependencies and instant feedback.

---

## 2. Upload Pipeline: `memoryStorage` vs. `diskStorage`

### Q: "Why use Multer with `memoryStorage` instead of `diskStorage`? What are the exact trade-offs regarding Node.js RAM usage and disk I/O bottlenecks?"

**The Decision in Code:**

[posts.js (routes) — Line 7](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js#L7):
```js
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });
```

This is then applied as middleware on the POST route ([posts.js — Line 12](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js#L12)):
```js
router.post('/', upload.single('image'), createPost);
```

And consumed in the controller ([postController.js — Line 23](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L23)):
```js
const result = await uploadToCloudinary(req.file.buffer);
```

**Answer — Deep Technical Comparison:**

### How `memoryStorage` Works Internally

1. **Multer's `memoryStorage` engine** allocates a Node.js `Buffer` object (backed by V8's `ArrayBuffer`) in the heap.
2. As the multipart form data streams in from the client, Multer's internal `BusboyStream` parser accumulates chunks into this buffer.
3. Once the entire file is received, `req.file.buffer` is set to the complete `Buffer`, and control passes to the next middleware.
4. The buffer lives in V8's **old-generation heap** (since it's a large allocation). It remains there until all references are released — specifically, until the request handler completes and `req` is garbage collected.

### How `diskStorage` Would Work Instead

1. Multer writes incoming chunks to a temporary file in `os.tmpdir()` (or a configured `destination` path).
2. `req.file.path` is set to the temp file path (no `req.file.buffer`).
3. The controller would need to create a `fs.createReadStream(req.file.path)` and pipe it to the Cloudinary upload stream.
4. After the upload completes, the temp file must be explicitly deleted (via `fs.unlink()`), or it becomes an orphan.

### Side-by-Side Comparison

| Dimension | `memoryStorage` (Current) | `diskStorage` (Alternative) |
|-----------|---------------------------|---------------------------|
| **Where file lives during processing** | V8 heap (RAM) | OS filesystem (disk) |
| **I/O operations** | 0 disk syscalls | Write to temp → Read from temp → Delete temp (3 syscalls minimum) |
| **Latency per upload** | Lower (no disk round-trip) | Higher (disk write is typically 1-10ms per MB on SSD, orders of magnitude more on HDD) |
| **Memory consumption per request** | `fileSize` bytes held in heap | Minimal — only a small streaming buffer (~64KB) in memory |
| **Concurrent upload ceiling** | Limited by available heap (see OOM analysis below) | Limited by disk throughput and available disk space |
| **Cleanup complexity** | Zero — GC handles it automatically | Manual `fs.unlink()` required; failure = orphaned temp files |
| **Resumability on failure** | None — buffer is lost on crash | File persists on disk; could theoretically retry from it |
| **Streaming capability** | Must hold entire file before forwarding | Can pipe readStream → uploadStream chunk-by-chunk (constant memory) |

### The Exact RAM Trade-Off Calculation

Given the current configuration:
- `fileSize` limit: **5 MB** (5 * 1024 * 1024 = 5,242,880 bytes)
- Node.js default heap limit: **~1.5 GB** (V8 default, adjustable via `--max-old-space-size`)
- Free-tier server RAM (Render, Heroku): **512 MB**

**Scenario Analysis:**

| Concurrent Uploads | RAM Used (buffers only) | % of 512 MB Instance | Risk Level |
|--------------------|-----------------------|---------------------|------------|
| 1 | 5 MB | 1% | ✅ Safe |
| 10 | 50 MB | 10% | ✅ Safe |
| 50 | 250 MB | 49% | ⚠️ Warning — other processes need RAM too |
| 100 | 500 MB | 98% | 🔴 OOM imminent — OS needs RAM, Node runtime overhead ~50MB |
| 200 | 1 GB | 200% | 💀 Process killed by OOM killer |

**Critical nuance:** The 5 MB figure is the *maximum* — most photo uploads will be 1-3 MB. But the `memoryStorage` engine buffers the **entire** file before Multer passes control to the next handler. There is no streaming during reception — the buffer must be fully constructed first.

### Why `memoryStorage` Was Still the Right Choice Here

1. **The file size limit is strict** — 5 MB cap means even worst-case scenarios are bounded. The `limits` option in Multer ([posts.js#L7](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/routes/posts.js#L7)) rejects larger files with a `MulterError` before the buffer grows further.

2. **The deployment target is a portfolio project** — concurrent uploads of 50+ are unrealistic for a demo. The design optimizes for latency and simplicity, not throughput at scale.

3. **Eliminates an entire category of bugs** — with `diskStorage`, you must handle:
   - Temp directory permissions (`EACCES` errors)
   - Disk full scenarios (`ENOSPC`)
   - Orphaned temp files from failed uploads
   - Cross-platform path differences (Windows vs. Linux temp dirs)
   - File name collisions (Multer uses random hex names by default, but collision is theoretically possible)

4. **Single-hop streaming to Cloudinary** — The `uploadToCloudinary` function ([postController.js#L4-L15](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L4-L15)) takes the buffer and ends the writable stream:
   ```js
   stream.end(buffer);
   ```
   This pushes the entire buffer into the Cloudinary upload stream in a single call. With `diskStorage`, this would be:
   ```js
   const readStream = fs.createReadStream(req.file.path);
   readStream.pipe(cloudinaryStream);
   ```
   While the piped approach is more memory-efficient, it adds error handling complexity (what if `readStream` emits an error? What if the pipe breaks mid-transfer?).

### Production-Grade Migration Path (If Scaling Beyond Portfolio)

**Phase 1: Rate-limit + keep `memoryStorage`**
```js
import rateLimit from 'express-rate-limit';
const uploadLimiter = rateLimit({ windowMs: 60 * 1000, max: 5 }); // 5 uploads/min per IP
router.post('/', uploadLimiter, upload.single('image'), createPost);
```

**Phase 2: Switch to `diskStorage` + streaming for server-relayed uploads**
```js
const upload = multer({ 
  storage: multer.diskStorage({ destination: '/tmp/uploads' }),
  limits: { fileSize: 10 * 1024 * 1024 }
});
// In controller:
const readStream = fs.createReadStream(req.file.path);
readStream.pipe(cloudinaryUploadStream);
readStream.on('end', () => fs.unlink(req.file.path, () => {}));
```

**Phase 3: Bypass the server entirely with direct-to-Cloudinary uploads**
- Generate a signed upload preset on the server.
- Client uploads directly to `https://api.cloudinary.com/v1_1/<cloud_name>/image/upload`.
- Client sends the resulting `secure_url` + `public_id` back to the server to create the Post document.
- Server RAM usage for uploads: **zero**.

---

## 3. Database: MongoDB vs. Neo4j vs. PostgreSQL for a Social Graph

### Q: "Why MongoDB for a social graph instead of a graph database like Neo4j or a relational database like PostgreSQL?"

**The Decision in Code:**

The social graph is implemented as **embedded arrays of ObjectId references** on the User schema ([User.js#L28-L35](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L28-L35)):
```js
followers: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
following: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }]
```

The database connection uses Mongoose ODM ([db.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/config/db.js)):
```js
const conn = await mongoose.connect(process.env.MONGODB_URI);
```

**Answer — The Full Reasoning:**

### Why MongoDB Was Chosen

**1. Schema Flexibility for Rapid Iteration**

SnapGallery's data model evolved during development. MongoDB's schemaless nature (enforced optionally through Mongoose) allowed rapid prototyping:
- The Post schema ([Post.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js)) has only 4 fields: `author`, `imageUrl`, `caption`, `likes`. Adding a field (e.g., `tags`, `location`) requires zero migrations — just update the Mongoose schema and existing documents coexist peacefully.
- With PostgreSQL, every schema change requires an `ALTER TABLE` migration, a migration runner (Knex, Prisma Migrate, TypeORM), and careful handling of default values for existing rows.

**2. Document Model Naturally Fits the Post Entity**

A Post is a self-contained entity: it has an author reference, an image URL, a caption, and an array of likes. In MongoDB, this is a single document:
```json
{
  "_id": "ObjectId(...)",
  "author": "ObjectId(userId)",
  "imageUrl": "https://res.cloudinary.com/...",
  "caption": "Sunset at the beach",
  "likes": ["ObjectId(user1)", "ObjectId(user2)"],
  "createdAt": "2026-08-13T...",
  "updatedAt": "2026-08-13T..."
}
```

In PostgreSQL, this would require at minimum:
- A `posts` table (id, author_id, image_url, caption, created_at)
- A `post_likes` junction table (post_id, user_id) for the many-to-many likes relationship
- A JOIN query or subquery to get the like count

The MongoDB approach collapses the like relationship into the document itself, which means a single document read returns the complete post data including all likes.

**3. Mongoose ODM Provides Schema Enforcement Where Needed**

While MongoDB is schemaless, Mongoose provides:
- **Type validation:** `required: true`, `type: String`, `maxlength: 500` ([Post.js#L14-L16](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js#L14-L16))
- **Pre-save hooks:** Password hashing before write ([User.js#L38-L41](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L38-L41))
- **Instance methods:** `comparePassword()`, `toJSON()` for encapsulated behavior ([User.js#L43-L51](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L43-L51))
- **Population (pseudo-JOINs):** `.populate('author', 'username profilePic')` hydrates ObjectId references into actual document data

This gives us the benefits of schema enforcement without the rigidity of SQL migrations.

**4. The `$in` Operator Handles the Feed Query Elegantly**

The feed generation query ([postController.js#L43-L46](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L43-L46)):
```js
const posts = await Post.find({ author: { $in: following } })
  .sort({ createdAt: -1 })
  .populate('author', 'username profilePic')
  .limit(50);
```

MongoDB's `$in` operator on an indexed field performs efficient multi-key lookups on the B-tree index. For a typical user following 50-200 accounts, this query is fast and straightforward.

---

### Comprehensive Comparison: MongoDB vs. Neo4j vs. PostgreSQL

| Dimension | MongoDB (Current) | Neo4j (Graph DB) | PostgreSQL (Relational) |
|-----------|-------------------|-------------------|------------------------|
| **Social graph storage** | Embedded arrays on User document | Native nodes (User) + edges (FOLLOWS) | `follows` junction table (follower_id, following_id) |
| **"Does A follow B?" query** | `user.following.includes(B_id)` — O(N) array scan in application code ([userController.js#L15](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L15)) | `MATCH (a:User)-[:FOLLOWS]->(b:User) RETURN true` — O(1) edge traversal | `SELECT 1 FROM follows WHERE follower_id = A AND following_id = B` — O(1) with index |
| **"Friends of friends" query** | Requires multiple application-level queries and manual deduplication | `MATCH (a)-[:FOLLOWS]->()-[:FOLLOWS]->(fof) RETURN fof` — single query, native traversal | Recursive CTE or self-JOIN — expensive but possible |
| **Feed generation** | `Post.find({ author: { $in: following } })` — single query | `MATCH (me)-[:FOLLOWS]->(f) MATCH (f)-[:POSTED]->(p) RETURN p ORDER BY p.createdAt DESC` | `SELECT * FROM posts WHERE author_id IN (SELECT following_id FROM follows WHERE follower_id = me) ORDER BY created_at DESC` |
| **Schema flexibility** | Schemaless + optional Mongoose validation | Schema-optional (node labels + properties) | Rigid schema, requires migrations |
| **Transactions** | Multi-document transactions (requires replica set) | Full ACID transactions | Full ACID transactions by default |
| **Scaling** | Horizontal sharding (native) | Clustering (limited horizontal scale) | Vertical scaling primarily; horizontal via Citus/read replicas |
| **Ecosystem maturity** | Massive — Mongoose, Atlas, MongoDB Compass | Niche — smaller community, fewer ORMs | Massive — Sequelize, Prisma, TypeORM, Knex |
| **Operational complexity** | Low (Atlas manages everything) | Medium-high (self-managed or Aura cloud) | Low-medium (RDS, Supabase, Neon) |
| **Best for this project** | ✅ General-purpose data + simple social graph | ❌ Overkill — no complex graph traversals needed | ⚠️ Viable but adds migration overhead |

### Why Not Neo4j?

Neo4j excels at **deep graph traversals** — "find all users within 3 degrees of connection," "recommend friends based on mutual connections," "detect community clusters." SnapGallery's social graph operations are shallow:
- Follow/unfollow (1 hop)
- Get followers/following list (1 hop)
- Generate feed from followed users' posts (1 hop + post lookup)

None of these require Neo4j's Cypher query language or its native index-free adjacency traversal engine. Using Neo4j would introduce:
- A separate database to manage (connection pooling, backups, monitoring)
- A completely different query language (Cypher vs. MongoDB query language)
- Operational overhead for a graph that can be represented as simple arrays
- Potential need for two databases: Neo4j for the graph + MongoDB/PostgreSQL for post content

**When Neo4j becomes necessary:** If SnapGallery evolved to include features like "suggested users based on mutual follows," "trending within your network (2nd-degree connections)," or "cluster analysis for explore page content," then a graph database would be architecturally justified.

### Why Not PostgreSQL?

PostgreSQL is a perfectly viable choice. The decision against it was pragmatic, not technical:

1. **Schema rigidity adds friction.** Every schema change (adding `tags` to posts, adding `bio` to users) requires a migration file, a migration runner, and production deployment of the migration before the code that uses the new field.

2. **No native document population.** The `.populate()` calls throughout the codebase (e.g., [postController.js#L31](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L31), [userController.js#L42-L43](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/userController.js#L42-L43)) would become explicit JOIN queries:
   ```sql
   SELECT p.*, u.username, u.profile_pic 
   FROM posts p 
   JOIN users u ON p.author_id = u.id 
   WHERE p.author_id = ANY($1) 
   ORDER BY p.created_at DESC 
   LIMIT 50;
   ```
   This is more verbose but equally performant with proper indexing.

3. **The likes relationship is more complex in SQL.** Currently, `post.likes` is an embedded array ([Post.js#L18-L21](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/Post.js#L18-L21)). In PostgreSQL:
   - Separate `post_likes` table: `(post_id, user_id, created_at)`
   - Toggle like: `INSERT ... ON CONFLICT DO DELETE` or separate endpoints
   - Get like count: `SELECT COUNT(*) FROM post_likes WHERE post_id = $1`
   - Check if user liked: `SELECT 1 FROM post_likes WHERE post_id = $1 AND user_id = $2`

**When PostgreSQL is the better choice:**
- When ACID transactions are critical (e.g., financial data, strict consistency requirements)
- When complex reporting queries are needed (PostgreSQL's query planner is superior for analytical workloads)
- When the data model is well-defined and stable (no frequent schema changes)
- When the team has strong SQL expertise
- When using an ORM like Prisma that generates type-safe queries from the schema

---

## 4. Framework: Express 5 vs. Fastify vs. NestJS

### Q: "Why Express 5? What about Fastify or NestJS?"

**The Decision in Code:**

[package.json#L19](file:///c:/Users/ARPRIT/Desktop/SnapGallery/package.json#L19):
```json
"express": "^5.2.1"
```

[server.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/server.js) — The server setup is 24 lines total, demonstrating Express's minimal boilerplate.

**Answer:**

**Why Express 5 specifically (not Express 4):**
- Express 5 natively supports `async/await` in route handlers and middleware. If an async handler throws, Express 5 automatically calls `next(err)` — no `try/catch` wrapper or `express-async-errors` package needed.
- However, the current codebase **still uses explicit try/catch** in every controller (e.g., [postController.js#L18-L35](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/postController.js#L18-L35)). This is a belt-and-suspenders approach — defensive coding, not redundant with Express 5.

**Why not Fastify:**
- Fastify offers ~2x throughput over Express for JSON serialization due to schema-based serialization (`fast-json-stringify`).
- However, for SnapGallery's workload (media uploads, DB queries, Cloudinary API calls), the bottleneck is I/O latency, not JSON serialization speed.
- Fastify's ecosystem is smaller — fewer middleware options, less community content.

**Why not NestJS:**
- NestJS provides architectural guardrails (modules, decorators, dependency injection, guards).
- For a 3-controller, 2-model application, NestJS would add ~10x more boilerplate (module files, DTO classes, injectable services) for minimal structural benefit.
- NestJS is optimal for large teams and complex applications where enforced structure prevents architectural drift.

---

## 5. Authentication: JWT (Stateless) vs. Sessions (Stateful)

### Q: "Why stateless JWT authentication instead of server-side sessions?"

**The Decision in Code:**

Token generation — [authController.js#L4-L6](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/controllers/authController.js#L4-L6):
```js
const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '7d' });
};
```

Token verification — [middleware/auth.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/middleware/auth.js):
```js
const decoded = jwt.verify(token, process.env.JWT_SECRET);
const user = await User.findById(decoded.id).select('-password');
```

**Answer:**

| Factor | JWT (Current) | Server Sessions |
|--------|--------------|-----------------|
| **State storage** | None on server (token contains claims) | Requires session store (Redis, DB) |
| **Horizontal scaling** | Trivial — any server can verify the token | Requires shared session store or sticky sessions |
| **Token revocation** | Not natively supported (7-day window) | Immediate — delete session from store |
| **Cross-domain support** | `Authorization` header works across origins | Requires `SameSite=None; Secure` cookie config |
| **Infrastructure** | Zero additional services | Redis cluster for session store |
| **Security exposure** | Token theft = full access for 7 days | Session theft = access until logout/expiry |

**Why JWT was chosen:** The application serves a static frontend (Vanilla JS) that communicates with the API via `fetch` with `Authorization` headers (see client-side files like [feed.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/public/js/feed.js)). JWT allows the frontend and API to be deployed on different domains without complex cookie configuration.

**Hybrid approach at scale:** Short-lived JWTs (15-minute expiry) + refresh tokens stored in `httpOnly` cookies. The access token handles stateless verification; the refresh token enables revocation and rotation.

---

## 6. Module System: ES Modules vs. CommonJS

### Q: "Why ES Modules (`import/export`) instead of CommonJS (`require/module.exports`)?"

**The Decision in Code:**

[package.json#L5](file:///c:/Users/ARPRIT/Desktop/SnapGallery/package.json#L5):
```json
"type": "module"
```

Every file uses `import`/`export`:
```js
import express from 'express';           // server.js#L2
import mongoose from 'mongoose';         // User.js#L1
import { v2 as cloudinary } from 'cloudinary'; // cloudinary.js#L1
```

**Answer:**

1. **ES Modules are the JavaScript standard** — they are part of the ECMAScript specification, while CommonJS is Node.js-specific.
2. **Tree-shaking support** — ES Modules enable static analysis for dead code elimination, critical for frontend bundlers but increasingly relevant for server-side optimization.
3. **Top-level `await`** — ES Modules support `await` at the module level, though not used in this codebase currently.
4. **Named imports** — `import { Router } from 'express'` is more explicit than `const { Router } = require('express')`.
5. **Forward compatibility** — Node.js is moving toward ESM as the default. Starting with ESM avoids a future migration.

**Trade-off:** Some older packages may not have proper ESM exports, requiring workarounds. The current dependency set ([package.json#L14-L23](file:///c:/Users/ARPRIT/Desktop/SnapGallery/package.json#L14-L23)) — Express, Mongoose, Cloudinary, Multer, bcryptjs, jsonwebtoken — all support ESM imports.

---

## 7. Password Hashing: bcryptjs vs. argon2 vs. scrypt

### Q: "Why bcryptjs? Why not argon2 (the Password Hashing Competition winner) or Node.js's native scrypt?"

**The Decision in Code:**

[User.js#L2](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L2):
```js
import bcrypt from 'bcryptjs';
```

[User.js#L40](file:///c:/Users/ARPRIT/Desktop/SnapGallery/src/models/User.js#L40):
```js
this.password = await bcrypt.hash(this.password, 10);
```

**Answer:**

| Factor | bcryptjs | argon2 (argon2id) | Node.js crypto.scrypt |
|--------|----------|-------------------|-----------------------|
| **Native binary deps** | None (pure JavaScript) | Requires C++ compilation (`node-gyp`) | Native to Node.js (no install) |
| **Memory hardness** | No (CPU-only) | Yes (configurable memory cost) | Yes (configurable memory cost) |
| **GPU resistance** | Moderate (Blowfish is not GPU-friendly) | Strong (memory-hard defeats GPU parallelism) | Strong |
| **Industry adoption** | Ubiquitous (default choice for years) | Growing (recommended by OWASP since 2019) | Less common in web apps |
| **Cross-platform deployment** | Zero issues (pure JS) | Compilation issues on Alpine Linux, ARM, Windows | Zero issues (built-in) |
| **Performance** | ~100ms at cost 10 | ~100ms at default params | ~100ms with recommended params |

**Why bcryptjs specifically (not `bcrypt`):** The `bcryptjs` package is a **pure JavaScript implementation** — no native C++ addon, no `node-gyp` compilation step. This means zero deployment friction on any platform. The `bcrypt` npm package (with C++ bindings) is faster (~30%) but can fail to compile on Alpine Linux Docker images or ARM-based cloud instances.

**Why not argon2:** Argon2 is technically superior (it won the Password Hashing Competition and provides memory-hardness that defeats GPU-based attacks). However:
- The `argon2` npm package requires `node-gyp` and C++ compilation, which adds deployment complexity.
- For a portfolio project, the attack vector (GPU-based brute force against password hashes) is not a realistic threat.
- bcryptjs with cost factor 10 is still OWASP-approved and industry-standard.

---

## 8. Deployment Architecture: Monolith vs. Microservices

### Q: "Why a monolithic architecture? When would you decompose into microservices?"

**The Decision in Code:**

[server.js](file:///c:/Users/ARPRIT/Desktop/SnapGallery/server.js) — All three route groups (`/api/auth`, `/api/posts`, `/api/users`) are mounted on a single Express instance:
```js
app.use('/api/auth', authRoutes);
app.use('/api/posts', postRoutes);
app.use('/api/users', userRoutes);
```

**Answer:**

The monolith is correct here because:
1. **3 controllers, 2 models, 1 database** — the cognitive overhead of microservices (service discovery, inter-service communication, distributed transactions, centralized logging, health checks, circuit breakers) would dwarf the application logic.
2. **Single deployment artifact** — one `npm start` runs the entire application. No Docker Compose orchestration, no Kubernetes manifests, no API gateway configuration.
3. **Shared database** — Auth, posts, and users all query the same MongoDB instance. Splitting into services but sharing the database creates the worst of both worlds (tight coupling + network overhead).

**When to decompose:**
- **Upload service** — If upload traffic spikes independently from read traffic, extract the upload pipeline into a separate service (or serverless function) that scales independently.
- **Feed service** — If feed computation becomes expensive (fan-out-on-write, ranking algorithms), extract it into a service with its own caching layer (Redis).
- **Auth service** — If multiple applications need to authenticate against the same user base, extract auth into a shared service with RSA-signed JWTs.

**Key Interview Phrase:**
> "The monolith is the right starting point. I'd decompose only when I have evidence of independent scaling needs — specifically, the upload pipeline would be the first candidate for extraction, since it has distinct memory and CPU characteristics from the read-heavy API routes."

---
