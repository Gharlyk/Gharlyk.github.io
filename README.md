# Gharlyk.github.io — source

This is the source for my portfolio/blog, published via GitHub Pages at https://gharlyk.github.io.

- New writeup → add a file to `_posts/` named `YYYY-MM-DD-title.md` (see `workflow/PUBLISHING_WORKFLOW.md` in the portfolio kit for the exact routine).
- New project listing → add a file to `_projects/`.
- Local preview: `bundle install && bundle exec jekyll serve`, then open http://localhost:4000.

GitHub Pages builds this automatically on push to `main` — no CI setup needed as long as the repo is named exactly `Gharlyk.github.io` and Pages is enabled in repo Settings → Pages (source: `main` branch, root).
