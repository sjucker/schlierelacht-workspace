# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **workspace wrapper**, not a project. It only versions this `CLAUDE.md`
and the `README.md`. The real projects live in their own Git repos and are
cloned as siblings (excluded via `.gitignore`):

- `schlierelacht-admin/` — Spring Boot backend + Vaadin admin UI (Java 25, Maven). Has its own `AGENTS.md` with
  backend-specific details.
- `schlierelacht-website/` — Public website (Nuxt 4 / Vue 3, npm).
- `schlierelacht-app/` — Flutter mobile app (Dart) consuming the same public API.

Treat them as **one system**: the admin app is the backend that serves the public
API; the website and the mobile app are its consumers. A change to a DTO or
endpoint usually requires a coordinated change in the consumers.

## The coupling (most important thing to understand)

The website consumes the backend's **public REST API** (`/api/**`) and depends on
**TypeScript types generated from the backend's Java DTOs**:

- Backend endpoints live in `schlierelacht-admin/src/main/java/ch/schlierelacht/admin/rest/*Endpoint.java`, mapped under
  `/api/...` (e.g. `/api/news`, `/api/attraction`).
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
stays in sync. The generator is **not** bound to the build lifecycle — run it
explicitly from `schlierelacht-admin/`:

```bash
./mvnw process-classes                # compile the changed DTOs first
./mvnw typescript-generator:generate  # rewrite schlierelacht-website/shared/types/rest.ts
```

After regeneration, expect `rest.ts` to change and update the website code that
consumes it. Field optionality matters: `rest.ts` marks a field optional (`?`)
only when the Java field is nullable — fields carrying `@NotNull` (and primitives)
become required. So an "optional" DTO field is simply one *without* `@NotNull`.

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

## Mobile app architecture notes (`schlierelacht-app/`)

- **Flutter / Dart**, Material 3. Targets Android + iOS. Scaffolded with `flutter create` (org `ch.schlierelacht`,
  package `schlierelacht_app`).
- Consumes the **same public `/api/**`** as the website. Native clients are not subject to CORS. Currently covers **News
  ** (`/api/news`) and **Programm** (`/api/programm`) — the schedule is a flat, pre-sorted list of programm points
  (`ProgrammPointDTO` = a `ProgrammEntryDTO` joined with an `AttractionRefDTO`), grouped per day client-side, exactly
  like the website. Attraction detail pages use `/api/attraction/{externalId}`.
- **No code generator**: unlike the website's `rest.ts`, the Dart models under `lib/models/` are **hand-written and must
  be kept in sync with the Java DTOs** by hand. When a DTO changes, update the matching model + its `fromJson`.
- Base URL: `lib/config.dart` reads
  `String.fromEnvironment('API_BASE_URL', defaultValue: 'https://schlierelacht-api.ch')` — mirrors the website's
  env-override-then-prod-fallback. Cloudflare image URLs use the same account id (`cloudflareImageUrl`).
- Lightweight architecture: plain `http` + `FutureBuilder` (no Provider/Riverpod/Bloc). Markdown rendered with
  `flutter_markdown`; images via `cached_network_image`. German formatting in `lib/utils/formatting.dart` (`de_CH`,
  initialized in `main.dart`).
- Common commands (run from `schlierelacht-app/`):

  ```bash
  flutter pub get
  flutter analyze
  flutter test
  flutter run                                              # against prod API
  flutter run --dart-define=API_BASE_URL=http://localhost:8080      # iOS sim → local backend
  flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8080       # Android emulator → local backend
  ```

## Working across the projects

A typical end-to-end change (e.g. a new public-facing field or endpoint):

1. Backend: add/modify the DTO and/or `*Endpoint`, plus service and any Flyway migration + jOOQ regen.
2. Build the backend so `typescript-generator` rewrites `schlierelacht-website/shared/types/rest.ts`.
3. Website: consume the updated type / endpoint in the relevant page or component.
4. Mobile app (if affected): update the matching hand-written model in `schlierelacht-app/lib/models/` and the UI.
5. Run backend tests, website lint, and `flutter analyze`/`flutter test`.
