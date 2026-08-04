# 03 — Backend Routes & Controllers

## How Express Routing Works

Express routing is a two-layer system in this project:

1. **`server.js`** — Mounts route groups under prefixes.
2. **`src/routes/*.js`** — Defines specific HTTP method + path combinations within each group.
3. **`src/controllers/*.js`** — The actual handler functions that run when a route matches.

```
server.js                    Routes File                  Controller
─────────                    ───────────                  ──────────
/api/auth  ──────────►  auth.js                     authController.js
                           POST /register  ──────►  register()
                           POST /login     ──────►  login()
                           GET  /me        ──────►  getMe()

/api/posts ──────────►  posts.js                    postController.js
                           GET  /feed      ──────►  getFeed()
                           POST /          ──────►  createPost()
                           GET  /user/:id  ──────►  getUserPosts()
                           DELETE /:id     ──────►  deletePost()
                           PUT  /:id/like  ──────►  toggleLike()

/api/users ──────────►  users.js                    userController.js
                           GET  /search    ──────►  searchUsers()
                           GET  /:id       ──────►  getUser()
                           PUT  /:id/follow──────►  toggleFollow()
```

---

## Auth Routes — `src/routes/auth.js`

```javascript
import { Router } from 'express';
import { register, login, getMe } from '../controllers/authController.js';
import auth from '../middleware/auth.js';

const router = Router();

router.post('/register', register);   // Public — no auth needed
router.post('/login', login);         // Public — no auth needed
router.get('/me', auth, getMe);       // Protected — auth middleware runs first

export default router;
```

Key point: `register` and `login` are **public** routes. You can't require a JWT to login — you don't have one yet! But `/me` requires the `auth` middleware because it returns the **currently logged-in** user's data, so it needs to know who "me" is.

When you write `router.get('/me', auth, getMe)`, Express executes them left to right:
1. First `auth` middleware runs (verifies JWT, attaches `req.user`)
2. If `auth` calls `next()`, then `getMe` runs
3. If `auth` sends a 401 response, `getMe` never executes

---

## Auth Controller — `src/controllers/authController.js`

### The Token Generator

```javascript
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '7d' });
};
```

`jwt.sign()` creates a JWT token. Three arguments:

1. **Payload** — `{ id }` — The data embedded inside the token. Here, just the user's MongoDB `_id`.
2. **Secret** — `process.env.JWT_SECRET` — A secret string that only the server knows. Used to sign the token. If anyone tampers with the payload, the signature won't match.
3. **Options** — `{ expiresIn: '7d' }` — The token expires in 7 days. After that, the user must log in again.

### `register()` — Creating a New Account

```javascript
export const register = async (req, res) => {
  try {
    const { username, email, password } = req.body;
```

Destructures the request body. When the frontend sends `{ username: "arpit", email: "...", password: "..." }`, this line extracts those three values.

```javascript
    if (!username || !email || !password) {
      return res.status(400).json({ message: 'All fields are required' });
    }
```

Server-side validation. Even if the frontend validates inputs, **never trust the client**. Someone could bypass the frontend and send a raw HTTP request with missing fields.

```javascript
    const existingUser = await User.findOne({ $or: [{ email }, { username }] });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
```

`$or` is a MongoDB operator. It says: "Find a user where EITHER the email matches OR the username matches." This prevents duplicate accounts.

```javascript
    const user = await User.create({ username, email, password });
```

`User.create()` does two things:
1. Creates a new User document with the provided fields.
2. Calls `.save()` internally, which triggers the `pre('save')` hook and **hashes the password** with bcrypt.

So by the time the document reaches MongoDB, the password is already hashed. The plain text password is never stored.

```javascript
    const token = generateToken(user._id);
    res.status(201).json({ token, user });
```

After creating the user, immediately generate a JWT and send it back. This way the user doesn't have to login separately after registering — they're auto-logged-in.

`res.status(201)` — HTTP 201 means "Created." It tells the client: "Your resource was successfully created."

Note: When `user` is serialized to JSON, the `toJSON()` method on the schema runs, which **strips the password** from the response. So the client receives `{ token, user: { _id, username, email, ... } }` — no password field.

### `login()` — Authenticating an Existing User

```javascript
export const login = async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ message: 'All fields are required' });
    }

    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
```

Find the user by email. If no user exists with that email, return a generic "Invalid credentials" message. We deliberately don't say "User not found" vs "Wrong password" — that would let an attacker know which emails are registered.

```javascript
    const isMatch = await user.comparePassword(password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
```

`user.comparePassword()` is the instance method defined on the User schema. It bcrypt-compares the plain text password with the stored hash.

```javascript
    const token = generateToken(user._id);
    res.json({ token, user });
```

On success, generate a fresh JWT and send it. The frontend stores this token in `localStorage`.

### `getMe()` — Who Am I?

```javascript
export const getMe = async (req, res) => {
  try {
    const user = await User.findById(req.user._id)
      .populate('followers', 'username profilePic')
      .populate('following', 'username profilePic');
    res.json(user);
```

This route is protected by the `auth` middleware, so `req.user` is already set. It fetches the full user document with populated followers and following arrays.

`.populate('followers', 'username profilePic')` replaces the raw ObjectId array with actual user objects, but only includes `username` and `profilePic` — not email, password, etc.

---

## Post Routes — `src/routes/posts.js`

```javascript
import { Router } from 'express';
import multer from 'multer';
import { createPost, getFeed, getUserPosts, deletePost, toggleLike } from '../controllers/postController.js';
import auth from '../middleware/auth.js';

const router = Router();
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 }
});
```

**Multer setup:** We'll cover this in depth in Document 04, but the key point is:
- `multer.memoryStorage()` — Stores the uploaded file in RAM (as a Buffer), not on the hard drive.
- `limits: { fileSize: 5 * 1024 * 1024 }` — Maximum 5MB file size. `5 * 1024 * 1024` = 5,242,880 bytes = 5MB.

```javascript
router.use(auth);
```

**Every single route in this file requires authentication.** `router.use(auth)` applies the auth middleware to ALL routes registered on this router. You can't view the feed, create posts, or like anything without a valid JWT.

```javascript
router.get('/feed', getFeed);
router.post('/', upload.single('image'), createPost);
router.get('/user/:userId', getUserPosts);
router.delete('/:id', deletePost);
router.put('/:id/like', toggleLike);
```

For the `POST /` route, there are **three** middlewares in order:
1. `auth` (from `router.use(auth)` above) — Verifies JWT
2. `upload.single('image')` — Parses the multipart form data, extracts the file named `'image'`
3. `createPost` — Handles the business logic

`upload.single('image')` means: "I expect exactly one file, and its field name in the form data is `image`." After this middleware runs, `req.file` contains the uploaded file's buffer, mimetype, size, etc.

---

## Post Controller — `src/controllers/postController.js`

### `uploadToCloudinary()` — The Streaming Helper

```javascript
import Post from '../models/Post.js';
import cloudinary from '../utils/cloudinary.js';

const uploadToCloudinary = (buffer) => {
  return new Promise((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder: 'SnapGallery' },
      (error, result) => {
        if (error) reject(error);
        else resolve(result);
      }
    );
    stream.end(buffer);
  });
};
```

This function wraps Cloudinary's streaming upload in a Promise. Detailed explanation in Document 04.

### `createPost()` — Uploading a New Post

```javascript
export const createPost = async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ message: 'Image is required' });
    }
```

`req.file` is set by Multer. If no file was uploaded, reject the request.

```javascript
    const result = await uploadToCloudinary(req.file.buffer);
```

`req.file.buffer` is the raw binary data of the image (stored in memory by Multer). This is sent to Cloudinary, which returns a result object containing the URL.

```javascript
    const post = await Post.create({
      author: req.user._id,
      imageUrl: result.secure_url,
      caption: req.body.caption || ''
    });
```

Creates a new Post document. `result.secure_url` is the HTTPS URL where Cloudinary hosts the image. `req.body.caption` comes from the form data (sent alongside the image).

```javascript
    const populated = await post.populate('author', 'username profilePic');
    res.status(201).json(populated);
```

Before sending the response, populate the `author` field so the frontend gets the username and profile picture, not just a raw ID.

### `getFeed()` — The Personalized Feed

```javascript
export const getFeed = async (req, res) => {
  try {
    const user = req.user;
    const following = [...user.following, user._id];
```

Creates an array of IDs: everyone the current user follows + their own ID. The `...` (spread operator) copies the `following` array elements into a new array, and `user._id` is appended at the end.

```javascript
    const posts = await Post.find({ author: { $in: following } })
      .sort({ createdAt: -1 })
      .populate('author', 'username profilePic')
      .limit(50);

    res.json(posts);
```

This query chain:
1. **`find({ author: { $in: following } })`** — Get posts where author is in the following list
2. **`.sort({ createdAt: -1 })`** — Newest first
3. **`.populate('author', 'username profilePic')`** — Replace author ID with user object
4. **`.limit(50)`** — Cap at 50 posts

The response is a JSON array of post objects, each with a populated author.

### `getUserPosts()` — A Specific User's Posts

```javascript
export const getUserPosts = async (req, res) => {
  try {
    const posts = await Post.find({ author: req.params.userId })
      .sort({ createdAt: -1 })
      .populate('author', 'username profilePic');

    res.json(posts);
```

`req.params.userId` comes from the URL pattern `/user/:userId`. If the URL is `/api/posts/user/abc123`, then `req.params.userId` is `"abc123"`. Simple filter: find all posts where `author === abc123`.

### `deletePost()` — Removing a Post

```javascript
export const deletePost = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }
```

Find the post by ID. If it doesn't exist, return 404.

```javascript
    if (post.author.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }
```

**Authorization check.** Even though the user is authenticated (they have a valid JWT), they can only delete **their own** posts. `post.author` is an ObjectId object, `req.user._id` is also an ObjectId object. You can't compare objects with `===` in JavaScript (it checks reference, not value), so you convert both to strings first.

- **401 Unauthorized** = "I don't know who you are" (no/invalid token)
- **403 Forbidden** = "I know who you are, but you can't do this" (not the owner)

```javascript
    await post.deleteOne();
    res.json({ message: 'Post deleted' });
```

Delete the document from MongoDB and confirm success.

### `toggleLike()` — Like/Unlike a Post

```javascript
export const toggleLike = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }

    const userId = req.user._id;
    const index = post.likes.indexOf(userId);
```

`indexOf` returns the position of `userId` in the `likes` array. If the user hasn't liked the post, it returns `-1`.

```javascript
    if (index === -1) {
      post.likes.push(userId);     // Add like
    } else {
      post.likes.splice(index, 1); // Remove like
    }
```

**Toggle logic:**
- If `index === -1` → User hasn't liked it → Add their ID to the array
- If `index >= 0` → User already liked it → Remove their ID using `splice(index, 1)`, which removes 1 element at position `index`

This is why it's called "toggle" — one endpoint handles both liking AND unliking. The frontend doesn't need separate "like" and "unlike" buttons or API calls.

```javascript
    await post.save();
    const populated = await post.populate('author', 'username profilePic');
    res.json(populated);
```

Save the updated likes array, populate the author, and return the full post. The frontend uses the returned post to update the like count and heart icon state.

---

## User Routes — `src/routes/users.js`

```javascript
const router = Router();
router.use(auth);                          // All routes require JWT

router.get('/search', searchUsers);
router.get('/:id', getUser);
router.put('/:id/follow', toggleFollow);
```

**Important ordering:** `/search` is defined BEFORE `/:id`. If it were reversed, Express would interpret the URL `/api/users/search` as `/:id` with `id = "search"`. Express matches routes top-to-bottom and uses the first match, so specific routes must come before parameterized ones.

---

## User Controller — `src/controllers/userController.js`

### `toggleFollow()` — Follow/Unfollow

```javascript
export const toggleFollow = async (req, res) => {
  try {
    if (req.params.id === req.user._id.toString()) {
      return res.status(400).json({ message: 'Cannot follow yourself' });
    }
```

Self-follow guard. You shouldn't be able to follow yourself.

```javascript
    const targetUser = await User.findById(req.params.id);
    if (!targetUser) {
      return res.status(404).json({ message: 'User not found' });
    }

    const currentUser = await User.findById(req.user._id);
    const isFollowing = currentUser.following.includes(targetUser._id);
```

Load both users. Check if the current user already follows the target.

```javascript
    if (isFollowing) {
      currentUser.following.pull(targetUser._id);   // Remove from following
      targetUser.followers.pull(currentUser._id);   // Remove from followers
    } else {
      currentUser.following.push(targetUser._id);   // Add to following
      targetUser.followers.push(currentUser._id);   // Add to followers
    }
```

**Two-way update.** When you follow someone, two things change:
1. Your `following` array grows
2. Their `followers` array grows

`.pull()` is a Mongoose array method (similar to MongoDB's `$pull`). It removes the matching element.

```javascript
    await currentUser.save();
    await targetUser.save();

    res.json({
      isFollowing: !isFollowing,
      followersCount: targetUser.followers.length,
      followingCount: targetUser.following.length
    });
```

Save both documents and return the new state. The frontend uses `isFollowing` to toggle the button text between "Follow" and "Unfollow", and the counts to update the profile stats.

### `getUser()` — Fetch a User Profile

```javascript
export const getUser = async (req, res) => {
  try {
    const user = await User.findById(req.params.id)
      .select('-password')
      .populate('followers', 'username profilePic')
      .populate('following', 'username profilePic');
```

`.select('-password')` means "return all fields EXCEPT password." The minus sign is an exclusion operator. This is an extra safety layer on top of the `toJSON()` method.

Both `followers` and `following` arrays are populated so the frontend can display user cards with names and pictures (not just raw IDs).

### `searchUsers()` — Find Users by Username

```javascript
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
```

`req.query` contains URL query parameters. For `/api/users/search?q=arpit`, `req.query.q` is `"arpit"`.

**`$regex`** performs a regular expression search. `$options: 'i'` makes it case-insensitive. So searching for `"Ar"` would match "arpit", "Arjun", "ARCHER", etc.

`.select('username profilePic followers following')` explicitly lists which fields to return. This is a **whitelist approach** (opposite of the `-password` blacklist). It returns only what the search dropdown needs.

`.limit(20)` — Don't return more than 20 results. Search dropdowns should be small.
