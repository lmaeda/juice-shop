# Verified `snyk monitor` commands — Snyk UB registry evaluation

All flags below were checked against `snyk monitor --help` on **Snyk CLI 1.1306.2**.
Load credentials first, preferably in a subshell so they do not linger:

```bash
( set -a; . ./.env_snyk_ub; set +a; <command> )
```

> **`snyk monitor` does not route through the Universal Broker.** The CLI resolves
> dependencies locally and posts the graph to `api.snyk.io`. The broker exists so Snyk's
> SaaS can reach *inward*. "Registering through a private registry" therefore means
> pointing the **package manager** at that registry, not the broker.

## npm — juice-shop

```bash
cp snyk-ub-eval/npmrc.nexus.example .npmrc     # local only; never commit
npm ci --ignore-scripts --no-audit --no-fund

snyk monitor \
  --file=package-lock.json \
  --org="$SNYK_ORG_ID" \
  --project-name=juice-shop-training \
  --remote-repo-url=https://github.com/lmaeda/juice-shop \
  --target-reference=ai-agent-snyk-fix-training
```

**`--file=package.json` fails even when a lockfile exists** — it forces the `node_modules`
strategy and errors with `Missing node_modules folder`. Point `--file` at the lockfile, or
omit `--file` and let Snyk auto-detect.

npm rewrites the lockfile's `resolved` hosts to the configured registry, so a
`package-lock.json` generated against `registry.npmjs.org` works unchanged against Nexus and
`npm ci` leaves it byte-identical.

## Maven/Gradle — WebGoat-Gradle

```bash
export JAVA_HOME=~/.sdkman/candidates/java/11.0.31-tem   # Spring Boot 2.7 / Gradle 8.14.4

snyk monitor \
  --file=build.gradle \
  --init-script=snyk-ub-eval/nexus-init.gradle \
  --configuration-matching='^runtimeClasspath$' \
  --org="$SNYK_ORG_ID" \
  --project-name=webgoat-gradle \
  --remote-repo-url=https://github.com/lmaeda/WebGoat-Gradle \
  --target-reference=ai-agent-snyk-fix
```

Swap `--init-script` for `artifactory-init.gradle` to resolve through Artifactory instead.
Do **not** add `--all-sub-projects`: this repo has no `settings.gradle`, and the CLI
documents that both files must exist.

## Go — juice-shop-go

```bash
cd insecurity-go        # go.mod is NOT at the repo root

export GOPROXY="$NEXUS_GOPROXY" GOSUMDB=off GOFLAGS=-mod=mod
unset GOPRIVATE GONOPROXY          # see the warning below

go mod download all

snyk monitor \
  --org="$SNYK_ORG_ID" \
  --project-name=juice-shop-go-insecurity \
  --remote-repo-url=https://github.com/lmaeda/juice-shop-go \
  --target-reference=ai-agent-snyk-fix
```

> ⚠️ **Never set `GOPRIVATE`/`GONOPROXY` when testing a registry.** `GOPRIVATE=*` implies
> `GONOPROXY=*`, which makes Go **bypass the proxy entirely** and clone from the origin VCS.
> The command succeeds, modules land in the cache, and the registry sees **zero** requests —
> a silent false success that produced a wrong result during this evaluation.
>
> Artifactory cannot serve this: OSS offers no Go package type, and the go client refuses
> credentials over plain HTTP. `GOINSECURE` does **not** override that (tested).

## Proving the registry was actually used

Local caches will serve an entire install with zero registry traffic. Use a throwaway cache
and a before/after counter — not log timestamps:

```bash
B=$(docker exec nexus sh -c 'grep -c "npm-all" /nexus-data/log/request.log')
npm ci --cache "$(mktemp -d)" --ignore-scripts
A=$(docker exec nexus sh -c 'grep -c "npm-all" /nexus-data/log/request.log')
echo "delta: $((A-B))"      # 0 means the registry was NOT used
```

Artifactory logs the **requested path**, not the serving member, so grep the *virtual* repo
name (`maven-virtual`), never `libs-release-local`.
