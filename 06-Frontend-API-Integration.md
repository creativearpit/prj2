# 06 — Frontend API Integration

## The Core Pattern: How Every API Call Works

Every frontend JS file follows the same pattern for talking to the backend:

```javascript
const BACKEND_URL = 'https://snapgallery-jkz1.onrender.com';
const token = localStorage.getItem('token');

function getHeaders() {
  return {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  };
}
```

**`BACKEND_URL`** — The deployed backend URL. All API calls are absolute URLs to this server. In development, this would be `'http://localhost:3000'`.

**`localStorage.getItem('token')`** — Retrieves the JWT that was stored during login. This token is attached to every request.

**`getHeaders()`** — Returns the two headers every protected API needs:
1. `Authorization: Bearer <token>` — The JWT for authentication
2. `Content-Type: application/json` — Tells Express to parse the body as JSON

---

## File-by-File Breakdown

### `auth.js` — Login & Registration

#### Auto-Redirect for Logged-In Users

```javascript
if (localStorage.getItem('token')) {
  window.location.href = '/feed.html';
}
```

If a JWT already exists in localStorage, the user is already logged in. Don't show them the login page — send them to the feed immediately. This runs before anything else in the file.

#### Login Handler

```javascript
loginForm.addEventListener('submit', async (e) => {
  e.preventDefault();
```

`e.preventDefault()` stops the form from doing its default behavior — submitting via a full page reload. We want to handle the submission with JavaScript (via `fetch`), not with a traditional form POST.

```javascript
  const btn = document.getElementById('loginBtn');
  btn.disabled = true;
  btn.textContent = 'Signing in...';
```

**UX detail:** Disable the button and change its text to prevent double-submission. If the user clicks "Sign In" twice quickly, without this guard, two API calls would fire.

```javascript
  try {
    const res = await fetch(`${BACKEND_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: document.getElementById('loginEmail').value,
        password: document.getElementById('loginPassword').value
      })
    });
```

**The fetch call, dissected:**

- `method: 'POST'` — Login is a POST request (sending data to the server).
- `headers: { 'Content-Type': 'application/json' }` — No `Authorization` header here because the user doesn't have a token yet. This IS the request to get a token.
- `body: JSON.stringify({...})` — `fetch` requires the body to be a string. `JSON.stringify` converts the JavaScript object to a JSON string. The values are read directly from the input fields.

```javascript
    const data = await res.json();
    if (!res.ok) throw new Error(data.message);
```

`res.json()` parses the response body from JSON to a JavaScript object.

`res.ok` is a shorthand for "is the HTTP status code between 200-299?" If the status is 400 (bad request) or 500 (server error), `res.ok` is `false`, and we throw an error with the server's message.

```javascript
    localStorage.setItem('token', data.token);
    localStorage.setItem('userId', data.user._id);
    localStorage.setItem('username', data.user.username);
    window.location.href = '/feed.html';
```

**On success, store three things:**
1. `token` — The JWT. Used for all subsequent API calls.
2. `userId` — The user's MongoDB ID. Used to check "Is this my post?" and "Am I viewing my own profile?"
3. `username` — Used for display purposes.

Then redirect to the feed page.

```javascript
  } catch (err) {
    showError(err.message);
  } finally {
    btn.disabled = false;
    btn.textContent = 'Sign In';
  }
```

`catch` displays the error message (e.g., "Invalid credentials").
`finally` always runs — whether the request succeeded or failed — and re-enables the button.

#### Register Handler — Nearly Identical

```javascript
registerForm.addEventListener('submit', async (e) => {
  // Same pattern: preventDefault, disable button, fetch POST, store token, redirect
  const res = await fetch(`${BACKEND_URL}/api/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: document.getElementById('regUsername').value,
      email: document.getElementById('regEmail').value,
      password: document.getElementById('regPassword').value
    })
  });
  // ... same success/error handling
});
```

The only difference is the endpoint (`/register` vs `/login`) and the extra `username` field.

---

### `feed.js` — Feed, Likes, Search, Delete

#### Loading the Feed

```javascript
async function loadFeed() {
  const loading = document.getElementById('loadingState');
  const empty = document.getElementById('emptyState');
  const grid = document.getElementById('feedGrid');

  try {
    const res = await fetch(`${BACKEND_URL}/api/posts/feed`, {
      headers: getHeaders()
    });
```

A GET request to `/api/posts/feed`. The JWT is attached via `getHeaders()`. The backend uses this token to identify the user, look up their `following` array, and return a personalized feed.

```javascript
    if (res.status === 401) {
      localStorage.clear();
      window.location.href = '/';
      return;
    }
```

**Token expiry handling.** If the server returns 401, the token is invalid or expired. Clear all stored data and redirect to login. This is the "auto-logout" mechanism.

```javascript
    const posts = await res.json();
    loading.style.display = 'none';
```

Parse the response (an array of post objects) and hide the loading spinner.

```javascript
    if (posts.length === 0) {
      empty.style.display = 'block';
      return;
    }
```

If the feed is empty (user doesn't follow anyone and has no posts), show the "Your feed is empty" message.

```javascript
    grid.innerHTML = posts.map(post => createPinCard(post)).join('');
```

Map each post to an HTML card string, join them all, and inject into the grid. This is where the masonry grid gets populated.

#### Creating a Pin Card — `createPinCard()`

```javascript
function createPinCard(post) {
  const isLiked = post.likes.includes(currentUserId);
  const isOwner = post.author._id === currentUserId;
```

Two checks using the stored `currentUserId`:
- `isLiked` — Is the current user's ID in the post's `likes` array? Controls the heart icon state.
- `isOwner` — Is the current user the author of this post? Controls whether the delete button appears.

```javascript
  return `
    <div class="pin-card fade-in">
      <img src="${post.imageUrl}" alt="${post.caption || 'Pin'}" loading="lazy">
      <div class="pin-info">
        ${post.caption ? `<p class="pin-caption">${escapeHtml(post.caption)}</p>` : ''}
```

Conditional rendering with ternary operator. If there's a caption, render the `<p>` tag. If not, render nothing (empty string `''`). `escapeHtml()` prevents XSS attacks from malicious captions.

```javascript
        <div class="pin-author" onclick="viewProfile('${post.author._id}')">
          <img src="${post.author.profilePic}" alt="${post.author.username}">
          <span>${escapeHtml(post.author.username)}</span>
        </div>
```

Clicking the author's name/avatar navigates to their profile page. The `onclick` uses `viewProfile()`:

```javascript
function viewProfile(userId) {
  window.location.href = `/profile.html?id=${userId}`;
}
```

```javascript
        <button class="like-btn ${isLiked ? 'liked' : ''}"
                onclick="toggleLike('${post._id}', this)">
          <svg ... fill="${isLiked ? 'currentColor' : 'none'}" ...>
            <!-- heart icon -->
          </svg>
          <span>${post.likes.length}</span>
        </button>
```

The like button:
- CSS class toggles between `like-btn` (gray heart) and `like-btn liked` (red heart)
- SVG `fill` toggles between `none` (outline) and `currentColor` (filled)
- The `<span>` shows the like count
- `this` in `onclick` passes the button element itself for DOM manipulation later

```javascript
        ${isOwner ? `
          <button class="delete-btn" onclick="deletePost('${post._id}', this)">
            <!-- trash icon -->
          </button>
        ` : ''}
```

The delete button only renders if `isOwner` is true. Non-owners don't see it.

#### Toggle Like — Optimistic UI

```javascript
async function toggleLike(postId, btn) {
  try {
    const res = await fetch(`${BACKEND_URL}/api/posts/${postId}/like`, {
      method: 'PUT',
      headers: getHeaders()
    });
    const post = await res.json();
    const isLiked = post.likes.includes(currentUserId);

    btn.className = `like-btn ${isLiked ? 'liked' : ''}`;
    const svg = btn.querySelector('svg');
    svg.setAttribute('fill', isLiked ? 'currentColor' : 'none');
    btn.querySelector('span').textContent = post.likes.length;
  } catch {
    showToast('Failed to update like', 'error');
  }
}
```

**The flow:**
1. Send PUT request to toggle the like on the server
2. Server returns the updated post (with the new likes array)
3. Check if current user is now in the likes array
4. Update the button's CSS class (for color)
5. Update the SVG fill (filled vs outline heart)
6. Update the count text

**Note:** This is NOT "optimistic UI" — it waits for the server response before updating. True optimistic UI would update the button immediately and revert if the request fails. This approach is simpler and guarantees the UI matches the server state.

#### Delete Post

```javascript
async function deletePost(postId, btn) {
  if (!confirm('Delete this post?')) return;
```

`confirm()` shows a native browser dialog. If the user clicks "Cancel", the function returns early.

```javascript
  try {
    await fetch(`${BACKEND_URL}/api/posts/${postId}`, {
      method: 'DELETE',
      headers: getHeaders()
    });
    btn.closest('.pin-card').remove();
    showToast('Post deleted');
```

`btn.closest('.pin-card')` traverses **up** the DOM tree from the delete button and finds the nearest ancestor with class `.pin-card`. `.remove()` removes the entire card from the DOM.

This is cleaner than finding the card by ID. `closest()` walks up the parent chain, so it doesn't matter how deeply nested the button is inside the card.

#### User Search — Debounced Input

```javascript
let searchTimeout;
const searchInput = document.getElementById('searchInput');
const searchResults = document.getElementById('searchResults');

searchInput.addEventListener('input', (e) => {
  clearTimeout(searchTimeout);
  const q = e.target.value.trim();
  if (q.length < 2) {
    searchResults.classList.remove('active');
    return;
  }

  searchTimeout = setTimeout(async () => {
    // ... fetch search results
  }, 300);
});
```

**What is debouncing?**

Without debouncing: User types "arpit" → 5 API calls fire ('a', 'ar', 'arp', 'arpi', 'arpit').

With debouncing: User types "arpit" → Timer resets on every keystroke → Only ONE API call fires 300ms after the user stops typing.

**How it works:**
1. `searchTimeout` stores a timer ID
2. On every keystroke, `clearTimeout(searchTimeout)` cancels the previous timer
3. `setTimeout(..., 300)` starts a new 300ms timer
4. Only when the user pauses typing for 300ms does the callback execute and make the API call

`q.length < 2` — Don't search for single characters. It would return too many results and waste a request.

```javascript
    try {
      const res = await fetch(
        `${BACKEND_URL}/api/users/search?q=${encodeURIComponent(q)}`,
        { headers: getHeaders() }
      );
      const users = await res.json();
```

`encodeURIComponent(q)` makes the search query URL-safe. If the user types `"a b"`, it becomes `"a%20b"` in the URL. Without encoding, the space would break the URL.

```javascript
      if (users.length === 0) {
        searchResults.innerHTML = '<div ...>No users found</div>';
      } else {
        searchResults.innerHTML = users.map(u => `
          <div class="search-result-item" onclick="viewProfile('${u._id}')">
            <img src="${u.profilePic}" alt="${u.username}">
            <span>${escapeHtml(u.username)}</span>
          </div>
        `).join('');
      }
      searchResults.classList.add('active');
```

`classList.add('active')` makes the dropdown visible (CSS `.search-results.active { display: block }`).

#### Click-Outside to Close Search

```javascript
document.addEventListener('click', (e) => {
  if (!e.target.closest('.search-container')) {
    searchResults.classList.remove('active');
  }
});
```

This listens for clicks **anywhere** on the page. If the click target is NOT inside the `.search-container`, close the search dropdown. `closest()` checks if the clicked element (or any of its ancestors) has the class `search-container`.

#### Logout

```javascript
document.getElementById('logoutBtn').addEventListener('click', () => {
  localStorage.clear();
  window.location.href = '/';
});
```

Clear all stored data (token, userId, username) and redirect to the login page. There's no API call — since JWTs are stateless, there's nothing to invalidate on the server. The token will still be technically valid until it expires, but without it stored in localStorage, the user can't use it.

#### Bootstrapping

```javascript
loadFeed();   // Last line of the file
```

The `loadFeed()` function is called at the end of the file. Since the script runs when the page loads (because it's at the bottom of the HTML `<body>`), this immediately starts fetching and rendering the feed.

---

### `upload.js` — Image Upload with Drag & Drop

#### File Selection — Three Ways

**Way 1: Click the upload zone**

```javascript
uploadZone.addEventListener('click', () => imageInput.click());
```

Programmatically triggers the hidden `<input type="file">`. The browser's native file picker opens.

**Way 2: Drag and drop**

```javascript
uploadZone.addEventListener('dragover', (e) => {
  e.preventDefault();
  uploadZone.style.borderColor = 'var(--accent)';
  uploadZone.style.background = 'var(--accent-soft)';
});

uploadZone.addEventListener('dragleave', () => {
  uploadZone.style.borderColor = '';
  uploadZone.style.background = '';
});

uploadZone.addEventListener('drop', (e) => {
  e.preventDefault();
  const file = e.dataTransfer.files[0];
  if (file && file.type.startsWith('image/')) {
    handleFile(file);
  } else {
    showError('Please select a valid image file');
  }
});
```

`dragover` — fires continuously while a file hovers over the zone. `e.preventDefault()` is required — without it, the browser would navigate to the dropped file. The visual feedback (red border + tint) tells the user "you can drop here."

`dragleave` — fires when the file leaves the zone. Resets the visual feedback.

`drop` — fires when the file is dropped. `e.dataTransfer.files[0]` is the first dropped file. `file.type.startsWith('image/')` validates it's an image (not a PDF, text file, etc.).

**Way 3: File input change**

```javascript
imageInput.addEventListener('change', (e) => {
  if (e.target.files[0]) handleFile(e.target.files[0]);
});
```

This fires when the user selects a file through the native file picker (triggered by Way 1).

#### Processing the Selected File

```javascript
function handleFile(file) {
  if (file.size > 5 * 1024 * 1024) {
    showError('Image must be under 5MB');
    return;
  }

  selectedFile = file;
  uploadBtn.disabled = false;
```

Client-side validation: reject files over 5MB before even trying to upload. Store the file reference in `selectedFile` and enable the upload button.

```javascript
  const reader = new FileReader();
  reader.onload = (e) => {
    uploadZone.innerHTML = `<img src="${e.target.result}" alt="Preview">`;
    uploadZone.classList.add('has-preview');
  };
  reader.readAsDataURL(file);
}
```

**FileReader** reads the file's binary content and converts it to a **data URL** — a Base64-encoded string that can be used as an `<img>` `src`. This creates a preview of the image without uploading it to any server.

`reader.readAsDataURL(file)` starts the read. When complete, `reader.onload` fires with the result.

A data URL looks like: `data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ...`

The upload zone's content is replaced with the preview image, and the `.has-preview` class changes the zone's styling (solid border, no padding).

#### The Upload Request

```javascript
uploadBtn.addEventListener('click', async () => {
  if (!selectedFile) return;

  uploadBtn.disabled = true;
  uploadBtn.innerHTML = '<div class="spinner"></div> Uploading...';
```

Disable the button and show a spinner inside it. This provides visual feedback during the upload.

```javascript
  const formData = new FormData();
  formData.append('image', selectedFile);
  formData.append('caption', document.getElementById('caption').value);
```

**`FormData`** is the Web API for creating `multipart/form-data` requests. This is the **only** way to send files via `fetch`.

- `formData.append('image', selectedFile)` — Adds the binary file with field name `'image'` (matching the Multer config: `upload.single('image')`)
- `formData.append('caption', captionText)` — Adds the text caption

```javascript
  try {
    const res = await fetch(`${BACKEND_URL}/api/posts`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` },
      body: formData
    });
```

**Critical detail:** The headers object has ONLY `Authorization`. There is **no** `Content-Type` header. This is intentional.

When you pass a `FormData` object as the `body`, the browser automatically sets:
```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryABC123
```

If you manually set `Content-Type: application/json`, you'd override this automatic behavior, and Multer wouldn't be able to parse the file. So for file uploads, **never** set `Content-Type` yourself.

```javascript
    const data = await res.json();
    if (!res.ok) throw new Error(data.message);

    showSuccess('Pin published!');
    selectedFile = null;
```

On success, clear the selected file reference.

```javascript
    uploadZone.innerHTML = `
      <svg ...><!-- upload icon --></svg>
      <p>Click or drag to upload an image</p>
      <small>JPG, PNG, WEBP — Max 5MB</small>
    `;
    uploadZone.classList.remove('has-preview');
    document.getElementById('caption').value = '';
```

Reset the upload zone back to its original state — remove the preview image, restore the upload icon and instructions, clear the caption.

```javascript
    setTimeout(() => { window.location.href = '/feed.html'; }, 1500);
```

Wait 1.5 seconds (so the user sees the success message), then redirect to the feed where they can see their new post.

---

### `profile.js` — Profile Page with Follow

#### Parallel API Calls

```javascript
const [userRes, postsRes] = await Promise.all([
  fetch(`${BACKEND_URL}/api/users/${profileId}`, { headers: getHeaders() }),
  fetch(`${BACKEND_URL}/api/posts/user/${profileId}`, { headers: getHeaders() })
]);
```

**`Promise.all`** fires both requests **simultaneously** and waits for both to complete. Without `Promise.all`, you'd do:

```javascript
const userRes = await fetch(...);   // Wait for this...
const postsRes = await fetch(...);  // THEN start this
```

That's sequential — if each takes 200ms, the total is 400ms. With `Promise.all`, both run in parallel, so the total is ~200ms (the slower of the two).

The **array destructuring** `[userRes, postsRes]` extracts the results in order.

#### Populating the Profile

```javascript
document.getElementById('profileAvatar').src = user.profilePic;
document.getElementById('profileUsername').textContent = user.username;
document.getElementById('profileEmail').textContent = user.email;
document.getElementById('postsCount').textContent = posts.length;
document.getElementById('followersCount').textContent = user.followers.length;
document.getElementById('followingCount').textContent = user.following.length;
document.title = `SnapGallery — ${user.username}`;
```

Each DOM element that was empty in the HTML gets filled with data from the API response. `document.title` changes the browser tab title.

#### Conditional Follow Button

```javascript
const actions = document.getElementById('profileActions');
if (!isOwnProfile) {
  const isFollowing = user.followers.some(f => (f._id || f) === currentUserId);
  actions.innerHTML = `
    <button class="btn ${isFollowing ? 'btn-secondary' : 'btn-primary'}" id="followBtn">
      ${isFollowing ? 'Unfollow' : 'Follow'}
    </button>
  `;
  document.getElementById('followBtn').addEventListener('click', () => handleFollow());
}
```

**`user.followers.some(...)`** — The `some()` array method returns `true` if at least one element passes the test. It checks if the current user's ID is in the target user's followers list.

**Why `(f._id || f)`?** The `followers` array might be populated (objects with `_id`) or unpopulated (raw ObjectId strings), depending on the API response. `f._id || f` handles both cases.

If `isOwnProfile` is true (viewing your own profile), no follow button is rendered.

#### Follow/Unfollow Handler

```javascript
async function handleFollow() {
  const btn = document.getElementById('followBtn');
  try {
    const res = await fetch(`${BACKEND_URL}/api/users/${profileId}/follow`, {
      method: 'PUT',
      headers: getHeaders()
    });
    const data = await res.json();

    btn.textContent = data.isFollowing ? 'Unfollow' : 'Follow';
    btn.className = `btn ${data.isFollowing ? 'btn-secondary' : 'btn-primary'} btn-sm`;
    document.getElementById('followersCount').textContent = data.followersCount;

    showToast(data.isFollowing ? 'Following!' : 'Unfollowed');
```

After the API responds:
1. Update button text — "Follow" ↔ "Unfollow"
2. Update button style — Primary (red, "Follow") ↔ Secondary (gray, "Unfollow")
3. Update follower count on the page
4. Show a toast notification

#### Delete Post on Profile — With Count Update

```javascript
async function deletePost(postId, btn) {
  if (!confirm('Delete this post?')) return;
  try {
    await fetch(`${BACKEND_URL}/api/posts/${postId}`, {
      method: 'DELETE',
      headers: getHeaders()
    });
    btn.closest('.pin-card').remove();
    const count = document.getElementById('postsCount');
    count.textContent = parseInt(count.textContent) - 1;
    showToast('Post deleted');
```

Same as the feed's delete, but with an extra step: **decrement the posts count**. `parseInt(count.textContent) - 1` converts the text "5" to number 5, subtracts 1, and sets it back to "4".

---

## Summary — Data Flow for Every User Action

| User Action       | Frontend JS             | API Endpoint                        | Backend Controller    | DOM Update                        |
| ----------------- | ----------------------- | ----------------------------------- | --------------------- | --------------------------------- |
| Login             | `auth.js` submit        | `POST /api/auth/login`              | `login()`             | Store token → redirect            |
| Register          | `auth.js` submit        | `POST /api/auth/register`           | `register()`          | Store token → redirect            |
| View Feed         | `feed.js` loadFeed()    | `GET /api/posts/feed`               | `getFeed()`           | Build masonry grid                |
| Like Post         | `feed.js` toggleLike()  | `PUT /api/posts/:id/like`           | `toggleLike()`        | Toggle heart, update count        |
| Delete Post       | `feed.js` deletePost()  | `DELETE /api/posts/:id`             | `deletePost()`        | Remove card from DOM              |
| Search Users      | `feed.js` input event   | `GET /api/users/search?q=...`       | `searchUsers()`       | Render dropdown results           |
| Upload Image      | `upload.js` click       | `POST /api/posts` (FormData)        | `createPost()`        | Show success → redirect to feed   |
| View Profile      | `profile.js` loadProfile| `GET /api/users/:id` + `GET posts`  | `getUser()` + `getUserPosts()` | Fill profile fields    |
| Follow/Unfollow   | `profile.js` handleFollow| `PUT /api/users/:id/follow`         | `toggleFollow()`      | Toggle button, update count       |
| Logout            | Any page logoutBtn      | (none — client-side only)           | (none)                | Clear localStorage → redirect     |
