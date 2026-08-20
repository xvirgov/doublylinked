# doublylinked

Source for [doublylinked.com](https://doublylinked.com) — a list of my tech notes, mostly
security-devopsy-cloudy-software.

## stack

- [Jekyll](https://jekyllrb.com/) — static site generator
- [jekyll-theme-console](https://github.com/b2a3e8/jekyll-theme-console) — terminal-style theme via `remote_theme`
- GitHub Pages — hosting (build + deploy via GitHub Actions)
- Custom domain via DNS A/AAAA + CNAME records

## writing a new post

1. Create a file in `_posts/` named `YYYY-MM-DD-some-slug.md`.
2. Start it with frontmatter:

   ```yaml
   ---
   layout: post
   title: "your title here"
   date: 2026-05-14
   tags: [optional, tags]
   ---
   ```

3. Write markdown below the frontmatter.
4. Commit and push to `main`. GitHub Actions rebuilds the site in ~30–60 seconds.

## local preview (optional)

Requires Ruby 3.x and Bundler.

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

## DNS setup reminder

A records for the apex `doublylinked.com`:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

AAAA records (IPv6):

```
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153
```

CNAME for `www` → `xvirgov.github.io.`

The `CNAME` file in the repo root tells GitHub Pages to serve the site on the custom domain.
Set these records to **DNS only** (not proxied) if using Cloudflare, so GitHub can provision HTTPS.
