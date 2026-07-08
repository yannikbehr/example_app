---
name: publish-github-pages
description: Publish a static, single-page web app (like index.html) through GitHub Pages, and harden it for real-world hosting. Use when the user wants to "publish", "deploy", "serve", or "host" a site/app on GitHub Pages, or reports it breaking once live (CDN errors, blank page, misleading error messages).
---

# Publish a static app through GitHub Pages

For a small app that's just static HTML/CSS/JS (a single `index.html`, or a handful of
static assets) with no build step required.

## 1. Confirm the site is Pages-ready

GitHub Pages serves files as-is from a branch/folder — no build step needed for a plain
static page. Check:

- `index.html` exists at the path you intend to serve from (repo root, or `/docs`).
- Any assets it references (scripts, images, CSS) are either committed to the repo or
  loaded from a CDN over HTTPS.
- There's no server-side code (PHP, etc.) — Pages only serves static files.

If those hold, no code changes are required just to "turn on" Pages.

## 2. Enable Pages (manual, one-time, cannot be automated from here)

Enabling Pages is a repo *setting*, not a file in the repo, and there is no GitHub API
tool available in this environment to flip it — it has to be done by the repo owner in
the GitHub UI:

1. `https://github.com/<owner>/<repo>/settings/pages`
2. **Build and deployment → Source**: choose **Deploy from a branch**
3. **Branch**: pick the branch (usually `main`) and folder (`/` root, or `/docs`)
4. **Save** — the site publishes at `https://<owner>.github.io/<repo>/` within a minute
   or two.

Don't try to work around this by adding a workflow file "to enable Pages" — a
`.github/workflows/pages.yml` using `actions/deploy-pages` is a *legitimate alternative*
(gives deploy history/logs, useful if a build step gets added later) but still requires
the same Settings page, with Source set to **GitHub Actions** instead of a branch. Ask
the user which they want; don't add a workflow file speculatively if "deploy from
branch" already satisfies the request.

## 3. Harden the app for real-world hosting

Static demo apps that lean on a CDN script (Chart.js, etc.) and a public API commonly
fail in the wild in ways that are easy to misdiagnose:

- **Don't lump unrelated failure modes into one catch-all error.** A `try/catch` around
  "load library, then fetch data, then render" will blame the API/CORS for a CDN script
  that never loaded. Check preconditions explicitly (e.g. `typeof Chart === "undefined"`)
  and give each failure mode its own accurate message.
- **Prefer the CDN a library's own docs recommend.** jsDelivr is generally more
  consistently reachable through corporate filters/ad blockers than cdnjs for common
  libraries like Chart.js — a real, previously-seen cause of "works for me, broken for
  the user" bug reports.
- **Verify the failure mode locally before trusting a guess.** Chromium is preinstalled;
  drive it with Playwright and capture `pageerror`/`console` events against a local
  `python3 -m http.server`:

  ```js
  const { chromium } = require('/opt/node22/lib/node_modules/playwright');
  const browser = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium' });
  const page = await browser.newPage();
  page.on('pageerror', e => console.log('PAGEERROR:', e.message));
  await page.goto('http://localhost:PORT/index.html', { waitUntil: 'networkidle' });
  ```

  Note: this sandbox's own egress proxy often blocks third-party CDN domains
  (403 on cdnjs, jsDelivr, etc.) — that failure is about *this environment*, not
  evidence the CDN is broken for real users. It's still useful: it reliably reproduces
  "CDN unreachable" so you can confirm the app's error handling behaves correctly under
  that condition.

## 4. Pushing the change, in Claude Code on the web

Public repos can be *read* (clone, `git ls-remote`, file contents) with no GitHub
connection at all, which can hide the fact that *write* access was never actually set
up. If `git push` and/or a GitHub API write both fail with 403 while reads work fine,
that's the signature of missing write auth, not a transient network error — don't retry
in a loop, fix the connection:

1. Have the user install/authorize the Claude GitHub App at
   `https://github.com/apps/claude`, granting it access to the target repo (it should
   then appear under the repo owner's `github.com/settings/applications` →
   **Installed GitHub Apps**), **or**
2. If they use the `gh` CLI locally: `gh auth login`, then in the Claude Code CLI
   `/login` followed by `/web-setup`.

Retry the push once either is done.

### Stop-hook "Unverified commit" false positive

`~/.claude/stop-hook-git-check.sh` checks `git log --format=%G?` locally, which can
report `N` (no signature) even for a commit that genuinely has a `gpgsig` block, if this
environment's local signer-verification file (`~/.ssh/commit_signing_key.pub` /
`gpg.ssh.allowedSignersFile`) is empty or unconfigured — that's an environment gap, not
a real missing signature. Before re-amending in a loop, check the raw commit object:

```bash
git cat-file commit HEAD | head -10   # look for a real gpgsig ... block
```

If a signature is present, amending again won't change anything — the real verification
happens server-side on GitHub once pushed, independent of this local check.

## 5. Opening the PR

Check for `.github/pull_request_template.md` (or similar) before writing the PR body; if
none exists, a plain Summary / Test plan body is fine.
