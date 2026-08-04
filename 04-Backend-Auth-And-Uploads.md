# 04 — Backend Auth & Uploads

## Part 1: JWT Authentication — The Full Story

### The Problem: How Do You Keep a User "Logged In"?

HTTP is **stateless**. Each request the browser sends to the server is completely independent. The server doesn't remember who sent the previous request. So how does the server know that request #2 is from the same user who logged in during request #1?

### The Old Way: Server-Side Sessions (Cookies)

```
1. User logs in → Server creates a "session" object in memory/database
2. Server sends a unique session ID as a cookie
3. Browser auto-attaches this cookie to every subsequent request
4. Server receives the cookie, looks up the session → "Ah, this is user #42"
```

**Analogy:** You walk into a library. The librarian gives you a numbered plastic card (cookie). Every time you come back, you show the card. The librarian looks up card #42 in their ledger (session store) and says, "Ah, you're Arpit. Here are your books."

**Problems with sessions:**
- The server must store session data in memory or a database (Redis, etc.)
- If you have 3 servers behind a load balancer, all 3 need access to the same session store
- Scaling horizontally is harder

### The New Way: JWTs (This Project's Approach)

```
1. User logs in → Server creates a JWT containing the user's ID
2. Server signs the JWT with a secret key and sends it to the browser
3. Browser stores the JWT in localStorage
4. Browser manually attaches the JWT in the Authorization header on every request
5. Server receives the JWT, verifies the signature → "This token is valid, user ID is abc123"
```

**Analogy:** Instead of giving you a library card, the librarian writes your name, membership level, and expiry date on a piece of paper, then stamps it with the library's official seal (signature). Every time you come back, you show the paper. The librarian doesn't need to look anything up — they just verify the seal is genuine. If someone tampers with the paper (changes the name), the seal won't match and they'll know it's fake.

### What's Inside a JWT?

A JWT has three parts, separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjY2YThmM2IyIn0.Kj8mN_signature
└──── Header ──────┘  └──── Payload ───────┘  └── Signature ──┘
```

1. **Header** — Algorithm used (HS256) and token type (JWT). Base64-encoded.
2. **Payload** — The data you put in. In this project: `{ id: "user's MongoDB _id" }`. Base64-encoded.
3. **Signature** — `HMAC-SHA256(header + "." + payload, JWT_SECRET)`. This is the tamper-proof seal.

**Key insight:** The payload is NOT encrypted. It's only Base64-encoded, which means anyone can decode it and read the user ID. The signature ensures that nobody can **modify** the payload without the server's secret key.

### Token Generation in This Project

```javascript
// src/controllers/authController.js
const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '7d' });
};
```

- `{ id }` — Shorthand for `{ id: id }`. Embeds the user's MongoDB `_id` in the payload.
- `process.env.JWT_SECRET` — The secret key from the `.env` file. **Never commit this to Git.**
- `{ expiresIn: '7d' }` — Token auto-expires after 7 days. After that, `jwt.verify()` will throw an error.

### Token Verification — The Auth Middleware

```javascript
// src/middleware/auth.js
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const auth = async (req, res, next) => {
  try {
    const header = req.headers.authorization;
```

Read the `Authorization` header from the incoming request. It looks like: `"Bearer eyJhbGciOiJIUzI1NiJ9..."`.

```javascript
    if (!header || !header.startsWith('Bearer ')) {
      return res.status(401).json({ message: 'No token provided' });
    }
```

If there's no header, or it doesn't start with `"Bearer "`, reject immediately. The `"Bearer "` prefix is a convention from the OAuth 2.0 standard.

```javascript
    const token = header.split(' ')[1];
```

`"Bearer eyJhbGci..."`.split(' ')` produces `["Bearer", "eyJhbGci..."]`. Index `[1]` gives us just the token.

```javascript
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
```

`jwt.verify()` does three things:
1. Checks the signature — Was this token signed with our secret key?
2. Checks expiration — Has `expiresIn: '7d'` been exceeded?
3. Decodes the payload — Returns `{ id: "abc123", iat: 1720..., exp: 1721... }`

If any check fails, it throws an error, which is caught by the `catch` block.

```javascript
    const user = await User.findById(decoded.id).select('-password');
```

Use the `id` from the decoded payload to look up the user in MongoDB. `.select('-password')` excludes the password hash.

```javascript
    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }
```

Edge case: The token is valid, but the user was deleted from the database after the token was issued. The token is "technically valid" but refers to a nonexistent user.

```javascript
    req.user = user;
    next();
```

Attach the full user document to `req.user`. Now every subsequent handler (`getMe`, `createPost`, `toggleLike`, etc.) can access the authenticated user via `req.user`.

`next()` passes control to the next middleware or route handler. Without calling `next()`, the request would hang forever.

```javascript
  } catch {
    res.status(401).json({ message: 'Invalid token' });
  }
};
```

If anything goes wrong (invalid signature, expired token, malformed token), catch the error and return 401.

### JWT vs Cookies — Summary

| Feature               | Session/Cookie                  | JWT (this project)              |
| --------------------- | ------------------------------- | ------------------------------- |
| Storage (server)      | Session store (memory/Redis/DB) | Nothing — stateless             |
| Storage (client)      | Cookie (auto-sent by browser)   | localStorage (manually sent)    |
| Scalability           | Harder (shared session store)   | Easy (any server can verify)    |
| Logout                | Delete session on server        | Delete token from localStorage  |
| Revocation            | Easy (delete session)           | Hard (token valid until expiry) |
| Cross-domain          | Complex (cookie policies)       | Simple (just a header)          |

---

## Part 2: Password Hashing with bcrypt

### Why Hash Passwords?

If you store passwords as plain text:
```
username: "arpit"
password: "mypassword123"    ← Anyone who accesses the DB can read this
```

If you hash them:
```
username: "arpit"
password: "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"
```

Even if the database is leaked/hacked, the attacker gets hashes, not passwords. They can't login with the hash.

### What bcrypt Does Differently from Simple Hashing

Simple hash (like SHA-256): `"password"` → always produces the same hash. Attackers can use pre-computed tables ("rainbow tables") to reverse common passwords.

bcrypt adds a **random salt** to each password before hashing:
```
bcrypt("password", salt1) → "$2a$10$abc123..."
bcrypt("password", salt2) → "$2a$10$xyz789..."   ← Different hash for same password!
```

The salt is embedded in the hash string itself, so `bcrypt.compare()` knows which salt was used.

### The Flow in This Project

**Registration:**
```javascript
// User.js — pre('save') hook
this.password = await bcrypt.hash(this.password, 10);
// "mypassword123" → "$2a$10$N9qo8u..."
```

**Login:**
```javascript
// authController.js — login()
const isMatch = await user.comparePassword(password);

// User.js — comparePassword method
return bcrypt.compare(candidatePassword, this.password);
// bcrypt.compare("mypassword123", "$2a$10$N9qo8u...") → true
```

---

## Part 3: The Multer → Cloudinary Image Pipeline

### The Problem

Users upload images. Where do you store them? Three options:

1. **Server's hard drive** — Simple, but if you have multiple servers, each has different files. Plus, server disk space is limited and expensive.
2. **MongoDB** — You can store binary data, but databases aren't designed for large file storage. It's slow and bloats the DB.
3. **Cloud storage (Cloudinary)** — Images are stored on a CDN (Content Delivery Network). Fast, scalable, global. You store the URL in MongoDB.

This project uses option 3.

### The Challenge: Not Saving to Disk

The typical file upload flow is:
```
Browser → Upload file → Save to /tmp/upload.jpg on server → Read file → Upload to Cloudinary → Delete temp file
```

But this project uses **Multer Memory Storage**, which skips the disk entirely:
```
Browser → Upload file → Hold in RAM (Buffer) → Stream directly to Cloudinary
```

**Analogy:** Imagine you're a delivery person. The old way: you receive a package, put it in your garage, then later drive it to the final destination, then clean out your garage. The new way: you receive the package in your hands and immediately pass it to the next delivery person. The package never touches the ground.

### Step 1: Multer Configuration

```javascript
// src/routes/posts.js
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 }
});
```

**`multer.memoryStorage()`** — Instead of writing to disk (`multer.diskStorage()`), the uploaded file is stored as a `Buffer` object in memory. A `Buffer` is Node.js's way of representing raw binary data.

After Multer processes the request, `req.file` looks like this:

```javascript
req.file = {
  fieldname: 'image',
  originalname: 'sunset.jpg',
  encoding: '7bit',
  mimetype: 'image/jpeg',
  buffer: <Buffer ff d8 ff e0 00 10 4a 46 49 46 ...>,   // ← The raw bytes
  size: 2458372
}
```

`req.file.buffer` is the actual image data — a sequence of bytes in RAM. No file was ever written to `/tmp` or any directory.

### Step 2: Cloudinary Configuration

```javascript
// src/utils/cloudinary.js
import { v2 as cloudinary } from 'cloudinary';

cloudinary.config({
  cloud_name: process.env.CLOUD_NAME,
  api_key: process.env.CLOUD_API_KEY,
  api_secret: process.env.CLOUD_API_SECRET
});

export default cloudinary;
```

This initializes the Cloudinary SDK with your account credentials from the `.env` file. `v2` is Cloudinary's current API version. The `as cloudinary` renames the import for convenience.

### Step 3: Streaming Upload to Cloudinary

```javascript
// src/controllers/postController.js
const uploadToCloudinary = (buffer) => {
  return new Promise((resolve, reject) => {
```

Wraps the upload in a Promise because Cloudinary's `upload_stream` uses a **callback pattern** (older style), but the rest of our code uses `async/await` (modern style). A Promise bridges the two.

```javascript
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'SnapGallery' },
```

`upload_stream` returns a **writable stream** — a pipe you can push data into. `{ folder: 'SnapGallery' }` tells Cloudinary to organize this image under a folder called "SnapGallery" in your cloud storage.

```javascript
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );
```

This callback fires when the upload is complete. If it failed, the Promise rejects (triggers the `catch` block). If it succeeded, the Promise resolves with the `result` object, which contains:

```javascript
result = {
  secure_url: "https://res.cloudinary.com/mycloud/image/upload/v123/SnapGallery/abc.jpg",
  public_id: "SnapGallery/abc",
  format: "jpg",
  width: 1920,
  height: 1080,
  bytes: 2458372,
  // ... more metadata
}
```

The `secure_url` is the HTTPS link to the image, hosted on Cloudinary's global CDN.

```javascript
    stream.end(buffer);
```

This is the magic line. `stream.end(buffer)` does two things:
1. **Writes** the buffer data to the stream
2. **Signals** that there's no more data coming (end of stream)

The entire buffer (the image in RAM) is pushed into the Cloudinary upload stream in one shot. No intermediate file, no temp directory.

### Step 4: Using the Upload Result

```javascript
// In createPost()
const result = await uploadToCloudinary(req.file.buffer);

const post = await Post.create({
  author: req.user._id,
  imageUrl: result.secure_url,    // ← Store the URL, not the image itself
  caption: req.body.caption || ''
});
```

Only the **URL** is stored in MongoDB. The actual image lives on Cloudinary's servers. When the frontend needs to display the image, it uses this URL as the `src` attribute of an `<img>` tag.

### The Complete Pipeline — Visualized

```
┌──────────┐    FormData     ┌──────────┐   Memory Buffer   ┌──────────┐
│ Browser  │ ──────────────► │  Multer  │ ─────────────────► │ Buffer   │
│ (upload  │   (multipart)   │ (middle- │   (req.file.buf)   │ in RAM   │
│  form)   │                 │  ware)   │                    │          │
└──────────┘                 └──────────┘                    └────┬─────┘
                                                                  │
                                                     stream.end(buffer)
                                                                  │
                                                                  ▼
┌──────────┐  Store URL      ┌──────────┐   Upload stream   ┌──────────┐
│ MongoDB  │ ◄────────────── │ Control- │ ◄───────────────── │Cloudinary│
│ (Post    │  imageUrl field  │ ler      │   secure_url       │ (CDN)    │
│  doc)    │                 │          │                    │          │
└──────────┘                 └──────────┘                    └──────────┘
```

### Why Memory Storage + Streaming?

| Approach            | Disk I/O | Cleanup Needed? | Speed    | Server Disk Usage |
| ------------------- | -------- | --------------- | -------- | ----------------- |
| Disk Storage        | Yes (2x) | Yes (delete tmp)| Slower   | Temporary files   |
| Memory Storage      | No       | No (GC handles) | Faster   | Zero              |

**Tradeoff:** Memory storage uses RAM. If 100 users upload 5MB images simultaneously, that's 500MB of RAM. For a small app this is fine, but at scale you'd add file size limits and potentially use disk storage with cleanup.

This project already has the safeguard:
```javascript
limits: { fileSize: 5 * 1024 * 1024 }  // Max 5MB per upload
```

### What is `multipart/form-data`?

When the frontend uploads a file, it can't use `application/json` — JSON can't represent binary file data efficiently. Instead, it uses `multipart/form-data`, a special encoding that allows mixing text fields and binary files in a single request.

```
POST /api/posts
Content-Type: multipart/form-data; boundary=----FormBoundary

------FormBoundary
Content-Disposition: form-data; name="image"; filename="sunset.jpg"
Content-Type: image/jpeg

[binary image data here]
------FormBoundary
Content-Disposition: form-data; name="caption"

Golden hour vibes 🌅
------FormBoundary--
```

Multer parses this format. It extracts:
- `req.file` → The binary image
- `req.body.caption` → The text caption

The frontend creates this format using the `FormData` API:

```javascript
const formData = new FormData();
formData.append('image', selectedFile);     // Binary file
formData.append('caption', captionText);    // Text field
```

When you use `FormData` with `fetch()`, the browser automatically sets the `Content-Type` header to `multipart/form-data` with the correct boundary. That's why in `upload.js`, the `fetch` call does **not** set `Content-Type: application/json` — it deliberately omits it so the browser can set the right one.
