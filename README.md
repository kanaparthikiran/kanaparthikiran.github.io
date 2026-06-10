# kanaparthikiran.github.io

Personal site of Kiran Kanaparthi. Built with [Jekyll](https://jekyllrb.com/) +
the [`minima`](https://github.com/jekyll/minima) theme; published at
<https://kanaparthikiran.github.io>.

## Local preview

```bash
bundle install
bundle exec jekyll serve
# → http://127.0.0.1:4000
```

## Structure

```
.
├── _config.yml          # Jekyll configuration
├── _posts/              # Blog posts (filename = YYYY-MM-DD-slug.md)
├── index.md             # Homepage
├── about.md             # About / portfolio page
├── Gemfile              # Ruby dependencies
└── README.md            # this file
```

## Adding a new post

Create a file in `_posts/` named `YYYY-MM-DD-slug.md` with this frontmatter:

```yaml
---
layout: post
title: "Your title here"
date: 2026-06-01
tags: [tag1, tag2]
---
```

Push to `main` and GitHub Pages rebuilds within a minute.
