# 01 — Architecture & Full Request-Response Flow

## What Is SnapGallery?

A Pinterest-style image sharing app. Users register, login, upload images, follow other users, see a personalized feed, and like/delete posts. The entire frontend is plain HTML/CSS/JavaScript — no React, no Vue, no framework. The backend is a Node.js REST API.

---

## The Two Halves of the Application

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER'S BROWSER                           │
│                                                                 │
│   index.html ──► auth.js          (Login / Register)            │
│   feed.html  ──► feed.js          (View feed, like, search)     │
│   upload.html──► upload.js        (Upload images)               │
│   profile.html─► profile.js       (View profile, follow)        │
│                                                                 │
│   All JS files use fetch() to talk to the backend               │
│   JWT token stored in localStorage                              │
└────────────────────────────┬────────────────────────────────────┘
                             │  HTTP Requests (JSON + Bearer Token)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       NODE.JS SERVER                            │
│                                                                 │
│   server.js (entry point)                                       │
│     ├── /api/auth   ──► authController.js                       │
│     ├── /api/posts  ──► postController.js  (+ Multer + Cloud.)  │
│     └── /api/users  ──► userController.js                       │
│                                                                 │
│   Middleware: auth.js (JWT verification on every protected req) │
│   Database:  MongoDB via Mongoose                               │
│   Uploads:   Multer (memory) → Cloudinary (cloud CDN)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Tech Stack — What Does Each Piece Do?

| Technology     | Role                                                                 |
| -------------- | -------------------------------------------------------------------- |
| **HTML**       | Structure of the pages. There are 4 static `.html` files.            |
| **CSS**        | One single `style.css`. Dark theme, masonry grid, responsive layout. |
| **Vanilla JS** | DOM manipulation + `fetch()` API calls. No framework.                |
| **Node.js**    | Server-side JavaScript runtime. Runs the backend.                    |
| **Express**    | HTTP framework on top of Node. Handles routing, middleware.          |
| **MongoDB**    | NoSQL database. Stores Users and Posts as JSON-like documents.       |
| **Mongoose**   | ODM (Object Data Modeling) library for MongoDB. Defines schemas.     |
| **JWT**        | JSON Web Tokens. Stateless authentication (no sessions on server).   |
| **bcryptjs**   | Hashes passwords before storing them in the database.                |
| **Multer**     | Middleware that handles `multipart/form-data` (file uploads).        |
| **Cloudinary** | Cloud image hosting service. Images are stored there, not locally.   |
| **CORS**       | Allows the frontend (different origin) to call the backend API.      |

---

## The Complete Lifecycle: User Clicks "Like" → What Happens?

Let's trace a single user action end-to-end. The user is on `feed.html` and clicks the heart button on a post.

### Step 1 — The Click (Frontend DOM)

In `feed.html`, each post card is rendered with an `onclick` attribute:

```html
<button class="like-btn" onclick="toggleLike('POST_ID_HERE', this)">
```

The `this` keyword passes the button element itself so we can update it later without reloading the page.

### Step 2 — The Fetch Request (Vanilla JS)

In `public/js/feed.js`, the `toggleLike` function fires:

```javascript
async function toggleLike(postId, btn) {
  const res = await fetch(`${BACKEND_URL}/api/posts/${postId}/like`, {
    method: 'PUT',
    headers: getHeaders()   // ← Attaches: { Authorization: 'Bearer <JWT>' }
  });
  const post = await res.json();
  // ... update the button's appearance
}
```

**What `getHeaders()` does:** It reads the JWT token from `localStorage` and attaches it as a `Bearer` token in the `Authorization` header. Without this, the server will reject the request with a `401 Unauthorized`.

### Step 3 — The Request Hits Express (Backend Routing)

The request arrives at `server.js`:

```javascript
app.use('/api/posts', postRoutes);
```

Express sees the URL starts with `/api/posts` and forwards it to `src/routes/posts.js`. Inside that file:

```javascript
router.use(auth);                      // ← ALL post routes require a valid JWT
router.put('/:id/like', toggleLike);   // ← Matches PUT /api/posts/<id>/like
```

### Step 4 — JWT Middleware Runs (Authentication Gate)

Before `toggleLike` controller runs, the `auth` middleware in `src/middleware/auth.js` executes:

```javascript
const token = header.split(' ')[1];           // Extract token from "Bearer <token>"
const decoded = jwt.verify(token, SECRET);     // Decode + verify signature
const user = await User.findById(decoded.id);  // Find the user in MongoDB
req.user = user;                               // Attach user object to the request
next();                                        // Pass control to the next handler
```

**Analogy:** Think of this middleware as a security guard at a nightclub. You show your wristband (JWT). The guard checks if it's real (verifies the signature). If it's real, the guard lets you in and notes who you are (attaches `req.user`). If it's fake or expired, you get turned away (401 error).

### Step 5 — The Controller Logic (Business Logic)

The `toggleLike` function in `src/controllers/postController.js` runs:

```javascript
const post = await Post.findById(req.params.id);  // Find the post in MongoDB
const userId = req.user._id;                       // Get the logged-in user's ID
const index = post.likes.indexOf(userId);          // Check if already liked

if (index === -1) {
  post.likes.push(userId);     // Not liked → add the like
} else {
  post.likes.splice(index, 1); // Already liked → remove the like (unlike)
}

await post.save();             // Save updated document back to MongoDB
```

### Step 6 — MongoDB Updates (Database Layer)

Mongoose sends the updated `likes` array to MongoDB. The Post document now looks something like:

```json
{
  "_id": "abc123",
  "author": "user456",
  "imageUrl": "https://res.cloudinary.com/...",
  "likes": ["user456", "user789"],
  "caption": "Sunset view"
}
```

### Step 7 — Response Sent Back (Server → Browser)

The controller sends the updated post back as JSON:

```javascript
const populated = await post.populate('author', 'username profilePic');
res.json(populated);
```

### Step 8 — DOM Update (Frontend Paints the Change)

Back in `feed.js`, the response is received and the DOM is updated **without reloading the page**:

```javascript
const isLiked = post.likes.includes(currentUserId);
btn.className = `like-btn ${isLiked ? 'liked' : ''}`;               // Toggle CSS class
btn.querySelector('svg').setAttribute('fill', isLiked ? 'currentColor' : 'none'); // Fill/unfill heart
btn.querySelector('span').textContent = post.likes.length;           // Update count
```

No page reload. The heart icon fills/unfills and the number changes instantly. This is the power of DOM manipulation with Vanilla JS.

---

## Project File Structure

```
SnapGallery/
├── server.js                     ← Entry point. Boots Express, connects DB.
├── package.json                  ← Dependencies + scripts.
├── .env                          ← Secrets (DB URI, JWT secret, Cloudinary keys).
│
├── src/
│   ├── config/
│   │   └── db.js                 ← MongoDB connection via Mongoose.
│   ├── models/
│   │   ├── User.js               ← User schema (username, email, password, followers, following).
│   │   └── Post.js               ← Post schema (author, imageUrl, caption, likes).
│   ├── controllers/
│   │   ├── authController.js     ← Register, Login, GetMe.
│   │   ├── postController.js     ← Create, Feed, UserPosts, Delete, ToggleLike.
│   │   └── userController.js     ← ToggleFollow, GetUser, SearchUsers.
│   ├── middleware/
│   │   └── auth.js               ← JWT verification middleware.
│   ├── routes/
│   │   ├── auth.js               ← Maps URL paths to auth controllers.
│   │   ├── posts.js              ← Maps URL paths to post controllers. Sets up Multer.
│   │   └── users.js              ← Maps URL paths to user controllers.
│   └── utils/
│       └── cloudinary.js         ← Cloudinary SDK configuration.
│
└── public/
    ├── index.html                ← Login / Register page.
    ├── feed.html                 ← Personalized feed page.
    ├── upload.html               ← Image upload page.
    ├── profile.html              ← User profile page.
    ├── css/
    │   └── style.css             ← Single stylesheet. Dark theme, masonry, responsive.
    └── js/
        ├── auth.js               ← Login/Register form handling + localStorage.
        ├── feed.js               ← Feed loading, likes, search, delete.
        ├── upload.js             ← Drag-drop upload, FormData, Cloudinary pipeline.
        └── profile.js            ← Profile rendering, follow/unfollow, user's posts.
```

---

## How the Server Boots Up

`server.js` is the ignition key. Here's what happens when you run `node server.js`:

```javascript
import 'dotenv/config';       // 1. Load .env file → process.env now has all secrets
import express from 'express'; // 2. Import Express framework
import cors from 'cors';      // 3. Import CORS middleware

const app = express();         // 4. Create an Express application instance

app.use(cors());               // 5. Allow cross-origin requests (frontend ↔ backend)
app.use(express.json());       // 6. Parse incoming JSON bodies automatically

// 7. Mount route groups
app.use('/api/auth', authRoutes);   // All auth endpoints start with /api/auth
app.use('/api/posts', postRoutes);  // All post endpoints start with /api/posts
app.use('/api/users', userRoutes);  // All user endpoints start with /api/users

// 8. Connect to MongoDB FIRST, THEN start listening
connectDB().then(() => {
  app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
  });
});
```

**Why `connectDB().then()`?** You don't want the server to accept requests before the database is ready. If a request arrives before MongoDB is connected, every `User.findById()` call would crash. So we connect first, then open the doors.

---

## ES6 Modules (the `import`/`export` syntax)

Notice this project uses `import` instead of `require`. That's because `package.json` has:

```json
"type": "module"
```

This tells Node.js to treat `.js` files as ES Modules. The difference:

```javascript
// CommonJS (old way)
const express = require('express');
module.exports = router;

// ES Modules (this project)
import express from 'express';
export default router;
```

Both do the same thing. ES Modules is the newer standard that matches how browsers handle imports.

---

## API Endpoints — The Complete Map

| Method   | Endpoint                   | Auth Required? | What It Does                       |
| -------- | -------------------------- | -------------- | ---------------------------------- |
| `POST`   | `/api/auth/register`       | No             | Create a new user account          |
| `POST`   | `/api/auth/login`          | No             | Login and receive a JWT            |
| `GET`    | `/api/auth/me`             | Yes            | Get the currently logged-in user   |
| `GET`    | `/api/posts/feed`          | Yes            | Get personalized feed              |
| `POST`   | `/api/posts`               | Yes            | Upload a new post (with image)     |
| `GET`    | `/api/posts/user/:userId`  | Yes            | Get all posts by a specific user   |
| `DELETE` | `/api/posts/:id`           | Yes            | Delete your own post               |
| `PUT`    | `/api/posts/:id/like`      | Yes            | Toggle like on a post              |
| `GET`    | `/api/users/search?q=...`  | Yes            | Search users by username           |
| `GET`    | `/api/users/:id`           | Yes            | Get a user's profile data          |
| `PUT`    | `/api/users/:id/follow`    | Yes            | Toggle follow/unfollow a user      |

---

## The Golden Rule: Frontend Never Touches the Database

```
                 ┌────────────┐
                 │  Browser   │
                 │  (HTML/JS) │
                 └─────┬──────┘
                       │ fetch() with JWT
                       ▼
                 ┌────────────┐
                 │  Express   │
                 │  (Routes + │
                 │  Middleware)│
                 └─────┬──────┘
                       │ Mongoose queries
                       ▼
                 ┌────────────┐
                 │  MongoDB   │
                 └────────────┘
```

The browser **never** connects to MongoDB directly. It only talks to Express via HTTP. Express is the gatekeeper. This separation is what makes it a "full-stack" application — two independent systems communicating over HTTP.
