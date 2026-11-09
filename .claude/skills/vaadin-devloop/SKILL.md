---
name: vaadin-devloop
description: Workspace entry point to the Vaadin dev loop of schlierelacht-admin (the Spring Boot + Vaadin 25.3 backend/admin UI). Makes a saved edit live in the already-running admin app in seconds and verifies it there, through a daemon that owns the app process — so it replaces `mvn spring-boot:run`, IDE run configurations, `mvn compile` and `mvn test` as the way to run the app and to find out whether an edit works. Read this BEFORE the first edit to admin application source — a Vaadin view, component, layout, service, stylesheet or resource under schlierelacht-admin/src/main — and follow it again once the file is saved, because a source edit that has not been applied is not done. Also covers starting/restarting the admin app, checking whether a change is live, and looking at the admin UI in a browser ("start the backend", "apply my edits", "hot reload", "why doesn't the page show my change").
---

# Vaadin dev loop — from the workspace

The authoritative instructions are generated into the admin repo by
`./mvnw vaadin:install-dev-cli` (re-run after every Vaadin upgrade) and must not be
copied or hand-edited. **Read them now and follow them**:

- `schlierelacht-admin/.claude/skills/vaadin-devloop/SKILL.md` — the cycle, commands,
  how to read `apply` output, Claude Code tooling notes
- `schlierelacht-admin/.agents/skills/vaadin-devloop/reference.md` — full reference,
  when you need it
- `schlierelacht-admin/AGENTS.md` — the admin repo's conventions (feature packages,
  jOOQ, MapStruct, API versioning)

This file only adds what differs when Claude Code was started in the **workspace root**
instead of `schlierelacht-admin/`.

## Running `vaadin-dev` from the workspace root

Paths in the admin skill (`.vaadin/vaadin-dev …`) are relative to `schlierelacht-admin/`.
From the workspace root, call the admin's script and name the app explicitly:

```bash
# Windows (this machine) — via the .cmd launcher, from the Bash tool:
cmd //c "schlierelacht-admin\.vaadin\vaadin-dev.cmd --app schlierelacht-admin status"
cmd //c "schlierelacht-admin\.vaadin\vaadin-dev.cmd --app schlierelacht-admin apply"

# start needs the `local` Spring profile (gitignored application-local.properties)
# and a running Postgres (see the workspace CLAUDE.md); the env var is inherited by cmd:
VAADIN_DEV_DAEMON_OPTS="-Dspring.profiles.active=local" \
  cmd //c "schlierelacht-admin\.vaadin\vaadin-dev.cmd --app schlierelacht-admin start"
```

- **Do not use `bash schlierelacht-admin/.vaadin/vaadin-dev` here.** It resolves the
  daemon through the POSIX `mvnw`, which fails on this machine (a broken `wget` on the
  PATH; exit 70, "could not resolve flow-devloop-daemon"). The `.cmd` launcher takes the
  same commands, options and exit codes.
- Run these with the workspace root as the working directory. Every command prints the
  application it answered for on stderr — check that it names `schlierelacht-admin`.
- `status` auto-spawns the daemon. Leave it running across edits; `shutdown` when done.
- The app serves on http://localhost:8080. Don't also run `./mvnw` / `mvn spring-boot:run`
  or the IDE run config — they fight the daemon for the port.

## Workspace specifics

- **Vaadin API/docs:** the Vaadin MCP server (`vaadin`, `https://mcp.vaadin.com/docs`) is
  configured in the workspace `.mcp.json` too — use it against `vaadin.version` from
  `schlierelacht-admin/pom.xml`.
- **DTO / endpoint changes** ripple into the website and the app: `apply` makes them live
  in the backend, but regenerating `rest.ts` and updating the consumers is still required
  (see "The coupling" in the workspace `CLAUDE.md`).
