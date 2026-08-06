# Snyk Universal Broker evaluation — juice-shop (npm)

Branch `snyk-ub-eval-2026-08-06`. **No application code was changed.** This directory
records how this app was resolved from an on-prem private registry and registered in Snyk.

## Result

Registered as Snyk project **`juice-shop-training`** (npm), resolved from the on-prem
**Nexus** `npm-all` group — `snyksuccess-nexus-npm` (hosted, private) ordered before
`npm-proxy` (registry.npmjs.org), so private packages shadow upstream.

Two direct dependencies were published into the private hosted repo:
`jsonwebtoken@0.4.0` and `sanitize-html@1.4.2` — both exact pins in `package.json`.

> **Artifactory OSS cannot host npm.** Its package-type grid offers only Maven, Gradle,
> Ivy, SBT and Generic. npm needs live packument metadata that a Generic repo cannot
> synthesise, so there is no workaround at that edition.

## Reproduce

```bash
cp snyk-ub-eval/.env_snyk_ub_sample .env_snyk_ub    # fill in; keep it gitignored
( set -a; . ./.env_snyk_ub; set +a
  cp snyk-ub-eval/npmrc.nexus.example .npmrc        # LOCAL ONLY -- see below
  npm ci --ignore-scripts --no-audit --no-fund
  snyk monitor --file=package-lock.json --org="$SNYK_ORG_ID" \
    --project-name=juice-shop-training \
    --remote-repo-url=https://github.com/lmaeda/juice-shop \
    --target-reference=ai-agent-snyk-fix-training )
```

## Why `.npmrc` is not committed

The repo's real `.npmrc` is deliberately left untouched. A registry-pointing `.npmrc`
hardcodes `localhost:8091`, which exists only on the evaluation machine — committing it
would break `npm install` for every other clone and for CI. The template here uses npm's
`${VAR}` expansion so no credential is ever written to disk in the repo.

## Gotchas confirmed here

- **`--file=package.json` fails even with a lockfile present** — it forces the
  `node_modules` strategy and errors with `Missing node_modules folder`. Use
  `--file=package-lock.json`, or omit `--file`.
- **A lockfile generated against `registry.npmjs.org` works unchanged against Nexus.** npm
  rewrites the `resolved` hosts to the configured registry, and `npm ci` leaves the
  committed lockfile byte-identical.
- **Local cache can fake a pass.** An `npm ci` finished in 13s having made *zero* Nexus
  requests — `~/.npm/_cacache` served everything. Re-test with `--cache "$(mktemp -d)"` and
  check a before/after request count.
