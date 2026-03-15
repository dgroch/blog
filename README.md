# Your Blog

A clean, Medium-inspired blog powered by Jekyll and GitHub Pages.  
Dark mode · RSS feed · Custom domain ready · CSTMR-aligned palette.

---

## Quick start

### Option A: GitHub Pages (easiest)

1. Create a new repo on GitHub
2. Push this folder to it
3. Go to **Settings → Pages → Source** → select `main` branch
4. Your blog is live at `https://yourusername.github.io/repo-name/`

### Option B: Run locally

```bash
gem install bundler jekyll
bundle install
bundle exec jekyll serve
```

Then open `http://localhost:4000`

---

## Writing a post

Create a new `.md` file in `_posts/` with the naming format:

```
YYYY-MM-DD-your-post-title.md
```

Add front matter at the top:

```yaml
---
layout: post
title: "Your post title"
subtitle: "Optional subtitle shown on the homepage"
date: 2026-03-15
tags: [tag1, tag2]
---

Your markdown content here.
```

Push to GitHub. Done.

---

## Custom domain setup

### 1. Buy a domain

Use any registrar (Namecheap, Cloudflare, Google Domains, etc).

### 2. Configure DNS

**For an apex domain** (e.g. `yourblog.com`):

Add these A records pointing to GitHub's servers:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

**For a subdomain** (e.g. `blog.yourdomain.com`):

Add a CNAME record:

```
blog.yourdomain.com → yourusername.github.io
```

### 3. Tell GitHub about it

**Option A — via the UI:**
1. Go to your repo → **Settings → Pages**
2. Under "Custom domain", enter your domain (e.g. `yourblog.com`)
3. Check **Enforce HTTPS** (wait a few minutes for the certificate)

**Option B — via a CNAME file:**

Create a file called `CNAME` in the root of your repo containing your domain:

```
yourblog.com
```

### 4. Update `_config.yml`

```yaml
url: "https://yourblog.com"
baseurl: ""  # leave empty for custom domains
```

### 5. Wait for DNS propagation

Usually takes 5–30 minutes. You can check with:

```bash
dig yourblog.com +short
```

You should see GitHub's IP addresses. HTTPS typically provisions within 15 minutes after DNS resolves.

### Troubleshooting

- **"Domain not yet verified"**: Go to your GitHub account settings (not repo settings) → Pages → Add your domain and verify via DNS TXT record
- **Mixed content / broken CSS**: Make sure `url` in `_config.yml` uses `https://`
- **404 after custom domain**: Ensure the CNAME file exists and hasn't been deleted by a build

---

## Features

### Dark mode

Automatic detection based on system preference, with a manual toggle in the nav. Preference saved to `localStorage`.

### RSS feed

Available at `/feed.xml`. Linked in the nav and in the HTML `<head>` for auto-discovery by RSS readers.

### Reading time

Auto-calculated from word count (250 wpm). Shown on both the homepage listing and individual posts.

---

## Customise

| What | Where |
|------|-------|
| Site info (title, description, author) | `_config.yml` |
| Colours & typography | `assets/css/style.css` (CSS custom properties at the top) |
| Navigation links | `_includes/nav.html` |
| Pages | Add `.md` files in the root (like `about.md`) |
| Avatar | Add image to `assets/`, set path in `_config.yml` under `author.avatar` |

### Colour palette reference

The theme uses a palette aligned with CSTMR's visual identity:

| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `--color-text` | `#1a1d2e` | `#e2e4ea` | Body text |
| `--color-accent` | `#1a9a8e` | `#2cc4b6` | Links, tags, blockquote border |
| `--color-warm` | `#d4854a` | `#e69a5e` | Accent bar gradient end |
| `--color-surface` | `#f3f4f7` | `#1e2030` | Code blocks, hover states |
