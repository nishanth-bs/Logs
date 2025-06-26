## Hi there 👋

This site uses:
- **Hugo** static site generator
- **GitHub Pages**, hosted from `/docs` folder in `main` branch
- **PaperMod** theme

---

## ✅ One-Time Setup

### 1. Install Hugo

```bash
brew install hugo   # For macOS
```

---

## 🛠 Site Build & Deploy Commands

### 1. Clean previous builds

```bash
rm -rf public docs resources
```

### 2. Build the site into `/docs`

```bash
hugo -d docs
```

### 3. Ensure GitHub Pages serves all files

```bash
touch docs/.nojekyll
```

### 4. Commit and push

```bash
git add docs
git commit -m "build site"
git push origin main
```

---

## ⚙️ `hugo.toml` Configuration

```toml
baseURL = "https://nishanth-bs.github.io/"
relativeURLs = true
canonifyURLs = true
languageCode = "en-us"
title = "Decoded With Nishanth"
theme = "PaperMod"

[params]
  author = "Nishanth"
  showReadingTime = true
  showShareButtons = true
  defaultTheme = "auto"
```

---

## 🔧 GitHub Pages Settings

Go to **Repo → Settings → Pages**:

- ✅ Source: `main` branch
- ✅ Folder: `/docs`

---
## ✍️ Writing and Publishing a New Blog Post

### 1. Create a new post

```bash
hugo new posts/my-new-post.md
```

> This creates the file at `content/posts/my-new-post.md`

---

### 2. Edit your post

Open it in your editor and write in Markdown:

```md
---
title: "My New Post"
date: 2025-06-26T10:00:00+05:30
draft: false
---

Your content goes here...
```

Make sure `draft: false` so Hugo includes it in the build.

---

### 3. Rebuild and deploy

```bash
hugo -d docs
touch docs/.nojekyll
git add content/posts docs
git commit -m "Add new post: My New Post"
git push origin main
```

> Your blog is now live at: `https://nishanth-bs.github.io/posts/my-new-post/`
---

## 🧪 Troubleshooting

### 🔥 CSS/JS not loading?

- Check your `docs/index.html` → CSS should use:
  ```html
  <link href="./assets/css/stylesheet....css" ...>
  ```
- Make sure you set:
  ```toml
  relativeURLs = true
  canonifyURLs = true
  ```

### 🚫 Getting 404 on styles/scripts?

- Ensure you built with **correct `baseURL`**
  ```toml
  baseURL = "https://nishanth-bs.github.io/"
  ```
- Rebuild and check that `docs/assets/` actually contains the CSS

### 🐛 Seeing `/Logs/...` in output?

- You had a stale `baseURL` like:
  ```toml
  baseURL = "https://nishanth-bs.github.io/Logs"
  ```
  ❌ Wrong → change it and rebuild from scratch

### 🧼 Clean rebuild checklist

```bash
rm -rf docs public resources
hugo -d docs
touch docs/.nojekyll
git add docs
git commit -m "clean rebuild"
git push origin main
```

---

## 🧠 Tip

Always check what `index.html` links to:

```bash
cat docs/index.html | grep stylesheet
```

If it starts with `/assets/`, that’s 💥 wrong  
If it starts with `./assets/` or full `https://...`, you’re golden ✅
---
