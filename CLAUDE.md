# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a live-presentation demo showcasing Spring Application Advisor (SAA) upgrading a Spring Boot 1.5.0 application to Spring Boot 4.0. The entire demo is a single script: `demo.sh`.

## Running the Demo

```bash
./demo.sh
```

Before running, the following environment variables must be set (see `.envrc` for direnv-based setup):

```bash
export NVD_API_KEY=<your-nvd-api-key>
export OSSINDEX_USERNAME=<your-ossindex-username>
export OSSINDEX_PASSWORD=<your-ossindex-password>
```

## Demo Flow Architecture

`demo.sh` orchestrates everything in sequence. The `upgrade-example/` directory is ephemeral — deleted and recreated on every run. The source app is cloned fresh from `https://github.com/dashaun/hello-spring-boot-1-5.git` each time.

**Phase 1 — Baseline (Java 8 / Spring Boot 1.5):**
1. Clone the app, start it with Java 8
2. Validate health (`/health`), capture memory (`ps -o rss=`), start OWASP CVE check in background
3. Stop the app, wait for background CVE check to finish
4. Run `advisor build-config get` → capture SBOM dependency count from `target/.advisor/build-config.json`

**Phase 2 — Analysis & Upgrade:**
5. Show advisor metadata (keys, git info, SBOM count, submodules, tools)
6. Run `advisor upgrade-plan get`, then `advisor upgrade-plan apply --squash 17`
7. Run `advisor build-config get` again on the upgraded project → capture new SBOM count

**Phase 3 — Upgraded (Java 21 / Spring Boot 4.0):**
8. Start upgraded app with Java 21
9. Validate health (`/actuator/health`), capture memory, start OWASP CVE check in background
10. Stop app, wait for CVE check, display final comparison table

**Final table columns:** Configuration | Startup Time | Deps | CVEs | (MB) Used | (MB) Savings

## Key Implementation Details

### demo-magic integration
- `vendir sync` is skipped if `vendir/demo-magic/` already exists
- `pei` — types and immediately executes a command (visible to audience)
- `p` — types a command but does NOT execute it (used to show OWASP command without exposing credential values)
- `talkingPoint` — clears screen and waits; controls pacing between sections
- `PROMPT_TIMEOUT=6`, `TYPE_SPEED=100`

### OWASP CVE check (background)
The check is started in the background immediately after `validateApp`, using `JAVA_HOME="$JAVA21_HOME"` to force Java 21 regardless of the current shell Java version (since `sdk use` only affects the current shell, not subshells). `CVE_CHECK_PID` stores the background PID; `collectCVECount` calls `wait "$CVE_CHECK_PID"` after `springBootStop`.

### Credentials are never displayed
The `p` function (not `pei`) is used to type the OWASP maven command with single quotes, so `$NVD_API_KEY` etc. appear literally on screen. The actual command runs silently in the background.

### Metric log files (written inside `upgrade-example/`)
| File | Contents |
|------|----------|
| `java8with1.5.log` | Spring Boot 1.5 startup output (startup time parsed from it) |
| `java8with1.5.log2` | Memory usage in MB |
| `java8with1.5.cves` | CVE count from OWASP report |
| `java8with1.5.deps` | SBOM component count |
| `java21with4.0.log` | Spring Boot 4.0 startup output |
| `java21with4.0.log2` | Memory usage in MB |
| `java21with4.0.cves` | CVE count from OWASP report |
| `java21with4.0.deps` | SBOM component count |

### Custom advisor mappings
Set as script-level variables (not exported env vars):
- `SPRING_ADVISOR_MAPPING_CUSTOM_0_GIT_URI=https://github.com/dashaun-tanzu/advisor-mappings.git`
- `SPRING_ADVISOR_MAPPING_CUSTOM_0_GIT_PATH=mappings/`

### Java versions
- `JAVA8_VERSION="8.0.482-librca"` — used for baseline Spring Boot 1.5 run
- `JAVA21_VERSION="21.0.10-librca"` — used for upgraded Spring Boot 4.0 run and all OWASP checks
- `JAVA21_HOME` is derived from `$SDKMAN_DIR/candidates/java/$JAVA21_VERSION` at script start (before SDKMAN is sourced)

## Manual Commands (run inside `upgrade-example/`)

```bash
# Advisor
advisor build-config get
advisor upgrade-plan get
advisor upgrade-plan apply --squash 17

# App lifecycle
./mvnw -q clean package spring-boot:start -Dfork=true -DskipTests
./mvnw spring-boot:stop -Dspring-boot.stop.fork -Dfork=true

# OWASP CVE check
./mvnw org.owasp:dependency-check-maven:check \
  -Dnvd.api.key=$NVD_API_KEY \
  -DossIndexUsername=$OSSINDEX_USERNAME \
  -DossIndexPassword=$OSSINDEX_PASSWORD \
  -Dformats=HTML,JSON

# Parse CVE count from report
cat target/dependency-check-report.json | jq '[.dependencies[].vulnerabilities[]?] | length'

# Parse SBOM count
cat target/.advisor/build-config.json | jq '.sbom.components | length'
```
