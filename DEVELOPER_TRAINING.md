# Snyk Developer Training — OWASP Juice Shop (Node.js / TypeScript)

**Repository:** <https://github.com/lmaeda/juice-shop>
**Training branch:** `ai-agent-snyk-fix-training`
**Audience:** application developers
**Duration:** ~3 hours hands-on (or 2 × 90 min)
**Version:** v3.0 · August 2026

> ⚠️ **This repository is intentionally vulnerable.** OWASP Juice Shop ships real, exploitable
> flaws and deliberately outdated dependencies, and `terraform/` plus `infrastructure/` contain
> intentionally insecure IaC. Never deploy any of it anywhere reachable, and never copy code out
> of it into a production project.

---

## Status legend

Snyk ships features in stages. Every step below is tagged so you know what to expect in your own
organization today:

| Tag | Meaning |
| --- | --- |
| **GA** | Generally available on plans that include the product |
| **EA** | Early Access — must be enabled by Snyk, and/or gated by plan |
| **Preview** | Experimental / design partner — expect change, probably not enabled for you |

If a tagged step doesn't work in your org, that is expected. Skip it and move on — your Snyk admin
or account team controls enablement.

> **A note on version drift.** Snyk ships Snyk Code accuracy improvements roughly monthly, so
> finding *counts* move. The *classes* of finding are the stable part. Always capture your own
> baseline (§3.5) rather than trusting the numbers printed here.

---

## Contents

1. [Required resources](#1-required-resources)
2. [Git flow branching steps](#2-git-flow-branching-steps)
   - 2.1 [Snyk GitHub Integration](#21-snyk-github-integration)
3. [Build the apps](#3-build-the-apps)
   - 3.1 [Build the Node.js app (npm)](#31-build-the-nodejs-app-npm--exact-steps)
   - 3.2 [Build the .NET app (dotnet)](#32-build-the-net-app-dotnet--exact-steps)
   - 3.3 [Verified baselines](#33-verified-baselines-from-a-clean-build)
4. [VS Code Snyk extension](#4-vs-code-snyk-extension)
5. [Ignore settings](#5-ignore-settings)
6. [Improvements with the Ignore Approval Workflow](#6-improvements-with-the-ignore-approval-workflow)
7. [Fix issues](#7-fix-issues)
8. [Snyk-fix](#8-snyk-fix)
9. [Snyk CLI Remediation Agent](#9-snyk-cli-remediation-agent)
10. [Snyk PR Checks](#10-snyk-pr-checks)

- [Appendix A — Expected findings](#appendix-a--expected-findings)
- [Appendix B — Troubleshooting](#appendix-b--troubleshooting)
- [Appendix C — What's new since v2.0](#appendix-c--whats-new-since-v20)
- [Appendix D — Resources](#appendix-d--resources)

---

## 1. Required resources

### 1.1 Node.js — required

The repository declares its supported range in `package.json`:

```json
"engines": { "node": "22 - 26" }
```

⚠️ **That range is necessary but not sufficient.** The Angular CLI used by the frontend build
enforces a stricter floor of its own, and it fails hard rather than warning:

```
Node.js version v24.13.0 detected.
The Angular CLI requires a minimum Node.js version of v22.22.3 or v24.15.0 or v26.0.0.
```

So the version you actually need is one of:

| Line | Minimum that works |
| --- | --- |
| Node 22 | **v22.22.3** or later |
| Node 24 | **v24.15.0** or later |
| Node 26 | **v26.0.0** or later |

A version manager is strongly recommended so you can switch per project:

```bash
# macOS / Linux — nvm. `nvm install 24` resolves to the latest 24.x,
# which clears the v24.15.0 floor.
nvm install 24
nvm use 24

# macOS — Homebrew
brew install node@24

# Windows — winget
winget install OpenJS.NodeJS.LTS
```

Verify:

```bash
node --version    # expect v22.22.3+ / v24.15.0+ / v26.0.0+
npm --version
```

Two distinct failure modes, and they look nothing alike:

- **Outside `22 - 26` entirely** — `npm install` warns or fails on native modules (`sqlite3`).
- **Inside `22 - 26` but below the Angular floor** — `npm install` succeeds, then
  `npm run build` dies on the frontend with the message above. This one is confusing precisely
  because the install looked fine.

Fix the Node version before continuing — this is the single most common setup failure.

> Bumping *within* a major (e.g. v24.13.0 → v24.19.0) keeps the same native module ABI, so an
> existing `node_modules` stays valid and needs no reinstall. Crossing a major (24 → 26) changes the
> ABI: delete `node_modules` and reinstall, or `sqlite3` will fail to load at runtime.

### 1.2 .NET SDK — optional

**This** repository is Node.js and TypeScript only; nothing in §3.1 or §4–§10 needs the .NET SDK.

Install it only if you are also running the cross-stack exercises (§3.2) against the companion
`juice-shop-dotnet` repository, where the same vulnerability classes are demonstrated on a
compiled, statically typed stack. That is useful when someone in the room believes SAST is a
scripting-language problem.

```bash
# macOS — Homebrew
brew install --cask dotnet-sdk

# Ubuntu / Debian
sudo apt-get update && sudo apt-get install -y dotnet-sdk-8.0

# Windows — winget
winget install Microsoft.DotNet.SDK.8
```

Verify:

```bash
dotnet --list-sdks
dotnet --list-runtimes
```

**Any SDK from 8.0 upward works — you do not need the 8.0 SDK specifically.** The API project
targets `net8.0` but sets `<RollForward>Major</RollForward>`, so a newer SDK builds and runs it
without the .NET 8 runtime installed. Verified for this course on SDK **10.0.302** with no .NET 8
runtime present at all.

The one wrinkle is the **test** project, which targets `net10.0` on purpose — `dotnet test` hosts its
in-process `TestServer` on the system runtime, and `Microsoft.AspNetCore.Mvc.Testing` has to match
that runtime. If you want everything on genuine .NET 8, install the ASP.NET Core 8 runtime and
retarget `tests/JuiceShop.Tests/JuiceShop.Tests.csproj` to `net8.0` with `Mvc.Testing 8.0.x`.

### 1.3 Snyk CLI — required

```bash
# npm — also gives you the MCP server used in §8–§9
npm install -g snyk@latest

# Homebrew
brew install snyk

# Windows — Scoop
scoop install snyk
```

> ⚠️ **If you use nvm, prefer Homebrew (or the standalone binary) over `npm install -g`.** nvm keeps
> global npm packages **per Node version**. Install the CLI under v24.13.0, then follow §1.1 and
> `nvm use` a newer 24.x, and `snyk` vanishes:
>
> ```
> $ nvm use 24 && snyk --version
> zsh: command not found: snyk
> ```
>
> Nothing is broken — you're just looking at a different `bin` directory. Either reinstall under the
> new version (`npm install -g snyk@latest`) or install it outside nvm so it survives Node switches.
> **Your authentication is safe either way**: the token lives in `~/.config/configstore/snyk.json`,
> outside any Node version, so you do not need to re-run `snyk auth`.

Authenticate. This opens a browser; log in with SSO if your company uses it:

```bash
snyk auth
snyk --version
```

**Version floor.** Some steps below need a recent CLI:

| Feature | Minimum CLI |
| --- | --- |
| `snyk ignore create` / Ignore Approval Workflow (§5–§6) | **v1.1297.1** |
| Reachability flags (§3.5, Snyk Preview) | v1.1301.0 |
| `snyk doctor` diagnostics | v1.1306.0 |

Snyk supports CLI, IDE and CI plugin versions for 12 months. Anything older than that is
unsupported, not merely old.

**EU / other regions.** If your organization is hosted outside the default region, set the endpoint
*before* authenticating, then log in at the regional URL:

```bash
snyk config set endpoint=https://api.eu.snyk.io
# then log in at https://app.eu.snyk.io/login
```

The **Log in** button on snyk.io does *not* redirect to regional login pages. Getting this wrong
silently creates a second account and your scans land in the wrong tenant.

### 1.4 Everything else

| Tool | Why | Check |
| --- | --- | --- |
| Git 2.30+ | branching exercises (§2) | `git --version` |
| VS Code | Snyk extension (§4) | — |
| GitHub account with a fork of the repo | PR checks (§10) | — |
| Snyk account with access to an organization | all sections | <https://app.snyk.io> |
| *Optional:* Claude Code / Cursor / Codex CLI / Gemini CLI | §8–§9 | — |

### 1.5 Pre-flight check

Run this **before** the session starts. Every line must succeed:

```bash
node --version && npm --version && git --version && snyk --version
```

Check the Node version against the **Angular floor** in §1.1 — v22.22.3 / v24.15.0 / v26.0.0 — not
just against `22 - 26`. A version that satisfies `engines` but not the floor installs cleanly and then
fails the build, which is a bad thing to discover at 09:05.

Then do the slow part ahead of time (§3.1) — it is the one step you cannot recover from in a live
session:

```bash
npm install --legacy-peer-deps && npm run build && snyk test --all-projects
```

Expect `Projects tested: 2 projects`, and expect a non-zero exit code — issues are the content here.

If anything fails, `snyk doctor` (CLI v1.1306.0+) reports on auth, connectivity, proxy and
configuration in one pass.

---

## 2. Git flow branching steps

This repository follows the upstream Juice Shop model with one addition for this course:

- **`master`** is the release branch. Nobody commits to it.
- **`develop`** is the integration branch and the target for all training pull requests.
- **`ai-agent-snyk-fix-training`** is the pre-staged base for the AI remediation labs (§8–§9).
- Your work happens on short-lived feature branches, and every exercise ends in a pull request into
  `develop` so you can watch the Snyk PR check run.

> This fork uses `master`, not `main`. If you have muscle memory for `main`, expect
> `git checkout main` to fail.

### Fork and clone

```bash
# 1. Fork https://github.com/lmaeda/juice-shop in the GitHub UI.

# 2. Clone your fork
git clone https://github.com/<your-username>/juice-shop.git
cd juice-shop

# 3. Add the training repo as upstream so you can pull updates
git remote add upstream https://github.com/lmaeda/juice-shop.git
git fetch upstream

# 4. Confirm both remotes
git remote -v
```

### Create your training branch

```bash
git checkout develop
git pull upstream develop

# One branch per exercise. Naming convention for this course:
#   snyk-training/<your-initials>/<exercise>
git checkout -b snyk-training/lm/01-first-scan
```

For the AI labs in §8–§9, branch from the pre-staged branch instead:

```bash
git fetch upstream
git checkout -b snyk-training/lm/08-agentic-fix upstream/ai-agent-snyk-fix-training
```

### Work, scan, commit

```bash
# make a change, then always scan before committing
snyk test --all-projects
snyk code test

git add -A
git commit -m "fix(sca): bump sanitize-html to a non-vulnerable version"
```

### Push and open the PR

```bash
git push -u origin snyk-training/lm/01-first-scan
```

Open the pull request **into `develop`** (not `master`). The Snyk PR check runs here — that's §10.

### Branch reference

| Branch | Purpose | Commit to it? |
| --- | --- | --- |
| `master` | release / stable | No |
| `develop` | integration target for all training PRs | No — open a PR into it |
| `snyk-training/<initials>/<exercise>` | your work | Yes |
| `ai-agent-snyk-fix-training` | pre-staged base for the AI remediation labs (§8–§9) | Branch from it |

### Resetting between exercises

The intentionally vulnerable code *is* the training content, so revert rather than fix when you want
a clean start:

```bash
git checkout develop
git branch -D snyk-training/lm/01-first-scan   # discard the exercise branch
git pull upstream develop
```

---

## 2.1 Snyk GitHub Integration

Before PR checks (§10) can work, your repository has to be imported into Snyk through the GitHub
integration. Your Snyk **org admin** normally does the integration setup once per org; the per-repo
import is often something developers can do themselves.

### What the integration gives you

| Capability | What you see as a developer |
| --- | --- |
| Repository import | Your repo appears as a Snyk **Target**, with one Snyk **Project** per manifest / code root |
| Recurring tests | Snyk re-scans the monitored branch daily or weekly and can notify you on new issues |
| PR checks | A Snyk status check on every pull request (§10) |
| Fix PRs | Snyk opens PRs that bump vulnerable dependencies (§7.1) |
| Inline PR comments | Snyk Code findings annotated on the changed lines, max 10 per PR |

### Steps

1. **Connect GitHub** *(admin, once per Snyk org)*
   Snyk Web UI → **Integrations** → **GitHub** (or **GitHub Enterprise**). Prefer the **GitHub App**
   where possible: fine-grained permissions and commit signing, versus a broad OAuth grant.
   Authorize the org that owns your fork.

   > Since April 2026, Snyk Agent Fix in the PR on GitHub requires a **dedicated account
   > authenticated with a PAT**. This is a common silent blocker — if inline fixes never appear,
   > check this before debugging anything else.

2. **Import the repository**
   **Projects** → **Add Project** → **GitHub** → select `juice-shop` → **Add selected repositories**.

3. **Confirm what got imported.** For this repo you should end up with Projects for at least:

   | Project | Product |
   | --- | --- |
   | `package.json` | Snyk Open Source (npm) — server dependencies |
   | `frontend/package.json` | Snyk Open Source (npm) — Angular dependencies |
   | Code analysis (whole repo) | Snyk Code — the `.ts` flaws |
   | `Dockerfile` | Snyk Container — base image prediction |
   | `infrastructure/Dockerfile` | Snyk Container |
   | `terraform/*.tf`, `infrastructure/terraform/*.tf` | Snyk IaC |

   If Projects are missing, a product may be disabled for your org, or those files were excluded at
   import. That's an admin question, not a bug.

4. **Check the monitored branch.** Project settings → confirm the monitored branch is `develop` (or
   `master`, matching your team's convention). The monitored branch is what recurring tests and every
   reporting number are based on, and it is the setting people most often get wrong.

5. **Set your personal notifications.** Account settings → **Notifications**. Your admin sets org
   defaults; you override for yourself.

   > As of 2026, new-vulnerability notification emails are **off by default**. If you expect email
   > and get none, alerting isn't broken — it was never on.

**Docs:** <https://docs.snyk.io/developer-tools/integrations/scm-integrations/organization-level-integrations/github>

---

## 3. Build the apps

You need a working build so that (a) Snyk Open Source can resolve the full dependency tree, and
(b) you can prove your fixes didn't break anything.

### 3.1 Build the Node.js app (npm) — exact steps

**Snyk Open Source cannot scan this repository until `node_modules` exists.** There is no lockfile:
both `.npmrc` files set `package-lock=false`, so `snyk test` has nothing to resolve a dependency tree
from and fails outright rather than reporting zero issues:

```
ERROR   Unspecified Error (SNYK-CLI-0000)
        Failed to get dependencies for all 2 potential projects.
        .../package.json:          Missing node_modules folder: we can't test without
                                   dependencies. Please run 'npm install' first.
        .../frontend/package.json: Missing node_modules folder: ...
```

Both manifests need their own `node_modules`. Run these four commands from the repository root, in
order:

```bash
# 0. Confirm the Node version FIRST — see §1.1. Below v24.15.0 step 2 fails.
node --version

# 1. Install root + frontend dependencies.
#    --legacy-peer-deps is REQUIRED — see the note below.
npm install --legacy-peer-deps

# 2. Build the Angular frontend and compile the TypeScript server
npm run build

# 3. Run it
npm start            # serves on http://localhost:3000
```

> #### Why `--legacy-peer-deps` is mandatory
>
> A plain `npm install` **fails** on this repo. The root install succeeds, then the `postinstall`
> hook (`cd frontend && npm install && … && npm run build:frontend`) dies with `ERESOLVE`:
> `angularx-qrcode@21.0.5` declares an `@angular/core@^21.0.0` peer while the frontend is on
> `@angular/core@^22.0.1`. `frontend/package.json` *has* `overrides` intended to reconcile this, but
> npm 11.x still refuses to resolve the tree. The error text blames `rxjs@7.8.2` against a peer range
> of `^6.5.3 || ^7.4.0` — **that is a red herring**, since 7.8.2 satisfies that range. Don't spend
> time on the rxjs line.
>
> `--legacy-peer-deps` at the root is enough on its own: npm exports it to child processes through
> `npm_config_legacy_peer_deps`, so the nested frontend install inherits it. You do **not** need to
> install the frontend separately.

**Timing.** Budget **5–10 minutes** on a cold run (empty npm cache) and do it before the session —
`postinstall` installs and builds the whole Angular frontend. On a warm cache it is much faster:
measured ~**72 s** for step 1 and ~**13 s** for step 2 on an Apple-silicon laptop. If step 1 returns in
seconds, it did nothing — check the verification block below.

> **Step 1 already runs step 2 for you.** `postinstall` ends with
> `npm run build:frontend && (npm run --silent build:server || cd .)`, so a successful install leaves
> `build/` and `frontend/dist/` populated. Run `npm run build` anyway: the `|| cd .` in that hook
> **swallows a server compile failure**, so a broken `tsc` looks like a clean install. Step 2 is how
> you find out.

**`npm warn allow-scripts` is expected — do not chase it.** npm 11.17+ gates *dependencies'* install
scripts behind an approval list, and you will see warnings like:

```
npm warn allow-scripts 2 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   cypress@15.19.0 (postinstall: node dist/index.js --exec install)
npm warn allow-scripts   libxmljs2@0.37.0 (install: prebuild-install || node-gyp rebuild)
```

`package.json` has an `allowScripts` block covering `sqlite3`, `esbuild` and `cypress`, but versions
drift out of it. This does **not** affect the build, the server, or any Snyk scan. The one real
consequence is that a blocked `cypress` postinstall means no Cypress binary, so only
`npm run test:e2e` is affected — nothing in this course. Approve them if you want them:

```bash
npm approve-scripts --allow-scripts-pending    # review interactively
```

**Verify each step actually worked.** Success is not "no red text scrolled past":

```bash
# 1. Both dependency trees present — neither should be missing or empty
ls node_modules | wc -l            # ~846 top-level entries
ls frontend/node_modules | wc -l   # ~496 top-level entries

# 2. Build artifacts present
ls build/app.js                    # compiled server (tsc output)
ls frontend/dist/frontend/         # Angular production bundle

# 3. The server answers
curl -s http://localhost:3000/rest/admin/application-version
# => {"version":"20.2.0-SNAPSHOT"}
```

**Then confirm Snyk resolves both manifests** — this is the whole point of the build:

```bash
snyk test --all-projects
```

You want to see `Projects tested: 2 projects`. Anything less means one `node_modules` is missing.

> **If you see `3 potential projects` and one failure**, that third one is `build/package.json` — the
> `tsc` output directory carries a copy of the manifest but has no `node_modules` of its own. It is
> harmless noise. Silence it with `snyk test --all-projects --exclude=build`. The same thing happens
> for *any* stray `package.json` under the repo root, so never park a `node_modules` backup there —
> `--all-projects` will try to test every package inside it.

> ⚠️ **`snyk test` exits non-zero when it finds issues — that is success, not failure.** On this
> deliberately vulnerable repo a zero exit code would mean something went wrong. Judge the outcome by
> the summary block, not by `$?`.

Other useful targets, all real scripts in `package.json`:

```bash
npm run build          # build:frontend + build:server
npm run build:server   # tsc only — fast re-check after editing a .ts file
npm run serve:dev      # watch mode, backend + frontend concurrently
npm run test:server    # server unit tests (Node built-in test runner)
npm run test:api       # API integration tests (Supertest)
npm run test:frontend  # Angular unit tests (Vitest)
npm run lint
npm run rsn            # Refactoring Safety Net — required after challenge-code edits
npm run sbom           # CycloneDX SBOM → bom.json, useful with `snyk sbom test`
```

> When iterating on a Snyk Code fix in `routes/` or `lib/`, run `npm run build:server` rather than
> the full `npm run build`. It skips the Angular bundle and takes seconds.

### 3.2 Build the .NET app (dotnet) — exact steps

**Optional**, and only for the cross-stack exercises. This is a **separate repository**
(`lmaeda/juice-shop-dotnet`), not a directory of this one. Clone it as a sibling:

```bash
# from the parent directory of your juice-shop clone
git clone https://github.com/lmaeda/juice-shop-dotnet.git
cd juice-shop-dotnet/dotnet          # note: the solution lives in the dotnet/ subdirectory
```

The solution is `JuiceShop.sln`, with two projects — `src/JuiceShop.Api` (`net8.0`) and
`tests/JuiceShop.Tests` (`net10.0`, see §1.2).

```bash
# 1. Restore NuGet packages (Snyk needs this to resolve the dependency graph)
dotnet restore JuiceShop.sln

# 2. Build
dotnet build JuiceShop.sln --no-restore

# 3. Test — 8 xUnit tests, functional + exploitation
dotnet test JuiceShop.sln --no-build

# 4. Run
dotnet run --project src/JuiceShop.Api
```

**Expect `NU1902` / `NU1903` warnings on restore and build.** They are the point — NuGet is flagging
the intentionally outdated packages (`Newtonsoft.Json 12.0.3`, `RestSharp 106.11.7`,
`SharpZipLib 1.3.2`, `System.IdentityModel.Tokens.Jwt 6.24.0`, and the transitive
`SQLitePCLRaw.lib.e_sqlite3 2.1.6`). `NuGetAudit` is set to `false` in `JuiceShop.Api.csproj` so they
don't turn the build red; the packages stay vulnerable and Snyk still reports them.

Success looks like:

```
JuiceShop.Api   -> src/JuiceShop.Api/bin/Debug/net8.0/JuiceShop.Api.dll
JuiceShop.Tests -> tests/JuiceShop.Tests/bin/Debug/net10.0/JuiceShop.Tests.dll
Build succeeded.  7 Warning(s)  0 Error(s)

Passed!  Failed: 0, Passed: 8, Skipped: 0, Total: 8
```

Then scan it — run these **from `juice-shop-dotnet/dotnet/`**, not from the Node repo:

```bash
snyk test --all-projects     # Open Source: 2 projects
snyk code test               # Snyk Code (SAST)
```

> **`dotnet restore` is the .NET equivalent of `npm install` for Snyk's purposes.** Skip it and
> `snyk test` fails the same way the Node repo does without `node_modules` — Snyk reads the restored
> `project.assets.json`, not the `.csproj` alone.

### 3.3 Verified baselines from a clean build

Captured on the `ai-agent-snyk-fix-training` branch with Node **v24.19.0**, npm **11.17.0**, Snyk CLI
**1.1306.2**, .NET SDK **10.0.302**. Treat these as a shape-check that your build is complete, not as
targets — counts drift (see the version-drift note in the preamble, and Appendix A). The useful signal
is **"2 projects tested" plus a three-digit Node SCA count**, not the exact numbers.

| Stack | Command | Result |
| --- | --- | --- |
| Node | `snyk test --all-projects` | **2 projects**, 103 security + 2 license issues |
| ↳ `package.json` | | 714 deps · **95 issues** / 158 vulnerable paths |
| ↳ `frontend/package.json` | | 463 deps · **8 security + 2 license** issues |
| Node | `snyk code test` | **293 issues** — 26 high, 11 medium, 256 low |
| .NET | `snyk test --all-projects` | **2 projects**, 11 issues — 3 high, 8 medium |
| .NET | `snyk code test` | **14 issues** — 11 high, 2 medium, 1 low |

> The `snyk code test` count is **source-only**. `build/`, `frontend/dist/` and `node_modules/` are
> all gitignored and Snyk Code honours `.gitignore`, so building first does not inflate your SAST
> numbers. Worth saying out loud — people assume it does.

### 3.4 What makes this repo a good target

| Surface | Why it matters for this course |
| --- | --- |
| `package.json` | Deliberately pinned ancient packages — `express-jwt@0.1.3`, `jsonwebtoken@0.4.0`, `sanitize-html@1.4.2` |
| `frontend/package.json` | Its own set of stale Angular-era dependencies |
| `routes/`, `lib/` | The TypeScript flaws Snyk Code traces with data flow |
| `Dockerfile` | Hardened: `node:24` builder → `gcr.io/distroless/nodejs24-debian13` runtime, `USER 65532` |
| `infrastructure/Dockerfile` | Multi-stage on Node 20: `-bookworm` builder, `-bookworm-slim` runtime, `USER node`. The better Container demo — an older base, not an insecure Dockerfile |
| `terraform/`, `infrastructure/terraform/` | Intentionally insecure IaC, including a committed RSA private key |
| No `.snyk` file, no Snyk CI workflow | Clean slate — you create both during the labs |

### 3.5 First scan — the baseline

Run all scan types and **keep the output**; you'll compare against it after fixing. Everything here
assumes §3.1 completed — without `node_modules` the Open Source test fails rather than reporting zero.

```bash
# from the repository root

# Open Source (SCA) — every manifest in the repo
snyk test --all-projects

# Code (SAST)
snyk code test

# Infrastructure as code
snyk iac test terraform/
snyk iac test infrastructure/terraform/

# Container — the base image of the image that actually SHIPS.
# infrastructure/Dockerfile is multi-stage: node:20.19.2-bookworm is only the
# throwaway builder; the runtime stage is built on -slim. Scan the runtime one.
snyk container test node:20.19.2-bookworm-slim --file=infrastructure/Dockerfile

# Secrets (GA, July 2026) — must be enabled for your org
snyk secrets test

# Publish a snapshot to the Snyk UI
snyk monitor --all-projects
```

Flags worth memorising, with the scope caveats that trip people up:

| Flag | Effect | Caveat |
| --- | --- | --- |
| `--all-projects` | scan every manifest found, recursively | pair with `--detection-depth` / `--exclude` |
| `--severity-threshold=high` | non-zero exit only on high + critical | `test` only, not `monitor` |
| `--show-vulnerable-paths=all` | every dependency path, not just one | not supported with `--json-file-output` |
| `--sarif-file-output=out.sarif` | machine-readable output | returns **no results for Open Source** tests |
| `--dev` | include dev dependencies (excluded by default) | — |
| `--include-ignores` | show ignored findings | `snyk code test` and `snyk secrets test` **only** |
| `--ignore-policy` | scan as if `.snyk` didn't exist | verifying an ignore |
| `--reachability=true` | is the vulnerable path actually called? | **Snyk Preview**, CLI ≥ v1.1301.0 — returns a new findings schema that can break JSON automations |

> `snyk test` is the **gate** — it exits non-zero and can fail a build. `snyk monitor` is the
> **record** — it publishes a snapshot Snyk tracks and alerts on. In a pipeline, run `monitor`
> *after* a successful `test`, so the monitored snapshot only ever reflects green builds.

Expected results are in [Appendix A](#appendix-a--expected-findings).

---

## 4. VS Code Snyk extension

The extension is the shortest feedback loop you have — findings appear as you save, not after you
push.

### 4.1 Install

1. VS Code → **Extensions** (`Ctrl/Cmd+Shift+X`)
2. Search **Snyk Security** (publisher: **Snyk**)
3. **Install**

Or:

```bash
code --install-extension snyk-security.snyk-vulnerability-scanner
```

The extension downloads and bundles its own Snyk CLI, so a separate CLI install isn't required *just*
for the IDE — but you want one anyway for §5–§9.

### 4.2 Authenticate

Open the Snyk panel in the Activity Bar, then either:

- **Connect IDE to Snyk** — opens a browser, logs in with SSO if applicable. Preferred.
- **Use a token** — paste an API token from account settings, or one your admin issued. For machines
  without a browser.

Then confirm the right organization: **Settings → Extensions → Snyk Security → Organization**. If
you belong to several orgs and pick the wrong one, your scans land in the wrong place *and*
org-level ignores won't apply.

> **Unified IDE configuration** went GA in July 2026, so settings are now consistent across VS Code,
> IntelliJ, Visual Studio and Eclipse. If a colleague's screenshots don't match yours, one of you is
> on a pre-July build. Note also that **Visual Studio 2026 is not supported**.

### 4.3 Configure for this repo

| Setting | Value | Why |
| --- | --- | --- |
| Snyk Open Source | ✅ enabled | npm findings in both manifests |
| Snyk Code | ✅ enabled | SAST findings in `.ts` |
| Snyk IaC | ✅ enabled | `terraform/` and `infrastructure/terraform/` |
| Scan mode | on save (default) | fastest loop |
| Trusted folders | add your clone | **required** before the first scan |

The first time you open the repo, Snyk asks you to **trust the folder**. Accept — it will not scan an
untrusted folder. This is the single most common "it isn't working" report.

### 4.4 Run a scan and read a finding

Open `routes/login.ts` and find the SQL injection at **line 34**:

```ts
models.sequelize.query(
  `SELECT * FROM Users WHERE email = '${req.body.email || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`,
  { model: UserModel, plain: true }
)
```

In the Snyk panel, expand **Code Security** → select **SQL Injection**. The issue card shows:

- **Data flow** — the traced path from the untrusted source (`req.body.email`) to the sink (the raw
  SQL string). Click any step to jump to that line. This is a traced path, not a pattern match, and
  it's the most convincing thing in the IDE for a sceptical developer.
- **Details** — CWE-89, plus links to the Snyk Vulnerability Database and the CWE record.
- **Remediation advice** — the concrete change to make.

> **Changed in August 2026:** Snyk Code **fix examples were removed** from the Web UI *and* from
> SARIF output on 17 August 2026. If your notes mention "how often this is fixed one way vs
> another," that panel is gone. Anything parsing fix examples out of SARIF is now broken.

Now do the same for a dependency finding: open `package.json` and inspect the `sanitize-html@1.4.2`
or `express-jwt@0.1.3` finding. Make the contrast explicit:

- **SCA tells you which version to move to.**
- **SAST tells you which line to change.**

Same tool, different kind of answer, different kind of work.

### 4.5 AI-assisted fixes in the IDE — Snyk Agent Fix

Where enabled, Snyk Code findings offer a generated fix you can apply inline. Snyk Agent Fix
(formerly DeepCode AI Fix) generates candidate fixes, re-scans each one to confirm the vulnerability
is gone and nothing new appeared, and retries when a candidate fails verification. Rather than
fine-tuning, it enriches the prompt from a Snyk database of **more than 35,000 expert-written
fixes**.

- **Status:** core Agent Fix is **GA**, re-architected onto Claude models and fully rolled out in
  late May 2026. It now covers **all Snyk Code languages and rules** — retire any older
  "only works for language X" caveats you may have heard.
- **Enablement:** **Settings → Snyk Agent Fix**, at **Group *or* Organization** level. Somebody has
  to turn it on before it appears for you. Snyk Learn describes it as an Enterprise-tier feature;
  the product docs don't state a tier, so confirm with your admin rather than assuming.
- **Not supported** with Snyk Code Local Engine — which is itself deprecated.
- **Data:** Snyk does not use customer code to train the underlying models, add to datasets, or
  improve performance.
- **The discipline:** read the diff, every time. The verification loop checks for *the
  vulnerability*. It does not check whether the change makes sense in your codebase. You own the
  commit.

**Docs:** <https://docs.snyk.io/scan-fix-and-prevent/scan-with-snyk/snyk-code/manage-code-vulnerabilities/fix-code-vulnerabilities-automatically>

---

## 5. Ignore settings

Ignoring is sometimes the right call — a genuine false positive, or an issue with no fix available.
It should never be the default response.

**Best practice order: fix → patch → remove or replace the dependency → ignore.**

Snyk has **two distinct ignore mechanisms**, and mixing them up is the single largest source of
"but I ignored it":

|  | Snyk Open Source / Container / IaC | Snyk Code (SAST) and Snyk Secrets |
| --- | --- | --- |
| Mechanism | `.snyk` policy file, Web UI, or Security Policies | **Consistent Ignores** — asset-scoped |
| CLI command | `snyk ignore --id=…` | `snyk ignore create` **(EA)** |
| Stored where | a committed file in your repo | the Snyk backend, tied to the finding's identity |
| Travels across branches? | only where the file is checked out | yes — all branches and all surfaces |
| Audit trail | Git history | the ignore record in Snyk |

> ⚠️ **`.snyk` files do not work for Snyk Code.** Use `snyk ignore create` or the Web UI instead.
> Note also that org **Security Policies** are a *different* feature that applies to Open Source and
> Container — not to Snyk Code. Two traps, one sentence apart.

### 5.1 Open Source ignores — the `.snyk` file

There is no `.snyk` file in this repo yet, so your first ignore creates one. Take an issue ID from
your scan output, then:

```bash
snyk ignore \
  --id='SNYK-JS-SANITIZEHTML-1070786' \
  --expiry='2026-11-30' \
  --reason='Only used on server-rendered admin pages behind auth; tracked in JIRA-1234'
```

This creates or updates `.snyk` in the working directory:

```yaml
version: v1.19.0
ignore:
  SNYK-JS-SANITIZEHTML-1070786:
    - '*':
        reason: >-
          Only used on server-rendered admin pages behind auth; tracked in JIRA-1234
        expires: '2026-11-30T00:00:00.000Z'
        created: '2026-08-04T00:00:00.000Z'
```

Options:

| Option | Notes |
| --- | --- |
| `--id=` | **required.** The Snyk issue ID from `snyk test` output |
| `--expiry=` | `YYYY-MM-DD`. **Defaults to 30 days** if omitted — people assume permanence |
| `--reason=` | optional in the tool, mandatory in practice. Write one the next person can act on |
| `--path=` | narrow the ignore to one dependency path |
| `--policy-path=` | point at a `.snyk` file elsewhere |

> **The mistake everyone makes once:** not committing `.snyk`. The ignore then exists only on your
> machine and CI still fails. Do this deliberately once so you recognise it later.

Verify it took effect:

```bash
snyk policy                              # print the effective .snyk policy
snyk test --all-projects
snyk test --all-projects --ignore-policy # scan as if no policy existed
```

### 5.2 Snyk Code ignores — Consistent Ignores **(GA)**

A Snyk Code ignore is **asset-scoped**: it attaches to the finding's identity, not to a
project-plus-branch pair. Create it once and it's respected in the Web UI, your IDE, the CLI, your CI
pipeline and PR checks — across every branch.

Consistent Ignores has been **enabled by default for all new customers since 19 June 2025**. The
API and Group-level capabilities are Enterprise-only.

Create one from the CLI **(EA, requires CLI ≥ v1.1297.1)**:

```bash
# 1. Get the finding ID
snyk code test --json > snyk-code.json
# the ID lives at runs[].results[].fingerprints."snyk/assets/finding/v1"

# 2. Create the ignore — interactive by default
snyk ignore create

# 3. Or fully non-interactive, which is what you need in CI
snyk ignore create \
  --finding-id='<finding-id>' \
  --ignore-type='temporary-ignore' \
  --expiration='2026-10-31' \
  --reason='Endpoint is internal-only; hardening tracked in JIRA-5678'
```

The complete flag set is `--finding-id`, `--ignore-type`, `--reason`, `--expiration`, `--org` and
`--remote-repo-url`. The first three are required; `--expiration` accepts a date or `never` and is
required in non-interactive mode. `--remote-repo-url` is auto-detected from `.git`.

Ignore types are exactly `not-vulnerable`, `wont-fix` and `temporary-ignore`.

Scope limits worth knowing: this covers findings from `snyk code test` in the CLI and IDE. It does
not apply to SCM-imported projects or CLI Upload projects.

Or from the Web UI — open the issue card → **Ignore** → choose:

| Choice | Use when |
| --- | --- |
| **Not vulnerable** | the path is genuinely not exploitable |
| **Ignore temporarily** | accepted risk, with a review date |
| ↳ *Until fix is available* | no fix exists yet — resurfaces automatically the moment one does |
| **Won't fix** | accepted permanently, with justification |

*Until fix is available* is the most defensible ignore a developer can create. It's checked by
default when no fix exists.

Two behaviours to expect:

- After creating, modifying or deleting an ignore, the **Project must be retested** before the issue
  status updates. A banner on the Project page tells you. Recurring tests run nightly or weekly; you
  can also retest on demand.
- Ignored findings are **hidden by default** in reports and CLI output. Show them with
  `snyk code test --include-ignores`.

### 5.3 Viewing what's ignored

```bash
snyk code test --include-ignores          # Snyk Code
snyk secrets test --include-ignores       # Snyk Secrets
snyk policy                               # effective .snyk policy
snyk test --all-projects --ignore-policy  # ignore .snyk entirely
```

### 5.4 Exercise

1. Run `snyk code test`. Pick the **Open Redirect** finding in `routes/redirect.ts`.
2. Ignore it as `temporary-ignore`, 30 days, with a reason a reviewer could actually act on.
3. Re-run `snyk code test`. Confirm the count dropped by one.
4. Re-run with `--include-ignores`. Confirm it reappears, marked ignored.
5. Now do the same for an Open Source finding with `snyk ignore`, and inspect the resulting `.snyk`.
6. **Discuss:** which of these two could your security team audit? Which one could a developer add
   quietly without anyone noticing? That question is what §6 is about.

**Docs:** <https://docs.snyk.io/scan-fix-and-prevent/fix/prioritize-issues-for-fixing/ignore-issues>

---

## 6. Improvements with the Ignore Approval Workflow

### 6.1 The problem with §5

Everything in §5 is developer-unilateral. Most orgs end up with one of two bad outcomes:

- **Ignores open to everyone.** Any collaborator can suppress a critical finding instantly. Fast,
  but AppSec has no control and finds out later, if ever.
- **Ignores admin-only.** Nothing gets suppressed without a ticket. Safe, but a developer blocked by
  a false positive now waits on a human queue — so the pressure moves to bypassing the check
  entirely.

There is no built-in "request and wait for approval" setting on the org ignore policy page. Some
teams build one by hand: restrict UI ignores to admins, and require developers to add `.snyk`
entries through a pull request that security approves. That works for Open Source and gives you Git
history as the audit trail — but it does nothing for Snyk Code, because `.snyk` doesn't apply there.
**That gap is what the Ignore Approval Workflow fills.**

### 6.2 What the workflow adds

A maker/checker step, captured where the developer already is:

1. Developer runs `snyk code test` from the IDE or CLI. Findings are stored in Snyk.
2. Developer requests an ignore **from the IDE, the CLI, or the API**, with a reason.
3. The request lands in the review queue as **PENDING**.
4. A reviewer — AppSec, or a security champion — approves or rejects it.
5. On approval, the ignore applies across CLI, IDE and SCM at the next re-scan, on every branch,
   because it is a Consistent Ignore.

Request states are exactly `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` and `NOT_REQUIRED` — the
last for orgs not using approval workflows at all.

### 6.3 Why it's better

|  | Plain ignores | Ignore Approval Workflow |
| --- | --- | --- |
| Who decides | whoever holds the permission | developer proposes, AppSec decides |
| Developer wait | none, or a ticket queue | request without leaving the IDE |
| Justification | optional free text | required with the request |
| Audit trail | UI event log or Git history | recorded request + reviewer decision |
| Covers Snyk Code | `.snyk` doesn't apply | yes — this is its scope |
| Consistency | varies by surface | one ignore, respected everywhere |

### 6.4 Permissions

| Action | Requirement |
| --- | --- |
| Request an ignore | minimum **Collaborator** role, or the granular permission `org.project.ignore.create` |
| Review (approve / reject) | the granular permission `org.policy.ignore.review` |

In the Roles UI these appear as *Create and Edit Ignore requests* and *Review Ignore Requests*.

### 6.5 Prerequisites and honest caveats

**Prerequisites**

- **Snyk Code Consistent Ignores** enabled first — this is a hard dependency
- **Snyk CLI v1.1297.1 or later**, plus a supported IDE plugin version
- Enablement by Snyk for your Group, then by your org admin per Org

**Caveats — say these out loud in training**

- **Scope is Snyk Code and Snyk Secrets. Not Open Source.** Open Source ignores still go through
  `.snyk`, the Web UI, or Security Policies.
- **Release stage is genuinely ambiguous right now.** The Snyk release note badges the workflow as
  **generally available**, while the CLI documentation still labels the `snyk ignore create` path
  **Early Access**. The Open Source extension is the part that is clearly not GA. Ask your account
  team what applies to your tenant rather than quoting either page.
- **Requesting from PR comments is not supported.** Requests come from the CLI, IDE or API. The
  Snyk inline PR comment only deep-links into the Snyk UI.
- **Reviewers may not be able to see the code.** If a finding was reported from a developer
  workstation and never reached a monitored branch, Snyk has no source to render — so the reviewer
  sees the finding metadata but not the source-to-sink data flow. Snyk's data processing terms
  prevent storing customer source code for this purpose.
  **Practical consequence: for anything non-trivial, push your branch first.** Otherwise you're
  asking a reviewer to accept risk on a finding they cannot inspect, and they'll reasonably reject
  it.
- **All reviewers are notified for every request** — requests are not assigned to an individual.
  Agree a rota inside your team, or the queue becomes nobody's job.
- Legacy DeepCode inline ignores do not carry over; migrate them to the standard ignore system.

### 6.6 Exercise *(only if enabled for your org)*

1. From your IDE, request an ignore on the **code injection** finding in `routes/b2bOrder.ts` — the
   `vm.runInContext('safeEval(orderLinesData)', …)` on line 23, where `safeEval` is `notevil`'s
   `eval`. There is no literal `eval(` in that file, so don't describe it that way to the room.
   Write a reason a reviewer could actually act on.
2. As a reviewer, open the review queue. Note what information you have — and what you don't.
3. Reject it and write the reason. Discuss what the developer should do next.
4. Push the branch, re-request, and compare how much more the reviewer can now see.

**Docs:** <https://docs.snyk.io/scan-fix-and-prevent/fix/prioritize-issues-for-fixing/ignore-issues/consistent-ignores-for-snyk-code>

---

## 7. Fix issues

Two very different jobs. Do them in this order — dependency upgrades are cheap and often clear
several findings at once.

### 7.1 Fix Open Source findings (SCA)

Snyk always recommends the **smallest** upgrade that resolves the vulnerability. For a *transitive*
vulnerability it calculates the minimum bump to the **direct** dependency that reaches a clean
transitive version. Most reported CVEs live transitively, in packages nobody on your team chose.

The pinned ancients in `package.json` are the highest-value targets:

```jsonc
// package.json — before
"express-jwt": "0.1.3",        // exact pin, no caret — deliberately ancient
"jsonwebtoken": "0.4.0",       // exact pin
"sanitize-html": "1.4.2",      // exact pin — known XSS filter bypass
"js-yaml": "^3.14.0",          // old major, code execution via load()
"helmet": "^4.6.0"             // five majors behind
```

Bump one, then prove it:

```bash
npm install sanitize-html@latest
npm run build
npm run test:server && npm run test:api
snyk test --all-projects
```

Or let npm do the arithmetic first, then verify with Snyk:

```bash
npm audit fix
snyk test --all-projects
```

> **Expect breakage, and treat it as the lesson.** `express-jwt@0.1.3 → 8.x` and
> `jsonwebtoken@0.4.0 → 9.x` are enormous jumps with real API changes; `lib/insecurity.ts` will stop
> compiling. That is the single most valuable five minutes in this course:
> **a security upgrade is a code change, and the build and the tests are how you find out.**
> Don't skip ahead to the fix — let it break first.

**When no upgrade exists.** Options in order of preference: remove the dependency; replace it with a
maintained alternative (check <https://snyk.io/advisor>); or ignore it with a review date (§5).
`snyk protect` — the old patch mechanism — is deprecated; don't teach it.

**Snyk Fix PRs.** From the Snyk UI, an issue card's **Fix this vulnerability** button opens a PR
against your repo with the required upgrade, and several issues can be bundled into one. Each PR
carries a **merge advice badge** — Snyk's confidence that merging won't break you. Review the diff;
the badge is advice, not a guarantee. Two 2026 additions: obsolete Fix PRs now **auto-close** by
default, and **breakability (merge risk) analysis** is available in EA via Snyk Preview.

### 7.2 Fix Snyk Code findings (SAST)

Dependency bumps don't touch these. Work from highest-value down. Every one of these is a real file
in this repository:

| # | File | Flaw | The fix, in one line |
| --- | --- | --- | --- |
| 1 | `routes/login.ts:34` | SQL Injection (CWE-89) | parameterize — `replacements` / bind params; never interpolate into SQL |
| 2 | `routes/search.ts:23` | UNION SQL Injection (CWE-89) | same — use the Sequelize query builder or bound parameters |
| 3 | `routes/trackOrder.ts:18` | NoSQL Injection via `$where` (CWE-943) | query by field equality; never build `$where` from input |
| 4 | `lib/xml.ts:35` | XXE (CWE-611) | drop `XML_PARSE_NOENT` and `XML_PARSE_DTDLOAD`; don't register FS input providers |
| 5 | `routes/fileUpload.ts:31-34` | Zip Slip (CWE-22) | resolve each entry and assert it is *under* the target dir — the `includes()` check on line 33 is not containment |
| 6 | `routes/fileServer.ts:33` | Path Traversal (CWE-22) | canonicalize with `path.resolve`, then assert the result starts with the allowed root |
| 7 | `routes/profileImageUrlUpload.ts:24` | SSRF (CWE-918) | allow-list schemes and hosts; block link-local and private ranges |
| 8 | `routes/b2bOrder.ts:21-23` | Unsafe eval / sandbox escape (CWE-94) | parse the payload; delete the `notevil` `safeEval` import and the `vm` context |
| 9 | `routes/userProfile.ts:65` | `eval` on user input → SSTI (CWE-94) | never `eval` a username; and drop `unsafe-eval` from the CSP at line 91 |
| 10 | `frontend/src/app/search-result/search-result.component.ts:144` | DOM XSS (CWE-79) | remove `bypassSecurityTrustHtml`; bind as text |
| 11 | `routes/redirect.ts:19` + `lib/insecurity.ts:136` | Open Redirect (CWE-601) | allow-list exact targets; `allowed \|\| url.includes(allowedUrl)` is not an allow-list |
| 12 | `lib/insecurity.ts:41` | Weak hash — MD5 for passwords (CWE-327) | use a password KDF (argon2 / bcrypt / scrypt), salted |
| 13 | `lib/insecurity.ts:21` | Hardcoded RSA private key (CWE-798) | load from config or a secret store; rotate the key |
| 14 | `routes/captcha.ts:14` | Insecure randomness (CWE-338) | `crypto.randomInt` |
| 15 | `terraform/networking.tf:171` | Private key committed in IaC (CWE-798) | reference a secret; never inline a key in Terraform |
| 16 | `routes/basket.ts:18` | **IDOR / BOLA (CWE-639)** | authorize the **object**, not just the caller |

> **#16 is the most important line in this table, and it's the one SAST doesn't report.** No static
> analyser reliably infers your application's authorization model, so the IDOR in `routes/basket.ts`
> — a basket fetched by `req.params.id` with no ownership check — never appears in
> `snyk code test` output. `server.ts:389` even carries the deliberately commented-out
> `security.isAuthorized()` line. Tooling has boundaries. Code review and threat modelling still
> matter, and a trainer who says so is worth trusting on everything else.

The loop for each fix:

```bash
snyk code test                          # confirm it's reported
# ...edit...
npm run build
npm run test:server && npm run test:api
snyk code test                          # confirm the finding is gone
```

> Several tests in `test/api/` and `test/server/` assert that the vulnerability **works** — they
> exercise the Juice Shop challenges. When you fix the code, those tests should start failing. That
> is the signal your fix is real. Update the test to assert the secure behaviour instead.
>
> If you touch challenge-related code, the Refactoring Safety Net must still pass: `npm run rsn`.

`data/static/codefixes/` holds upstream's curated before/after pairs — 35 files ending `_correct.*`
are the reviewed fixed variants, covering the SQLi, NoSQLi, XSS and redirect challenges (and some
IaC, Solidity and YAML ones). There is no codefix set for the `b2bOrder.ts` RCE. Useful for
comparing your fix against a reviewed one.

---

## 8. Snyk-fix

`/snyk-fix` is a **Snyk Studio command directive** — a slash command installed into your AI coding
assistant that runs an end-to-end remediation loop: scan → fix with Snyk's security context in the
prompt → re-scan to verify → repeat.

It is **not** a Snyk CLI subcommand. There is no `snyk fix`.

### 8.1 What Snyk Studio is

Snyk Studio connects your AI coding assistant to Snyk **locally**. Snyk describes it as five
interconnected layers, serving two use cases — *Secure at Inception* and *Intelligent Remediation*.
The parts you actually touch:

| Part | What it does |
| --- | --- |
| **Snyk MCP Server** | runs locally via the Snyk CLI; exposes scan tools your agent can call |
| **Guardrail directives** | make the agent scan generated code *before* it declares a task done. Implemented as **hooks** (deterministic) or **rules** (non-deterministic) |
| **Command directives** | `/snyk-fix` and `/snyk-batch-fix` — remediation playbooks you invoke on demand |

Snyk does **not** offer a hosted remote MCP server. It runs on your machine because it needs to read
your files.

The MCP server exposes 12 tools: `snyk_sca_scan`, `snyk_code_scan`, `snyk_iac_scan`,
`snyk_container_scan`, `snyk_sbom_scan`, `snyk_aibom`, `snyk_trust`, `snyk_auth`, `snyk_logout`,
`snyk_version`, `snyk_send_feedback` and `snyk_package_health_check`. Which ones load depends on the
profile — `snyk mcp --profile=lite|full|experimental`, or `SNYK_MCP_PROFILE`; default is `full`.

> **`snyk_package_health_check` is the best new demo in this course.** It vets a package *before*
> the agent adds it to `package.json` — the shift-left move that actually prevents the dependency
> finding instead of reporting it. GA, and on by default in the `full` profile.

### 8.2 Install

Hooks-based install — currently **Claude Code, Cursor, Codex CLI and Gemini CLI**. Note this is a
two-step download-then-run, not a pipe-to-shell:

```bash
# macOS / Linux
curl -fsSL 'https://raw.githubusercontent.com/snyk/studio-recipes/main/installer/dist/snyk-studio-install.sh' \
  -o snyk-studio-install.sh
bash ./snyk-studio-install.sh
```

```powershell
# Windows
powershell -Command "Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/snyk/studio-recipes/main/installer/dist/snyk-studio-install.ps1' -OutFile snyk-studio-install.ps1"
powershell -ExecutionPolicy Bypass -File .\snyk-studio-install.ps1
```

The installer configures the MCP server, the guardrail hooks, and the `/snyk-fix` and
`/snyk-batch-fix` commands. Hooks-based guardrails are tagged **EA** as of 11 May 2026. GitHub
Copilot gets commands, MCP and a skill — treat hooks support there as unconfirmed.

MCP-only install, for any other assistant:

```bash
# generic
npx -y snyk@latest mcp -t stdio

# per-ADE config helper
snyk mcp config --tool=cursor
# also: windsurf, antigravity, "visual studio code", gemini-cli, claude-cli
```

Manual config, e.g. `~/.claude.json`:

```json
{
  "mcpServers": {
    "Snyk": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "snyk@latest", "mcp", "-t", "stdio"],
      "env": {}
    }
  }
}
```

Then run `snyk auth`, and on the first scan let the agent call `snyk_trust` on this folder.

### 8.3 Use it

In your assistant's chat, from the repo root:

```
/snyk-fix
```

With no arguments it remediates what it finds. Scope it instead:

```
/snyk-fix the SQL injection findings in routes/
/snyk-fix only the Snyk Open Source findings in package.json
/snyk-batch-fix
```

Use `/snyk-batch-fix` to work through a large backlog in grouped passes rather than one issue at a
time.

### 8.4 What good looks like

A well-behaved run should:

- **Scan first** — call the Snyk MCP scan tools (`snyk_code_scan`, `snyk_sca_scan`) rather than
  guessing from the code
- **Change narrowly** — one focused edit per finding, not a refactor
- **Re-scan** — confirm the finding is gone *and* that nothing new appeared. If it skips this, it
  hasn't finished
- **Build and test** — then summarize what changed

What to watch for, and this is where the value is:

- a fix that only *moves* the vulnerability
- a suppression comment instead of a fix
- a change that also quietly edits a test so it stops failing

Guardrail (hooks) mode makes the scan automatic for code the agent itself writes. There is also a
lower-token "smart apply" mode where the LLM decides when to scan — faster, but more chances for
insecure code to land. Worth discussing in the room; it's a risk decision, not a performance one.

### 8.5 Exercise

```bash
git fetch upstream
git checkout -b snyk-training/lm/08-agentic-fix upstream/ai-agent-snyk-fix-training
```

1. Run `snyk code test` and record the baseline count.
2. In your assistant: `/snyk-fix the XXE finding in lib/xml.ts and the path traversal in routes/fileServer.ts`
3. Read **every** diff before accepting. Look specifically for the three failure modes in §8.4.
4. `npm run test:server && npm run test:api` — did it break the challenge tests? Good.
5. `snyk code test` — did the count actually drop?
6. Commit, push, open a PR into `develop`, and watch the PR check (§10).

**Docs:** <https://docs.snyk.io/agent-security/agentic-security-with-snyk-studio/getting-started-with-snyk-studio>

---

## 9. Snyk CLI Remediation Agent

### 9.1 What it is **(Preview — experimental, design partners)**

The Remediation Agent extends the remediation loop from your AI assistant's chat into the terminal,
so you can remediate **en masse** rather than issue by issue. It runs locally, uses **your own
model** — frontier or locally hosted — and is guided by Snyk's security intelligence rather than
relying on the model's own judgement about what's exploitable and what a correct fix looks like.

> **Scope today is Snyk Open Source (SCA) only.** Snyk Code, Container and IaC are described as in
> active development. If you want agentic remediation of SAST findings today, that is `/snyk-fix`
> (§8) — not this.

### 9.2 Why the Snyk context matters

Snyk's reported figures for a guided model versus the same model alone:

| Measure | Model alone | With Snyk context |
| --- | --- | --- |
| Overall fix rate | ~23% | ~45% |
| Critical / high / medium fix rate | ~44% | ~91% |
| Token cost **per fix** | baseline | −61% |

Keep the qualifiers when you quote these. They come from a controlled Snyk benchmark that added
Snyk tooling to `claude-haiku-4-5`; the methodology is stated as "to be published soon." The −61% is
cost *per fix*, not total spend.

It also does **breakability analysis** — roughly half of a typical backlog is low-breakability, which
makes "fix all the low-breakability findings" a genuinely safe single pass rather than a year of
tickets. And it makes code changes alongside version bumps, which is exactly what you need when an
upgrade is a breaking one (see the `express-jwt` jump in §7.1).

### 9.3 Status — read this before you plan around it

- **Experimental, for design partners.** That is Snyk's own wording. There is no paid feature gate,
  and there is no docs.snyk.io page.
- **The command name and flags are not public.** Run `snyk --help` on your installed CLI and check
  <https://docs.snyk.io/developer-tools/snyk-cli>. Do not copy syntax off a slide — including this
  document. Note the legacy `snyk fix` command is **not** the Remediation Agent.
- Design-partner builds authenticate the model through **LiteLLM**, so expect to supply a model
  endpoint and key. Your model, your inference cost.
- A **cloud** remediation agent and an **asynchronous** agent — one that runs unprompted against a
  backlog filter and produces validated, mergeable PRs, plus an AppSec experience for triggering
  remediation campaigns — are listed under "what's next" with **no committed date**.
- If it isn't enabled for your organization, that's normal. §8 is the generally available path to
  the same idea.

### 9.4 How to think about the three AI fix paths

|  | Where you are | Scope | Whose model | Status |
| --- | --- | --- | --- | --- |
| **Snyk Agent Fix** | IDE, and PR inline comments | one finding at a time | Snyk's | **GA** in IDE; **EA** in the PR |
| **`/snyk-fix`** | your AI assistant's chat | a file, a folder, a product | your assistant's | **GA** |
| **Remediation Agent** | your terminal | backlog-scale, en masse | yours, locally | **Preview**, SCA only |

Same discipline applies to all three: **you review the diff, you run the tests, you own the commit.**
None of these is run-and-merge.

---

## 10. Snyk PR Checks

This is where everything above becomes a team habit rather than a personal one.

### 10.1 What you'll see on a PR

By default Snyk scans every PR on a monitored repository and reports one security check and one
license check:

| Status | Meaning | Your move |
| --- | --- | --- |
| **Success / Passed** | nothing violating the configured fail conditions | merge |
| **Pending** | the test is still running | wait |
| **Failed / Issues found** | issues that must be fixed for the check to pass | **Details** → fix → push |
| **Error** | build file out of sync, unreadable, or not found | check your manifest and lockfile |
| **Canceled** | monthly test limit reached | talk to your admin |

Plus, for Snyk Code, **inline comments** on the changed lines — max 10 per PR — and a **summary
comment** counting active (unignored) findings.

> Snyk itself never blocks a merge. Blocking is your SCM's branch protection requiring the Snyk
> status check. Two different systems, and it matters when you're debugging why a red check merged
> anyway.

### 10.2 Reading the result

1. Select **Details** on the failing Snyk check.
2. You land on a summary in the Snyk UI, linking to a test page with an issue card per recommended
   fix — the same cards you saw in the IDE.
3. Fix on your branch, push, and the check re-runs on the new commit.

### 10.3 Fail conditions

Your admin configures these at org or project level. The two primary rules are mutually exclusive:

| Setting | Effect |
| --- | --- |
| *Only fail when the PR is adding a dependency with issues* | you're accountable for what you introduce, not the whole inherited backlog |
| *Fail if the repo has any issues* | strict — needs the backlog cleared first |
| *Only fail for high or critical severity issues* | less noise while you ramp |
| *Only fail when the issues found have a fix available* | never blocks on something you can't act on — the fairness setting |

Snyk Code uses a separate **Minimal severity to fail PR check**. Note that *fix available* relies on
Fix PR support, so it won't alert for languages Snyk can't generate a fix for.

For this training repo, the recommended starting point is: fail only on newly introduced
high/critical issues that have a fix available.

### 10.4 Ignores and PR checks

- Ignored findings **do not** fail a PR check and are not counted in the summary.
- If you ignore a finding *after* a check has completed, the check must be **retriggered by a new
  commit**. On retrigger, the finding drops out of the summary table and its inline comment
  collapses and is marked resolved.
- Ignores are respected whether they came from a policy or an individual finding-level ignore.
- Snyk Code ignores are respected in the CLI too — so `snyk code test --severity-threshold=high` in
  CI exits 0 if the only high findings are ignored. Same logic, same result, whichever gate runs.
  This is why ignore governance and gating are the same conversation.

### 10.5 Snyk Agent Fix in the PR **(EA)**

Where a Group admin has enabled it — SCM integration settings → *Pull request experience* → inline
comments → *Enable Snyk Agent fix in the PR* — you can reply to a Snyk inline comment:

```
@snyk /fix          # request a fix; repeat for a different suggestion (up to 5)
@snyk /apply 2      # apply suggestion #2 — Snyk commits it to the PR branch
```

Those two are the entire command set. Notes and limits:

- Only on **inline comments Snyk created after** the feature was enabled
- Only on findings flagged automatically fixable — the zap icon
- **Single-file fixes only**
- Suggestions expire; re-request with `/fix`
- Not supported with Snyk Code Local Engine
- Brokered integrations need **Broker 4.219 or higher** (and 4.194+ for inline comments at all)
- **GitHub requires a dedicated account authenticated with a PAT** since April 2026

Public docs still label this **Early Access** even though internal release notes call it GA. Treat
the docs as customer-facing truth and confirm with your admin.

### 10.6 Wire up CI in this repo

There is **no Snyk workflow in this repository** — `.github/workflows/` has `ci.yml`,
`codeql-analysis.yml`, `zap_scan.yml` and others, but nothing Snyk. Adding one is the exercise.

Create `.github/workflows/snyk.yml`:

```yaml
name: Snyk

on:
  push:
    branches: [develop]
  pull_request:

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
      - run: npm ci --ignore-scripts
      - run: npm install -g snyk@latest

      - name: Snyk Open Source test
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: snyk test --all-projects --severity-threshold=high

      - name: Snyk Code test
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: snyk code test --severity-threshold=high --sarif-file-output=snyk-code.sarif

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: snyk-code.sarif

      - name: Snyk monitor
        if: github.ref == 'refs/heads/develop'
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: snyk monitor --all-projects
```

It needs a `SNYK_TOKEN` repository secret.

Three things to notice, and they're the teaching points:

1. **`--severity-threshold=high` is what makes this a gate.** Without it the job passes on anything.
2. **`--sarif-file-output` returns no results for Open Source tests** — it's a Snyk Code flag here.
   Don't wire an SCA SARIF upload and wonder why the Security tab is empty.
3. **`snyk monitor` runs last, and only on `develop`.** The monitored snapshot should track green
   builds on your integration branch, not every feature branch.

If you want the scans to *report* without failing the pipeline while people are learning, add
`continue-on-error: true` to the test steps — then remove it, because a gate that can't fail isn't a
gate.

### 10.7 Exercise

1. On a branch, deliberately reintroduce a vulnerable dependency:

   ```bash
   npm install sanitize-html@1.4.2
   ```

2. Commit, push, open a PR into `develop`.
3. Watch the Snyk check fail. Read the details page.
4. Fix it — bump to a clean version — push, and watch the check go green **on the same PR**.
5. Open a second PR that introduces a Snyk Code finding: string-concatenate a query parameter into a
   `sequelize.query` call in a new route. Note the inline comment. If Agent Fix in the PR is
   enabled, try `@snyk /fix`.
6. **Compare the three feedback loops you've now used** — IDE, CLI, PR check. Which caught it
   earliest? Which one would your team actually be blocked by? Which one do you want to be blocked
   by?

---

## Appendix A — Expected findings

Baseline for this repository, `ai-agent-snyk-fix-training` branch.

> Counts drift — Snyk ships Snyk Code accuracy releases roughly monthly, and the vulnerability
> database changes daily. Capture your own baseline and treat everything here as approximate. The
> **classes** of finding are the stable part.

### Snyk Open Source — `snyk test --all-projects`

Two manifests, both with deliberately stale dependencies.

`package.json` — highest-value targets:

| Package | Version | Why it's there |
| --- | --- | --- |
| `express-jwt` | `0.1.3` | exact pin, ancient — authorization bypass class |
| `jsonwebtoken` | `0.4.0` | exact pin, ancient — signature verification issues |
| `sanitize-html` | `1.4.2` | exact pin — known XSS filter bypass |
| `marsdb` | `^0.6.11` | unmaintained; the `$where` eval sink in §7.2 #3 |
| `notevil` | `^1.3.3` | sandbox escape; the sink in §7.2 #8 |
| `js-yaml` | `^3.14.0` | old major — code execution via `load()` |
| `helmet` | `^4.6.0` | five majors behind |
| `download` | `^8.0.0` | deprecated; drags in old `got` / `decompress` transitively |
| `multer` | `^1.4.5-lts.1` | 1.x is EOL |
| `socket.io` | `^3.1.2` | old major |

`frontend/package.json`:

| Package | Version |
| --- | --- |
| `ethers` | `^5.7.2` (v5, while the server is on `^6.16.0`) |
| `@wagmi/core` | `^0.5.8` |
| `@fortawesome/free-solid-svg-icons` | `^5.14.0` |
| `snarkdown` | `^1.2.2` — feeds `innerHTML` at `frontend/src/hacking-instructor/index.ts:126` |
| `socket.io-client` | `^3.1.0` |
| `cookieconsent` | `^3.1.1` |

Most of the reported CVE count will be **transitive** — packages nobody chose. Use
`snyk test --all-projects --show-vulnerable-paths=all` to see every path.

### Snyk Code — `snyk code test`

The stable classes, with the file each lives in:

**High:** SQL Injection (`routes/login.ts`, `routes/search.ts`) · NoSQL Injection
(`routes/trackOrder.ts`, `routes/showProductReviews.ts`) · XXE (`lib/xml.ts`) · Zip Slip
(`routes/fileUpload.ts`) · Path Traversal (`routes/fileServer.ts`, `routes/keyServer.ts`,
`routes/logfileServer.ts`, `routes/dataErasure.ts`) · SSRF (`routes/profileImageUrlUpload.ts`) ·
Code Injection / unsafe eval (`routes/b2bOrder.ts`, `routes/userProfile.ts`) · Hardcoded private key
(`lib/insecurity.ts:21`)

**Medium:** DOM XSS (`frontend/src/app/search-result/search-result.component.ts` at lines 111 and
144, plus seven more `bypassSecurityTrustHtml` sites across six other files) · Open Redirect
(`routes/redirect.ts:19` + the substring "allow-list" at `lib/insecurity.ts:136`) · Weak hash — MD5
(`lib/insecurity.ts:41`)

**Low:** Insecure randomness (`routes/captcha.ts`, `data/datacreator.ts`,
`frontend/src/app/Services/conversation-storage.service.ts`)

**Not reported by SAST — on purpose:**

- IDOR / BOLA in `routes/basket.ts:18` — basket fetched by `req.params.id`, no ownership check.
  This is the clean example: the caller is authenticated, the *object* is not authorized.
- `routes/orderHistory.ts:25` — `allOrders()` returns every order in the system. Note it *is*
  role-gated: `server.ts:643` wraps it in `security.isAccounting()`, so anonymous and admin callers
  get 403 (see `test/api/order-history.test.ts`). The flaw is coarse role-based access with no
  per-object check, not a missing guard. Don't claim "no auth at all" — someone will grep `server.ts`.
- `server.ts:389` — the commented-out `security.isAuthorized()` line, with a
  `vuln-code-snippet vuln-line` marker beside it. The clearest possible artifact of a guard that
  was removed on purpose.

### Snyk IaC — `snyk iac test terraform/`

Two near-identical Terraform trees (`terraform/` and `infrastructure/terraform/`; the `.tf` files are
byte-identical apart from Dockerfile paths in `main.tf`), so expect duplicate findings:

- Hardcoded RSA private key in `aws_iam_server_certificate` — `networking.tf:171`
- ALB security group open to `0.0.0.0/0` on 80 and 443 — `networking.tf:68`
- Unrestricted egress, `protocol = "-1"` to `0.0.0.0/0` — `networking.tf:82`, `108`, `235`
- Plaintext HTTP listener with no HTTPS redirect — `networking.tf:157`
- `map_public_ip_on_launch = true` — `networking.tf:23`; ECS tasks public — `main.tf:109`
  (`assign_public_ip = true`, in the block starting at `main.tf:107`)
- EFS access point with IAM authorization disabled — `main.tf:88`
- No VPC flow logs, no ALB access logs, no WAF association

### Snyk Container

- `Dockerfile` (root) — `node:24` builder → `gcr.io/distroless/nodejs24-debian13`, `USER 65532`.
  Deliberately well-built; a good contrast.
- `infrastructure/Dockerfile` — multi-stage: `node:20.19.2-bookworm-slim` base and runtime,
  `node:20.19.2-bookworm` builder, `USER node`. The better demo target because the base is a year
  behind, not because the Dockerfile is badly written. Scan the `-slim` tag — that is what ships.
- `infrastructure/docker-compose.yml:59` — `mongo:4.4.29`, EOL and tagged in-source as the
  `vulnerableDockerImageChallenge`.

---

## Appendix B — Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `npm install` fails on native modules | Node version outside `22 - 26`. Check `node --version` first, always |
| `npm install` takes forever | expected — `postinstall` installs and builds the whole Angular frontend. 5–10 min cold |
| `npm install` fails with `ERESOLVE` in `postinstall` | use `npm install --legacy-peer-deps` (§3.1). `angularx-qrcode@21` peer-conflicts with Angular 22. The `rxjs@7.8.2` line in the error is a red herring |
| `snyk test` → `Missing node_modules folder` / `Failed to get dependencies for all 2 potential projects` | you haven't run `npm install` — and there is no lockfile to fall back on (`package-lock=false` in both `.npmrc` files). See §3.1 |
| `snyk test --all-projects` reports only 1 project | one of the two `node_modules` trees is missing — usually `frontend/`, because `postinstall` died. Re-run with `--legacy-peer-deps` |
| `npm run build` → `The Angular CLI requires a minimum Node.js version of v22.22.3 or v24.15.0 or v26.0.0` | Node satisfies `engines` but not the Angular floor. `nvm install 24 && nvm use 24` (§1.1) |
| `sqlite3` fails to load after switching Node | you crossed a major (24 → 26), changing the native ABI. `rm -rf node_modules` and reinstall |
| `command not found: snyk` right after `nvm use` | nvm scopes global npm packages per Node version. Reinstall (`npm install -g snyk@latest`) or install outside nvm (§1.3). Your auth token is unaffected |
| Scans suddenly fail with `Client request cannot be processed (SNYK-0003)` / `400 Bad Request` | **usually an expired login, not a malformed request.** `snyk auth` tokens are short-lived OAuth tokens. Confirm with `snyk doctor` — look for `✗ [AUTHENTICATION]` — then re-run `snyk auth`. A mid-session expiry is common in a long workshop |
| `snyk test --all-projects` reports far more projects than you have, or crawls | it found `package.json` files in a stray directory — `build/` (the `tsc` output includes one), a `node_modules` backup, or a copy of the repo. Use `--exclude=build` / `--detection-depth`, and never keep a `node_modules` backup inside the repo |
| `snyk test` exits non-zero and you think the scan broke | **expected.** Findings cause a non-zero exit. Read the summary block, not `$?` |
| `dotnet` restore/build shows `NU1902` / `NU1903` warnings | expected — NuGet flagging the intentionally outdated packages. The build still succeeds |
| `snyk test` finds nothing in the .NET repo | run `dotnet restore` first; Snyk reads the restored `project.assets.json`, not the `.csproj` |
| .NET `dotnet test` fails on `PipeWriter` / `System.Text.Json` | runtime/`Mvc.Testing` mismatch — the test project must target the runtime you actually have (§1.2) |
| `snyk code test` → "no supported files" | you're not in the repo root, or Snyk Code is disabled for your org |
| `snyk test` finds only one manifest | you need `--all-projects` to pick up `frontend/package.json` |
| SARIF upload is empty for SCA | expected — `--sarif-file-output` returns no results for Open Source tests |
| Ignores work locally but CI still fails | you didn't commit `.snyk`, or it's a Snyk Code finding (`.snyk` doesn't apply) |
| `--include-ignores` errors on `snyk test` | it's a `snyk code test` / `snyk secrets test` flag only |
| Ignored issue still shows in the Web UI | the Project needs a retest — look for the banner on the Project page |
| `snyk ignore create` not recognised | CLI older than **v1.1297.1** |
| The IDE extension shows nothing | the folder wasn't trusted, or the wrong organization is selected |
| Findings land in the wrong Snyk org | set the org in IDE settings, or pass `--org=<org-id>` to the CLI |
| Regional login loops | `snyk config set endpoint=…` and use the regional app URL |
| MCP tools don't appear in your assistant | run `snyk auth`, then let the agent call `snyk_trust` on the folder |
| Agent Fix never appears in PRs | not enabled at Group/Org; or on GitHub, no dedicated PAT account (April 2026 requirement) |
| Expected email about a new vulnerability, got none | new-vulnerability notification emails are **off by default** in 2026 |
| Challenge tests fail after you fix something | expected — they assert the vulnerability works. Update them to assert secure behaviour |
| `npm run rsn` fails after a challenge edit | the Refactoring Safety Net caught a challenge regression |
| Nothing else explains it | `snyk doctor` (CLI v1.1306.0+) checks auth, connectivity, proxy and config in one pass |

---

## Appendix C — What's new since v2.0

This course was retargeted from the dual-stack `juice-shop-dotnet` repository to Node.js-only on
`lmaeda/juice-shop @ ai-agent-snyk-fix-training`. Beyond that, the Snyk product facts that changed:

**Corrections to v2.0**

- CLI floor for `snyk ignore create` is **v1.1297.1**, not v1.299.0
- The Remediation Agent covers **SCA only** — not Snyk Code
- Ignore Approval Workflow scope is **Snyk Code + Snyk Secrets**; its GA/EA status is genuinely
  ambiguous between the release note and the docs
- Agent Fix is toggled at **Group *or* Org** level, and the PR flow is documented as **EA**, not GA
- PR check statuses are **Success/Passed · Pending · Failed/Issues found · Error · Canceled**
- `@snyk /apply #` is the documented notation
- The Studio installer is a **two-step download-then-run**, and hooks cover Claude Code, Cursor,
  Codex CLI and Gemini CLI — Copilot hooks are unconfirmed
- Studio is described as **five layers**, not three
- **CSV export has no row cap** (the 50-row truncation is PDF only)

**New, worth knowing as a developer**

- **Snyk Secrets GA** (July 2026) — `snyk secrets test`, IDE, pre-commit hook, PR checks. Needs
  org-level enablement, and repos must be **re-imported**
- **`snyk_package_health_check`** GA in the MCP `full` profile — vet a package before the agent adds
  it
- **`snyk doctor`** (v1.1306.0) — one-shot diagnostics
- **`snyk aibom` / `aibom test`** — AI bill of materials, now Python, Java, JS and Go; can gate CI
- **Reachability flags** in the CLI — `--reachability=true`, `--reachability-filter=reachable`,
  `--source-dir` (Snyk Preview; new findings schema can break JSON automations)
- **Snyk Code Rule Extensions GA** (Enterprise) — register your own sanitizers to cut false
  positives
- **Snyk 2.0 navigation (EA)** — Tenant/Group/Org scope selector; the **Organization Dashboard is
  removed**; Analytics is the single reporting home. Older screenshots are already wrong
- **Evo by Snyk / Agentic Development Security GA** — separate console at `evo.snyk.io`, separate
  roles, account-team enablement
- **Auto-close of obsolete Fix PRs** GA and on by default; **breakability risk analysis** in EA

**Removed / deprecated — don't teach these**

- `snyk protect` (the Node patch mechanism)
- Snyk Code **fix examples** — removed from the Web UI *and* SARIF on 17 August 2026
- Snyk Code **Local Engine** — deprecated
- `snyk redteam` / Snyk Agent Red Teaming — retired June 2026
- v1 Reporting API — deprecated; Organization Dashboard — removed
- Visual Studio 2026 — not supported

---

## Appendix D — Resources

**Snyk product docs**

- Snyk CLI — <https://docs.snyk.io/developer-tools/snyk-cli>
- Ignore issues — <https://docs.snyk.io/scan-fix-and-prevent/fix/prioritize-issues-for-fixing/ignore-issues>
- Consistent Ignores for Snyk Code — <https://docs.snyk.io/scan-fix-and-prevent/fix/prioritize-issues-for-fixing/ignore-issues/consistent-ignores-for-snyk-code>
- `snyk ignore create` — <https://docs.snyk.io/developer-tools/snyk-cli/snyk-cli/commands/ignore-create>
- Snyk Agent Fix — <https://docs.snyk.io/scan-fix-and-prevent/scan-with-snyk/snyk-code/manage-code-vulnerabilities/fix-code-vulnerabilities-automatically>
- Pull request experience — <https://docs.snyk.io/scan-fix-and-prevent/prevent/pull-request-checks/pull-request-experience>
- Analyze PR check results — <https://docs.snyk.io/scan-fix-and-prevent/prevent/pull-request-checks/analyze-pr-checks-results>
- GitHub integration — <https://docs.snyk.io/developer-tools/integrations/scm-integrations/organization-level-integrations/github>
- Snyk Studio — <https://docs.snyk.io/agent-security/agentic-security-with-snyk-studio>
- Snyk Secrets — <https://docs.snyk.io/scan-fix-and-prevent/scan-with-snyk/snyk-secrets>
- JavaScript support — <https://docs.snyk.io/supported-languages/supported-languages-list/javascript>
- IDE plugins — <https://snyk.io/platform/ide-plugins>
- CLI cheat sheet — <https://snyk.io/blog/snyk-cli-cheat-sheet>
- Remediation Agent in the CLI — <https://snyk.io/blog/snyk-remediation-agent-in-the-cli/>

**Learning**

- Snyk Learn — <https://learn.snyk.io>
- Product training catalog — <https://learn.snyk.io/catalog/product-training>
- Consistent Ignores lesson — <https://learn.snyk.io/lesson/snyk-consistent-ignores/>
- Vulnerability Database — <https://security.snyk.io>
- Snyk Advisor (package health) — <https://snyk.io/advisor>

**This repo**

- `AGENTS.md` — AI contribution guidelines and repo skills; the authoritative source for agents
- `.claude/CLAUDE.md` — Claude-specific pointer to `AGENTS.md`
- `SOLUTIONS.md` / `REFERENCES.md` — Juice Shop challenge material
- `data/static/codefixes/` — curated vulnerable / `*_correct.ts` fixed pairs
- Companion decks — `Snyk-Developer-Training.pptx`, `Snyk-Developer-Instructor-Guide.docx`
- Team leader track — `Snyk-Team-Leader-Training.pptx`, `Snyk-Team-Leader-Instructor-Guide.docx`

**Support**

- <https://support.snyk.io> · <mailto:support@snyk.io>
- Snyk Developer Community — <https://community.snyk.io>
