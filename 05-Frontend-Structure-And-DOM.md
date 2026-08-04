# 05 — Frontend Structure & DOM

## Overview — No Framework, No Build Step

There is no React. No Webpack. No bundler. No `npm run build` for the frontend. The `public/` folder contains raw HTML, CSS, and JavaScript files that the browser loads directly.

Each page is a standalone `.html` file with its own dedicated `.js` file:

```
public/
├── index.html    →  js/auth.js        (Login / Register)
├── feed.html     →  js/feed.js        (Feed, Search, Likes)
├── upload.html   →  js/upload.js      (Image Upload)
├── profile.html  →  js/profile.js     (User Profile, Follow)
└── css/
    └── style.css                      (Single shared stylesheet)
```

Navigation between pages is done via full page loads (`window.location.href`), not client-side routing. When you click "Profile" in the navbar, the browser navigates to `/profile.html` — a completely new HTTP request.

---

## The HTML Pages — Structure Breakdown

### `index.html` — Login / Register Page

This page has **two forms** that share the same space. Only one is visible at a time:

```html
<form id="loginForm">              <!-- Visible by default -->
  <input id="loginEmail" ... >
  <input id="loginPassword" ... >
  <button id="loginBtn">Sign In</button>
</form>

<form id="registerForm" style="display: none;">   <!-- Hidden by default -->
  <input id="regUsername" ... >
  <input id="regEmail" ... >
  <input id="regPassword" ... >
  <button id="registerBtn">Create Account</button>
</form>

<div class="auth-toggle">
  <span id="toggleText">Don't have an account?</span>
  <a href="#" id="toggleLink">Sign up</a>
</div>
```

The toggle link (`#toggleLink`) switches between the two forms by changing `display: none` and `display: block` via JavaScript. No page reload.

**Important:** The `<script>` tag is at the bottom of `<body>`:

```html
  <script src="/js/auth.js"></script>
</body>
```

This is intentional. If the script were in `<head>`, it would run before the HTML is parsed, and `document.getElementById(...)` would return `null` because the elements don't exist yet. By placing the script at the bottom, all HTML elements are already in the DOM when the script runs.

### `feed.html` — The Main Feed Page

**Structure:**

```
<nav>                         ← Sticky navbar with logo, search, links
  ├── .navbar-brand           ← Logo + "SnapGallery" text
  ├── .search-container       ← Search input + dropdown results
  └── .navbar-links           ← Home, Create, Profile, Logout
</nav>

<main class="main-content">
  ├── #loadingState           ← Spinner shown while fetching feed
  ├── #emptyState             ← "Your feed is empty" message (hidden by default)
  └── #feedGrid               ← Masonry grid container (empty, filled by JS)
</main>

<div id="toastContainer">    ← Toast notifications container
```

The `#feedGrid` div starts **empty**. JavaScript fills it after fetching data from the API. This is the core of DOM manipulation — the page loads with a skeleton, then JS paints the content.

### `upload.html` — Image Upload Page

```html
<div class="upload-zone" id="uploadZone">
  <svg ...></svg>                              ← Upload icon
  <p>Click or drag to upload an image</p>
  <small>JPG, PNG, WEBP — Max 5MB</small>
</div>
<input type="file" id="imageInput" accept="image/*">   ← Hidden file input

<textarea id="caption" ...></textarea>
<button id="uploadBtn" disabled>Publish Pin</button>
```

The `<input type="file">` is **hidden** (`display: none` in CSS). The visible upload zone is a styled `<div>`. When you click the div, JavaScript triggers the hidden file input's click event. This is a common pattern for custom file upload UIs.

The `disabled` attribute on the button means it can't be clicked until a file is selected. JavaScript removes this attribute after a file is chosen.

### `profile.html` — User Profile Page

```html
<div id="profileContent" style="display: none;">
  <div class="profile-header">
    <img id="profileAvatar" src="" alt="Profile">     ← Empty src, filled by JS
    <h1 id="profileUsername"></h1>                      ← Empty, filled by JS
    <p id="profileEmail"></p>                           ← Empty, filled by JS
    <div class="profile-stats">
      <div class="stat-value" id="postsCount">0</div>
      <div class="stat-value" id="followersCount">0</div>
      <div class="stat-value" id="followingCount">0</div>
    </div>
    <div id="profileActions"></div>                     ← Follow button injected here
  </div>
  <div id="postsGrid"></div>                            ← User's posts grid
</div>
```

The entire `#profileContent` starts hidden (`display: none`). JavaScript fetches the user data, fills in the empty elements, then reveals the content. This prevents the user from seeing empty placeholders while data is loading.

**Query parameter routing:** The profile page uses a URL query parameter to determine which user to display:

```
/profile.html           → Shows YOUR profile (no ?id= param)
/profile.html?id=abc123 → Shows user abc123's profile
```

JavaScript reads this with `new URLSearchParams(window.location.search)`.

---

## The CSS — `public/css/style.css`

### Design System — CSS Custom Properties

The entire color palette and spacing system is defined using CSS custom properties (variables) in `:root`:

```css
:root {
  --bg-primary: #0a0a0a;        /* Page background — near black */
  --bg-secondary: #141414;      /* Input backgrounds */
  --bg-card: #1a1a1a;           /* Card backgrounds */
  --bg-elevated: #222222;       /* Hover states, active nav links */
  --border-color: #2a2a2a;      /* Default borders */
  --border-hover: #3a3a3a;      /* Hover state borders */
  --text-primary: #f0f0f0;      /* Main text — off white */
  --text-secondary: #a0a0a0;    /* Secondary text — gray */
  --text-muted: #666666;        /* Placeholder text, icons */
  --accent: #e60023;             /* Pinterest red — primary action color */
  --accent-hover: #cc001f;       /* Darker red for hover */
  --accent-soft: rgba(230, 0, 35, 0.12);  /* Soft red tint for backgrounds */
  --success: #00c853;            /* Green for success messages */
  --radius-sm: 8px;
  --radius-md: 12px;
  --radius-lg: 16px;
  --radius-xl: 24px;             /* Pill-shaped buttons */
  --shadow: 0 4px 24px rgba(0,0,0,0.4);
  --transition: 0.2s ease;
}
```

Every component uses these variables. To change the entire color scheme, you only need to update `:root`. For example, changing `--bg-primary` from `#0a0a0a` to `#ffffff` would make the entire app light-themed.

### The Masonry Grid — CSS Columns

The Pinterest-style masonry layout is achieved with pure CSS:

```css
.masonry-grid {
  columns: 4;          /* 4 equal-width columns */
  column-gap: 16px;    /* Space between columns */
}

.pin-card {
  break-inside: avoid; /* Don't split a card across columns */
  margin-bottom: 16px; /* Space between cards vertically */
}
```

**How CSS columns work:** The browser fills column 1 top-to-bottom, then column 2, then column 3, etc. `break-inside: avoid` ensures a card isn't split across two columns.

This is different from CSS Grid or Flexbox. CSS columns are designed for text layout (like newspaper columns), but they work perfectly for masonry when combined with `break-inside: avoid`.

**Responsive breakpoints:**

```css
@media (max-width: 1200px) {
  .masonry-grid { columns: 3; }   /* Tablet: 3 columns */
}

@media (max-width: 768px) {
  .masonry-grid { columns: 2; }   /* Mobile: 2 columns */
}
```

### The Navbar — Sticky + Glassmorphism

```css
.navbar {
  position: sticky;
  top: 0;
  z-index: 100;
  background: rgba(10, 10, 10, 0.85);    /* Semi-transparent */
  backdrop-filter: blur(20px);             /* Frosted glass effect */
  border-bottom: 1px solid var(--border-color);
}
```

`position: sticky; top: 0;` makes the navbar stick to the top when you scroll. `backdrop-filter: blur(20px)` creates the frosted glass effect — content behind the navbar appears blurred. `z-index: 100` ensures the navbar stays above other content.

### Animations

Three CSS animations are defined:

```css
@keyframes slideUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

- **`slideUp`** — Used on the auth card. Slides up and fades in when the page loads.
- **`spin`** — Used on the loading spinner. Continuous rotation.
- **`fadeIn`** — Used on pin cards. Each card fades in when rendered. Applied via the `.fade-in` class.

### Toast Notifications

```css
.toast-container {
  position: fixed;        /* Stays in place even when scrolling */
  bottom: 24px;
  right: 24px;
  z-index: 1000;          /* Above everything */
}

.toast {
  animation: slideUp 0.3s ease;   /* Slides up when it appears */
}

.toast.success { border-left: 3px solid var(--success); }  /* Green bar */
.toast.error { border-left: 3px solid #ef4444; }           /* Red bar */
```

Toasts are dynamically created by JavaScript. They appear in the bottom-right corner, stay for 3 seconds, then are removed from the DOM.

### The Upload Zone

```css
.upload-zone {
  border: 2px dashed var(--border-color);   /* Dashed border = "drop here" */
  cursor: pointer;
}

.upload-zone:hover {
  border-color: var(--accent);              /* Red border on hover */
  background: var(--accent-soft);           /* Soft red tint */
}

.upload-zone.has-preview {
  padding: 0;                               /* Remove padding when image preview shown */
  border-style: solid;                      /* Solid border instead of dashed */
}
```

The `.has-preview` class is added by JavaScript after an image is selected, changing the zone from "drop area" to "preview area."

### Hidden File Input

```css
#imageInput {
  display: none;
}
```

The actual `<input type="file">` is invisible. The visible "Click or drag to upload" zone is a styled div that triggers the hidden input programmatically.

---

## DOM Manipulation Patterns

### Pattern 1: Guard Redirect

Every protected page starts with:

```javascript
const token = localStorage.getItem('token');
if (!token) window.location.href = '/';
```

If there's no JWT in localStorage, immediately redirect to the login page. This is a **client-side guard** — it prevents unauthenticated users from seeing the page. However, it's NOT a security measure. The real security is on the backend (the API will reject requests without valid tokens). This is just for UX — so users don't see a broken page.

### Pattern 2: Toggle Form Visibility

```javascript
// auth.js
toggleLink.addEventListener('click', (e) => {
  e.preventDefault();    // Prevent the <a> tag from navigating
  isLogin = !isLogin;    // Flip the boolean

  if (isLogin) {
    loginForm.style.display = 'block';
    registerForm.style.display = 'none';
  } else {
    loginForm.style.display = 'none';
    registerForm.style.display = 'block';
  }
});
```

`e.preventDefault()` is crucial here. The toggle link is an `<a href="#">`. Without `preventDefault()`, clicking it would add `#` to the URL and scroll to the top of the page.

### Pattern 3: Template Literals for HTML Generation

Instead of creating elements one by one with `document.createElement()`, this project uses template literals to build HTML strings:

```javascript
grid.innerHTML = posts.map(post => `
  <div class="pin-card fade-in">
    <img src="${post.imageUrl}" alt="${post.caption || 'Pin'}" loading="lazy">
    <div class="pin-info">
      ...
    </div>
  </div>
`).join('');
```

**How this works:**
1. `posts.map(...)` transforms each post object into an HTML string
2. `.join('')` concatenates all HTML strings into one big string
3. `grid.innerHTML = ...` replaces the grid's content with the generated HTML

This is faster and more readable than creating each element imperatively. The downside is that event listeners must use `onclick` attributes (inline handlers) because the elements are created as strings, not DOM nodes.

### Pattern 4: XSS Protection with `escapeHtml()`

```javascript
function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}
```

**What problem does this solve?** If a user sets their caption to `<script>alert('hacked')</script>`, and you insert it via `innerHTML`, the browser would execute that script. This is called **XSS (Cross-Site Scripting)**.

**How the fix works:**
1. Create a temporary div element (not added to the page)
2. Set `.textContent` — this treats the string as plain text, not HTML
3. Read `.innerHTML` — this returns the HTML-escaped version

Result: `<script>alert('hacked')</script>` becomes `&lt;script&gt;alert('hacked')&lt;/script&gt;`. The browser renders it as visible text, not executable code.

### Pattern 5: Lazy Loading Images

```html
<img src="${post.imageUrl}" loading="lazy">
```

`loading="lazy"` tells the browser: "Don't download this image until it's about to scroll into view." For a feed with 50 posts, this means only the first few images are loaded immediately. The rest load on-demand as the user scrolls. This dramatically improves initial page load time.

### Pattern 6: URL Query Parameters

```javascript
// profile.js
const params = new URLSearchParams(window.location.search);
const profileId = params.get('id') || currentUserId;
const isOwnProfile = profileId === currentUserId;
```

`window.location.search` returns the query string. For `/profile.html?id=abc123`, it returns `"?id=abc123"`.

`URLSearchParams` parses it into a key-value map. `params.get('id')` returns `"abc123"`.

If there's no `?id=` parameter (user clicked "Profile" in the navbar), `params.get('id')` returns `null`, and `|| currentUserId` falls back to the logged-in user's ID.

`isOwnProfile` is a boolean flag used to conditionally show/hide UI elements:
- Own profile → Show delete buttons on posts, no follow button
- Other user's profile → Show follow/unfollow button, no delete buttons
