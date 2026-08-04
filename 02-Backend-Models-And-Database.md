# 02 — Backend Models & Database

## How MongoDB Differs from SQL

In SQL (MySQL, PostgreSQL), you define rigid tables with rows and columns. In MongoDB, you store **documents** — JSON-like objects inside **collections**. There are no joins in the traditional sense; instead, you store references (IDs) and use Mongoose's `.populate()` to fetch related data.

**Analogy:** SQL is like a spreadsheet with strict columns. MongoDB is like a filing cabinet — each folder (document) can have different papers inside, though in practice you keep a consistent structure using schemas.

---

## The Two Models

SnapGallery has exactly two collections in MongoDB:

1. **`users`** — Stores user accounts, passwords, and the social graph (followers/following).
2. **`posts`** — Stores uploaded images with captions and likes.

---

## User Model — Line by Line

**File:** `src/models/User.js`

```javascript
import mongoose from 'mongoose';
import bcrypt from 'bcryptjs';
```

Imports Mongoose (to define schemas) and bcrypt (to hash passwords).

### The Schema Definition

```javascript
const userSchema = new mongoose.Schema({
```

A schema is a blueprint. It tells MongoDB: "Every User document MUST have these fields, with these types and rules."

```javascript
  username: {
    type: String,
    required: true,     // Can't be empty
    unique: true,       // No two users can have the same username
    trim: true,         // Removes whitespace: "  john  " → "john"
    minlength: 3        // At least 3 characters
  },
```

`unique: true` makes Mongoose create a MongoDB **index** on this field. If someone tries to register with an existing username, MongoDB throws a duplicate key error.

```javascript
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,    // "John@Gmail.COM" → "john@gmail.com"
    trim: true
  },
```

`lowercase: true` is a Mongoose **setter** — it transforms the value before saving. This prevents "john@gmail.com" and "John@Gmail.COM" from being treated as different accounts.

```javascript
  password: {
    type: String,
    required: true,
    minlength: 6
  },
```

The raw password is stored here temporarily. Before saving, it gets hashed (see the `pre('save')` hook below).

```javascript
  profilePic: {
    type: String,
    default: 'https://ui-avatars.com/api/?background=random&size=200&name=User'
  },
```

A default avatar URL. If a user doesn't upload a profile picture, this API generates a colored circle with initials. It's a third-party service — no image is stored locally.

### The Social Graph — followers & following arrays

```javascript
  followers: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }],
  following: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }]
```

This is the **most important architectural decision** in the entire app. Let's break it down.

#### What is `ObjectId`?

Every document in MongoDB gets a unique `_id` field, like `"66a8f3b2c1d2e3f4a5b6c7d8"`. An `ObjectId` is the type of this ID. When you say `type: mongoose.Schema.Types.ObjectId`, you're saying: "This field stores a reference to another document."

#### What is `ref: 'User'`?

`ref` tells Mongoose: "This ObjectId points to a document in the `User` collection." This enables `.populate()` — Mongoose can auto-replace the raw ID with the full user object.

#### How the Social Graph Works

When User A follows User B:

```
User A's document:
  following: [ ..., "userB_id" ]     ← A's following list gains B's ID

User B's document:
  followers: [ ..., "userA_id" ]     ← B's followers list gains A's ID
```

**Two-way bookkeeping.** Both documents get updated. This is crucial because:
- When you visit User A's profile, you need their `following` count.
- When you visit User B's profile, you need their `followers` count.
- When User A opens their feed, you query posts from all users in A's `following` array.

**Analogy:** Think of Instagram. When you tap "Follow" on someone's profile, two things happen simultaneously: your "Following" count goes up by 1, and their "Followers" count goes up by 1. That's exactly what these two arrays model.

### The Timestamps Option

```javascript
}, { timestamps: true });
```

This single option makes Mongoose auto-add two fields to every document:

```json
{
  "createdAt": "2026-07-06T14:30:00.000Z",
  "updatedAt": "2026-07-06T15:45:00.000Z"
}
```

You never set these manually. Mongoose handles it.

### Pre-Save Hook — Password Hashing

```javascript
userSchema.pre('save', async function () {
  if (!this.isModified('password')) return;
  this.password = await bcrypt.hash(this.password, 10);
});
```

**What is a "hook"?** A hook is code that runs automatically before or after a specific event. `pre('save')` runs before every `.save()` call.

**Line by line:**

1. `this.isModified('password')` — Checks if the password field was actually changed. If the user only updated their username, we don't want to re-hash the already-hashed password.
2. `bcrypt.hash(this.password, 10)` — Takes the plain password and hashes it with 10 rounds of salting.

**What is hashing?**

Plain password: `"mypassword123"`
After bcrypt:   `"$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"`

You **cannot** reverse a hash. You can only compare: "Does hashing `"mypassword123"` produce the same hash?" That's what `comparePassword` does.

**Why 10 rounds?** Each "round" makes the hashing slower. 10 rounds ≈ ~10ms per hash. That's nothing for a legitimate login, but if an attacker steals the database and tries to brute-force millions of passwords, each attempt takes 10ms instead of microseconds. It's a deliberate speed bump for attackers.

### Instance Method — comparePassword

```javascript
userSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};
```

`userSchema.methods` adds a method to every User **instance** (document). So you can call:

```javascript
const user = await User.findOne({ email: "john@gmail.com" });
const isMatch = await user.comparePassword("mypassword123"); // true or false
```

`bcrypt.compare` takes the candidate password, hashes it, and checks if it matches the stored hash.

### Instance Method — toJSON

```javascript
userSchema.methods.toJSON = function () {
  const obj = this.toObject();
  delete obj.password;
  return obj;
};
```

**This is a security measure.** Whenever you send a user object as JSON in a response (`res.json(user)`), Express internally calls `JSON.stringify(user)`, which calls `toJSON()`. This override removes the password hash before sending the response.

Without this, every API response containing a user would leak the hashed password. Even though it's hashed, it's still sensitive data.

### Exporting the Model

```javascript
export default mongoose.model('User', userSchema);
```

`mongoose.model('User', userSchema)` does two things:
1. Creates a model class called `User` with all the CRUD methods (`.find()`, `.create()`, `.findById()`, etc.).
2. Maps it to a MongoDB collection called `users` (Mongoose auto-pluralizes and lowercases).

---

## Post Model — Line by Line

**File:** `src/models/Post.js`

```javascript
import mongoose from 'mongoose';

const postSchema = new mongoose.Schema({
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
```

`author` stores the `_id` of the user who created this post. `ref: 'User'` means you can call `.populate('author')` to replace the ID with the full User document.

```javascript
  imageUrl: {
    type: String,
    required: true
  },
```

The Cloudinary URL of the uploaded image. Example: `"https://res.cloudinary.com/mycloud/image/upload/v123/SnapGallery/abc.jpg"`. The actual image is stored on Cloudinary's servers, not in MongoDB.

```javascript
  caption: {
    type: String,
    maxlength: 500,
    default: ''
  },
```

Optional caption. Max 500 characters. If not provided, defaults to an empty string.

```javascript
  likes: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }]
}, { timestamps: true });
```

The `likes` array stores the `_id` of every user who liked this post. To check if a user has liked a post:

```javascript
post.likes.includes(userId)  // true = already liked
```

To count likes:

```javascript
post.likes.length  // number of likes
```

This is a simpler alternative to creating a separate `Likes` collection with join tables, which is what SQL databases typically require.

---

## How `.populate()` Works

When you query a post from MongoDB, the raw data looks like this:

```json
{
  "_id": "post123",
  "author": "user456",           // ← Just a raw ObjectId string
  "imageUrl": "https://...",
  "likes": ["user456", "user789"]
}
```

After calling `.populate('author', 'username profilePic')`:

```json
{
  "_id": "post123",
  "author": {                     // ← Replaced with the actual user document
    "_id": "user456",
    "username": "john",
    "profilePic": "https://..."
  },
  "imageUrl": "https://...",
  "likes": ["user456", "user789"]
}
```

The second argument `'username profilePic'` is a **field selection** — it tells Mongoose to only include `username` and `profilePic` from the referenced User document. This saves bandwidth; the frontend doesn't need the user's email or password hash just to display a name.

**Analogy:** It's like a library card that has a book's ISBN number. `populate()` is the librarian going to the shelf, finding the book by ISBN, and handing you the actual book instead of just the number.

---

## The `$in` Operator — How the Feed is Built

This is the most important query in the entire application. In `postController.js`:

```javascript
const following = [...user.following, user._id];
const posts = await Post.find({ author: { $in: following } })
  .sort({ createdAt: -1 })
  .limit(50);
```

### What `$in` Does

`{ author: { $in: following } }` translates to:

> "Find all posts where the `author` field matches **any** of the IDs in the `following` array."

If `following = ["userA_id", "userB_id", "myOwnId"]`, the query finds all posts authored by User A, User B, or yourself.

### Why include `user._id`?

```javascript
const following = [...user.following, user._id];
```

The spread operator `...` copies all IDs from the user's `following` array, and then we add `user._id` at the end. This ensures your own posts also appear in your feed.

### The Sorting

```javascript
.sort({ createdAt: -1 })
```

`-1` means descending order. Newest posts first. If you used `1`, it would be oldest first.

### The Limit

```javascript
.limit(50)
```

Don't return the entire history. Just the 50 most recent posts. This prevents massive payloads and keeps the response fast.

**Analogy:** Imagine you follow 5 YouTube channels. Your YouTube home page doesn't show videos from every channel on the platform — it only shows videos from your 5 subscribed channels, sorted by newest first. That's exactly what `$in` does here. It's a personalized filter.

### In SQL, This Would Be a JOIN

The equivalent SQL query would be something like:

```sql
SELECT * FROM posts
WHERE author_id IN (SELECT following_id FROM user_following WHERE user_id = ?)
   OR author_id = ?
ORDER BY created_at DESC
LIMIT 50;
```

MongoDB's `$in` achieves the same result without explicit joins. The `following` array is already stored directly on the user document, so you don't need a separate table.

---

## Atomic Updates & Race Conditions — The Follow/Unfollow Problem

In `userController.js`, the `toggleFollow` function updates **two** documents:

```javascript
currentUser.following.push(targetUser._id);
targetUser.followers.push(currentUser._id);

await currentUser.save();
await targetUser.save();
```

### What's a Race Condition?

Imagine two things happen at the exact same time:
1. User A follows User B
2. User A follows User C

Both requests read User A's `following` array simultaneously. Let's say it's currently `["X", "Y"]`.

- Request 1 reads `["X", "Y"]`, adds "B" → saves `["X", "Y", "B"]`
- Request 2 reads `["X", "Y"]` (same stale data!), adds "C" → saves `["X", "Y", "C"]`

**Result:** User B's follow was lost! The final array is `["X", "Y", "C"]` instead of `["X", "Y", "B", "C"]`.

### The Current Implementation's Tradeoff

This project uses `.push()` + `.save()`, which is the standard Mongoose approach. For a small to medium app, this works fine because the probability of two follow requests for the same user arriving at the exact same millisecond is extremely low.

### The More Robust Alternative (For High Traffic)

For a production app with millions of users, you'd use MongoDB's **atomic operators** like `$addToSet` and `$pull`:

```javascript
// Atomic approach — happens in a single database operation
await User.findByIdAndUpdate(currentUser._id, {
  $addToSet: { following: targetUser._id }  // Only adds if not already present
});
await User.findByIdAndUpdate(targetUser._id, {
  $addToSet: { followers: currentUser._id }
});
```

`$addToSet` is atomic — it reads and writes in a single database operation, so there's no window for another request to interfere. It also prevents duplicates automatically.

**Analogy:** Imagine two people trying to add their name to a whiteboard list at the same time. With the current approach (read → modify → write), one person might overwrite the other's addition. With `$addToSet`, it's like having a strict receptionist who adds names one at a time — no overlap possible.

---

## Document Sample: What a Real User Document Looks Like

```json
{
  "_id": "66a8f3b2c1d2e3f4a5b6c7d8",
  "username": "arpit",
  "email": "arpit@gmail.com",
  "password": "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy",
  "profilePic": "https://ui-avatars.com/api/?background=random&size=200&name=User",
  "followers": [
    "66a8f4a1b2c3d4e5f6a7b8c9",
    "66a8f5b2c3d4e5f6a7b8c9d0"
  ],
  "following": [
    "66a8f4a1b2c3d4e5f6a7b8c9"
  ],
  "createdAt": "2026-07-06T14:30:00.000Z",
  "updatedAt": "2026-07-06T15:45:00.000Z",
  "__v": 0
}
```

## Document Sample: What a Real Post Document Looks Like

```json
{
  "_id": "66b1a2b3c4d5e6f7a8b9c0d1",
  "author": "66a8f3b2c1d2e3f4a5b6c7d8",
  "imageUrl": "https://res.cloudinary.com/mycloud/image/upload/v123/SnapGallery/sunset.jpg",
  "caption": "Golden hour vibes 🌅",
  "likes": [
    "66a8f4a1b2c3d4e5f6a7b8c9",
    "66a8f5b2c3d4e5f6a7b8c9d0"
  ],
  "createdAt": "2026-07-07T10:15:00.000Z",
  "updatedAt": "2026-07-07T11:30:00.000Z",
  "__v": 0
}
```

`__v` is Mongoose's internal version key. It tracks the number of times the document has been saved. You can ignore it.
