---
name: github-vercel-setup
description: >
  Sets up a new website project with a GitHub repository and Vercel deployment for Rune's
  projects. Use this skill whenever the user asks to create a new website, new project, add
  to GitHub, deploy to Vercel, set up hosting, or publish a site — even if they only mention
  one of these (e.g. "add it to GitHub" or "put it on Vercel" implies doing both). Also use
  when creating a new single-file HTML project from scratch that should be hosted. Trigger
  even for partial requests like "can you make this available online" or "create a repo for this".
---

# GitHub + Vercel Setup

This skill handles the full lifecycle of publishing a new static website:
creating the GitHub repo, pushing the code, and deploying to Vercel.

## Design discovery (do this first, before writing any code)

When the user asks to create a new website or project from scratch, **ask these questions before building anything**. You need the answers to make good decisions about layout, navigation, and structure — don't guess.

Ask all of these in a single message so the user can answer them together:

1. **Single page or multiple pages?** — One long scrolling page, or separate pages (About, Contact, etc.)?
2. **Navigation style** — Top navigation bar, sidebar, bottom tab bar (app-style), hamburger menu, or none?
3. **Device focus** — Designed primarily for mobile/tablet (app-like feel), desktop, or responsive for both?
4. **Visual style** — Minimal/clean, bold/colorful, professional/corporate, playful/creative?
5. **Key sections needed** — e.g. hero banner, about, gallery, contact form, pricing, etc.

If the user has already answered some of these in their original request (e.g. "a simple one-pager"), skip those and only ask what's missing. If the project is clearly an app (like the garden planner), you can infer answers and confirm rather than ask.

Once you have the answers, propose a project name for the directory/repo/Vercel URL, confirm it with the user, then proceed to build the HTML and set up GitHub + Vercel.

---

## User's credentials & config

- **GitHub username**: `runekeldsen`
- **GitHub PAT**: found in `/home/rune-keldsen/garden-planner/.git/config` (read it fresh each time — don't hardcode)
- **Git identity**: name `Rune Keldsen`, email `runekeldsen@gmail.com`
- **Vercel CLI**: installed and authenticated — run `vercel --yes` to deploy
- **`gh` CLI**: NOT available — use `curl` against the GitHub REST API instead
- **Project style**: single `index.html` file, no build step, no `package.json`
- **Projects live at**: `/home/rune-keldsen/<project-name>/`

## Step-by-step workflow

Run these steps in order. Most can be chained with `&&`.

### 1. Read the PAT

```bash
# Extract PAT from garden-planner remote URL
git -C /home/rune-keldsen/garden-planner remote get-url origin
```

The URL format is `https://runekeldsen:<PAT>@github.com/...` — extract the token between `:` and `@`.

### 2. Initialise the local repo

```bash
cd /home/rune-keldsen/<project-name>
git init
git config user.email "runekeldsen@gmail.com"
git config user.name "Rune Keldsen"
git branch -m main
git add index.html
git commit -m "<short description of project>"
```

If the directory doesn't exist yet, create it first (`mkdir`) and create `index.html` before this step.

### 3. Create the GitHub repository

```bash
PAT="<token-from-step-1>"
curl -s -X POST https://api.github.com/user/repos \
  -H "Authorization: token $PAT" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"<project-name>\",\"description\":\"<one-line description>\",\"private\":false}"
```

A `422` response with `"name already exists on this account"` is fine — the repo is already there, continue.

### 4. Add remote and push

```bash
PAT="<token-from-step-1>"
git remote add origin "https://runekeldsen:${PAT}@github.com/runekeldsen/<project-name>.git"
git push -u origin main
```

If a remote named `origin` already exists, remove it first:
```bash
git remote remove origin
```

### 5. Deploy to Vercel

Run from inside the project directory:

```bash
vercel --prod --yes
```

Vercel auto-detects the static HTML file (no framework, no build command needed). It:
- Links the directory to a new Vercel project (first run)
- Deploys to production
- Aliases to `https://<project-name>.vercel.app`

The `.vercel/` directory it creates is gitignored automatically.

**Important**: GitHub auto-deploy is NOT reliably connected (the Vercel GitHub integration often shows a warning and doesn't wire up). Always deploy with `vercel --prod --yes` — both on first setup and every time changes are pushed. Do not tell the user that git push will auto-deploy.

### 6. Deploy on every future change

After every `git push`, also run:

```bash
cd /home/rune-keldsen/<project-name> && vercel --prod --yes
```

### 7. Report to the user

Always end with:
- **Live URL**: `https://<project-name>.vercel.app`
- **GitHub**: `https://github.com/runekeldsen/<project-name>`

## Common issues

| Problem | Fix |
|---|---|
| `git commit` fails with "Author identity unknown" | Run `git config user.email` and `git config user.name` in the project dir |
| `remote origin already exists` | `git remote remove origin` then re-add |
| GitHub 422 on repo create | Repo already exists — skip creation, continue with push |
| Vercel asks interactive questions | Always use `vercel --yes` to accept defaults non-interactively |
| Vercel GitHub connection warning | Ignore — but auto-deploy will NOT work; always run `vercel --prod --yes` manually |

## Notes

- Never hardcode the PAT — always read it fresh from the garden-planner config
- The project name used for the directory, GitHub repo, and Vercel project should match
- Vercel deploys the root `index.html` automatically when there is no build command
- The live URL is reliably `https://<project-name>.vercel.app` after the first deploy
