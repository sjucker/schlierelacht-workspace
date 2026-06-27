# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **workspace wrapper**, not a project. It only versions this `CLAUDE.md`
and the `README.md`. The two real projects live in their own Git repos and are
cloned as siblings (excluded via `.gitignore`):

- `schlierelacht-admin/` — Spring Boot backend + Vaadin admin UI (Java 25, Maven). Has its own `AGENTS.md` with
  backend-specific details.
- `schlierelacht-website/` — Public website (Nuxt 4 / Vue 3, npm).

Treat them as **one system**: the admin app is the backend that serves the public
API, and the website is its only API consumer. A change to one side usually
requires a coordinated change to the other.

## The coupling (most important thing to understand)

The website consumes the backend's **public REST API** (`/api/**`) and depends on
**TypeScript types generated from the backend's Java DTOs**:

- Backend endpoints live in `schlierelacht-admin/src/main/java/ch/schlierelacht/admin/rest/*Endpoint.java`, mapped under
  `/api/...` (e.g. `/api/news`, `/api/artist`).
- DTOs live in `schlierelacht-admin/src/main/java/ch/schlierelacht/admin/dto/` (classes ending in `DTO`, plus their
  enums).
- The Maven `typescript-generator-maven-plugin` (configured in `schlierelacht-admin/pom.xml`) scans
  `ch.schlierelacht.admin.**DTO` and **writes directly into the website** at
  `schlierelacht-website/shared/types/rest.ts`. Note the relative `outputFile` path — the plugin assumes both repos are
  checked out side-by-side, exactly as in this workspace.
- The website imports those types via `~/../shared/types/rest` and fetches data with `useFetch(\`$
  {config.public.apiBaseUrl}/api/...\`)`. `apiBaseUrl` defaults to `https://api.schlierelacht.ch` (`nuxt.config.ts` → `
  runtimeConfig.public`).

**When you change a DTO or endpoint shape**, regenerate the types so the website
stays in sync (the generator runs as part of the Maven build; or build the admin
project explicitly). After regeneration, expect `rest.ts` to change and update the
website code that consumes it. Field optionality matters: `rest.ts` marks a field
optional (`?`) only when the Java field carries `@NotNull` semantics per the
plugin config (`@NotNull` → required, primitives → required).

Backend CORS for `/api/**` allows all origins with `GET`/`POST` only, sessions are
`STATELESS`, and CSRF is disabled — so the public API is read-mostly and
unauthenticated. The Vaadin admin UI is the authenticated part and is separate.

## Common commands

### Backend (`schlierelacht-admin/`, Maven wrapper `./mvnw`)

```bash
./mvnw                                    # run in dev mode (default goal: spring-boot:run) → http://localhost:8080
./mvnw test                               # all tests
./mvnw test -Dtest=TaskServiceTest        # single test class
./mvnw test -Dtest=TaskServiceTest#methodName   # single test method
./mvnw -Pproduction package               # production JAR
```

Database & jOOQ codegen (Docker must be running):

```bash
docker compose -p schlierelacht -f src/main/docker/postgres.yml up --build      # start Postgres
mvn clean test-compile -Djooq-codegen-skip=false                                # regenerate jOOQ code from migrations
```

jOOQ code generation is **skipped by default** (`jooq-codegen-skip=true`). It runs
a Postgres testcontainer, applies Flyway migrations, and generates classes into
`src/main/java/ch/schlierelacht/admin/jooq/`. Re-run it after adding a Flyway
migration under `src/main/resources/db/migration/`.

### Website (`schlierelacht-website/`, npm)

```bash
npm install            # also runs `nuxt prepare` (postinstall)
npm run dev            # dev server
npm run build          # production build
npm run generate       # static generation
npm run lint           # ESLint
npm run lint:fix       # ESLint with autofix
```

Requires `.env` (copy from `.env.example`) for `MAPBOX_API_TOKEN`.

## Backend architecture notes

- **Feature-based packages**, not layered. See `schlierelacht-admin/AGENTS.md` for the full convention. Each feature (
  e.g. `news`, `artist`, `gastro`, `sponsoring`, `meetup`, `ok`, `location`) has a Vaadin view under `views/<feature>/`,
  a `service/<Feature>Service.java`, and a public `rest/<Feature>Endpoint.java` where applicable.
- **Data access is jOOQ** (not JPA) against PostgreSQL; schema is managed by **Flyway** migrations. The `jooq/` package
  is generated — never hand-edit it.
- **DTO mapping**: MapStruct + Lombok (annotation processors configured in `pom.xml`). Services return DTOs; endpoints
  just wrap them in `ResponseEntity`.
- **Vaadin** server-side UI is the admin app; routing via `@Route`/`@Menu`, secured by `SecurityConfiguration` (form
  login). The `/api/**` chain is a separate, public `SecurityFilterChain` (`@Order(10)`).
- Java version is **25**; Spring Boot 4, Vaadin 25.

## Website architecture notes

- **Nuxt 4 / Vue 3** with `@nuxt/ui`, `@nuxt/image` (Cloudflare provider), `@nuxtjs/mdc`, and `nuxt-mapbox`.
- Pages in `app/pages/` map to routes; data is fetched client-side (`{server: false}`) from the backend public API.
- Generated API types live in `shared/types/rest.ts` (do not hand-edit — see coupling section). Shared helpers in
  `app/utils/`.
- Deployed on Netlify (prod + staging); images served via Cloudflare image delivery.

## Working across both projects

A typical end-to-end change (e.g. a new public-facing field or endpoint):

1. Backend: add/modify the DTO and/or `*Endpoint`, plus service and any Flyway migration + jOOQ regen.
2. Build the backend so `typescript-generator` rewrites `schlierelacht-website/shared/types/rest.ts`.
3. Website: consume the updated type / endpoint in the relevant page or component.
4. Run backend tests and website lint.
