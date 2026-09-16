# Encyclopedia

A personal, growing encyclopedia built from Claude conversations, one
question at a time. Built on [Quartz](https://quartz.jzhao.xyz).

## Adding a page

Ask Claude a question, then run `/reference` (the skill in
`.claude/skills/reference/`). Claude will turn the discussion into a clean
reference page under `content/`, wikilinking it to related notes, and either
extend an existing page or create a new one as appropriate. It commits the
change but doesn't push — run `git push` (or ask Claude to) when you want it
live.

## Editing by hand

Edit the `.md` files in `content/` directly — rephrase, add detail, correct
mistakes. Just commit your edits separately from anything Claude generates
(one topic per commit, same as the skill does). There's no visible marker
in the page distinguishing Claude's writing from yours by design; the split
lives in git history — `git log --follow -- content/<file>.md` or
`git blame` on any note shows who wrote what.

## Local preview

```
npx quartz build --serve
```

Serves the site at `http://localhost:8080` and rebuilds on file changes.

## Publishing

Push to `main`; a GitHub Actions workflow (`.github/workflows/deploy.yml`)
builds and deploys to GitHub Pages automatically. One-time setup: in the
repo's Settings → Pages, set Source to "GitHub Actions".
