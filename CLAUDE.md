# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **workspace wrapper**, not a project. It only versions this `CLAUDE.md`
and the `README.md`. The real projects live in their own Git repos and are cloned as siblings (excluded via `.gitignore`):

- `schlierelacht-admin/` — Spring Boot backend + Vaadin admin UI (Java 25, Maven). Has its own `AGENTS.md` with backend-specific details.
- `schlierelacht-website/` — Public website (Nuxt 4 / Vue 3, npm).
- `schlierelacht-app/` — Flutter mobile app (Dart) consuming the same public API.

Treat them as **one system**: the admin app is the backend that serves the public API; the website and the mobile app are its consumers. A change to a DTO or endpoint usually requires a coordinated
change in the consumers.

## The coupling (most important thing to understand)

The website consumes the backend's **public REST API** (`/api/**`) and depends on **TypeScript types generated from the backend's Java DTOs**:

- Backend endpoints live in `schlierelacht-admin/src/main/java/ch/schlierelacht/admin/rest/*Endpoint.java`, mapped under
  `/api/v<n>/...` (e.g. `/api/v1/news`, `/api/v1/attraction`) — see "API versioning" below.
- DTOs live in `schlierelacht-admin/src/main/java/ch/schlierelacht/admin/dto/` (classes ending in `DTO`, plus their enums).
- The Maven `typescript-generator-maven-plugin` (configured in `schlierelacht-admin/pom.xml`) scans
  `ch.schlierelacht.admin.**DTO` and **writes directly into the website** at
  `schlierelacht-website/shared/types/rest.ts`. Note the relative `outputFile` path — the plugin assumes both repos are checked out side-by-side, exactly as in this workspace.
- The website imports those types via `~/../shared/types/rest` and fetches data with
  ``useFetch(`${config.public.apiBaseUrl}${apiPath('news')}`)`` — `apiPath` (`app/utils/api-path.ts`) supplies the versioned path. `apiBaseUrl` defaults to `https://schlierelacht-api.ch`
  (`nuxt.config.ts` → `runtimeConfig.public`).

**When you change a DTO or endpoint shape**, regenerate the types so the website stays in sync. The generator is **not** bound to the build lifecycle — run it explicitly from `schlierelacht-admin/`:

```bash
./mvnw process-classes                # compile the changed DTOs first
./mvnw typescript-generator:generate  # rewrite schlierelacht-website/shared/types/rest.ts
```

After regeneration, expect `rest.ts` to change and update the website code that consumes it. Field optionality matters: `rest.ts` marks a field optional (`?`)
only when the Java field is nullable — fields carrying `@NotNull` (and primitives)
become required. So an "optional" DTO field is simply one *without* `@NotNull`.

Backend CORS for `/api/**` allows all origins with `GET`/`POST` only, sessions are
`STATELESS`, and CSRF is disabled — so the public API is read-mostly and unauthenticated. The Vaadin admin UI is the authenticated part and is separate. (The security matcher and the CORS registration
both stay on `/api/**`, so they cover every version automatically.)

## API versioning

`schlierelacht-admin` has a single `main` branch, while the website runs `main`
(PROD) and `develop` (STAGING) — and a released mobile app build keeps calling the backend for as long as it stays installed. So the backend can never change a response shape in place. Instead, the
public API is versioned **per resource**:

```
/api/v1/news          /api/v2/attraction    <- only attraction was bumped
/api/v1/programm      /api/v1/location
```

Each resource carries its own version, so a breaking change to one endpoint leaves the others untouched. **Each project keeps its versions in exactly one file** — that file is the only thing you edit
to move a consumer:

| Project   | Where the version lives                                                 |
|-----------|-------------------------------------------------------------------------|
| `admin`   | the `@RequestMapping("/api/v1/<resource>")` on each endpoint            |
| `website` | `app/utils/api-path.ts` (`API_VERSION` map, used via `apiPath('news')`) |
| `app`     | `lib/api/api_paths.dart` (`ApiPaths` constants)                         |

An ArchUnit rule (`ArchUnitTest#rest_endpoints_should_be_versioned`) fails the backend build if a `@RestController` is mapped without a version.

**When to bump**: only on a *breaking* change — removing/renaming a field, changing a field's type, making an optional field required, changing a path or status code, or changing the meaning of an
existing value. Additive changes (a new optional field, a new endpoint) stay on the current version.

**How to bump**:

1. `admin`: add a `/api/v<n+1>/<resource>` mapping with the new shape and leave the old mapping and its DTO in place, so current clients keep their responses. Regenerate `rest.ts` (see above).
2. `website`: bump that resource's entry in `app/utils/api-path.ts` on `develop`
   first; `main` follows when the change ships to PROD.
3. `app`: bump the constant in `lib/api/api_paths.dart` and update the hand-written model — it reaches users only with the next release.
4. Delete the old version in `admin` once nothing calls it any more.

**There are no unversioned paths.** Every consumer calls a versioned path, so the transitional forwarding filter was removed — `/api/news` is no longer routed. Always go through `apiPath()` /
`ApiPaths` rather than writing a path by hand.

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

jOOQ code generation is **skipped by default** (`jooq-codegen-skip=true`). It runs a Postgres testcontainer, applies Flyway migrations, and generates classes into
`src/main/java/ch/schlierelacht/admin/jooq/`. Re-run it after adding a Flyway migration under `src/main/resources/db/migration/`.

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

- **Feature-based packages**, not layered. See `schlierelacht-admin/AGENTS.md` for the full convention. Each feature (e.g. `news`, `artist`, `gastro`, `sponsoring`, `meetup`, `ok`, `location`) has a
  Vaadin view under `views/<feature>/`, a `service/<Feature>Service.java`, and a public `rest/<Feature>Endpoint.java` where applicable.
- **Data access is jOOQ** (not JPA) against PostgreSQL; schema is managed by **Flyway** migrations. The `jooq/` package is generated — never hand-edit it.
- **DTO mapping**: MapStruct + Lombok (annotation processors configured in `pom.xml`). Services return DTOs; endpoints just wrap them in `ResponseEntity`.
- **Vaadin** server-side UI is the admin app; routing via `@Route`/`@Menu`, secured by `SecurityConfiguration` (form login). The `/api/**` chain is a separate, public `SecurityFilterChain`
  (`@Order(10)`).
- **Push notifications (FCM)**: `rest/PushEndpoint.java` exposes `POST /api/v1/push/register` (`PushRegistrationDTO`) so the mobile app can register its device token (stored in the `push_device_token`
  table). `service/PushService.java`
  sends via the Firebase Admin SDK (`firebase-admin` in `pom.xml`) — multicast to all tokens or to the `general` topic — and `views/notification/PushNotificationView.java` is the admin "Push senden"
  view. `config/FirebaseConfig.java`
  builds the `FirebaseMessaging` bean from a service-account file at `app.firebase-credentials`
  (env `FIREBASE_CREDENTIALS`); when unset the bean is `null` and sending is cleanly disabled.
- Java version is **25**; Spring Boot 4, Vaadin 25.
- **Vaadin agentic development** also works from this workspace root: the Vaadin docs MCP server is in the workspace
  `.mcp.json`, and the workspace skill `.claude/skills/vaadin-devloop` delegates to the admin's generated
  `vaadin-devloop` skill (don't copy or hand-edit that one — `./mvnw vaadin:install-dev-cli` rewrites it). From the
  workspace, run the dev loop as
  `cmd //c "schlierelacht-admin\.vaadin\vaadin-dev.cmd --app schlierelacht-admin <status|start|apply|stop>"`;
  `bash .../vaadin-dev` fails here because it goes through the POSIX `mvnw`.

## Website architecture notes

- **Nuxt 4 / Vue 3** with `@nuxt/ui`, `@nuxt/image` (Cloudflare provider), `@nuxtjs/mdc`, and `nuxt-mapbox`.
- Pages in `app/pages/` map to routes; data is fetched client-side (`{server: false}`) from the backend public API.
- Generated API types live in `shared/types/rest.ts` (do not hand-edit — see coupling section). Shared helpers in
  `app/utils/` — dates come from `format-weekday.ts` ("Freitag"), `format-date-short.ts` ("03.09.2027") and
  `format-date.ts` (both, composed).
- **API paths go through `app/utils/api-path.ts`**: ``useFetch(`${config.public.apiBaseUrl}${apiPath('news')}`)``. Its
  `API_VERSION` map is the single place this branch's API versions are pinned — never hardcode `/api/...` in a page.
- **Filtering is client-side** on the list pages: `programm` (free-text over the act name, plus date and stage selects, all resettable), `gastro` (free-text over Nr. + name + operator), and
  `locations` (free-text over name).
- The programm list has no column header — a `USeparator` day divider (weekday emphasized, date secondary) starts each new day, in both the upcoming and the "Vergangene" group.
- Deployed on Netlify (prod + staging); images served via Cloudflare image delivery.

## Mobile app architecture notes (`schlierelacht-app/`)

- **Flutter / Dart**, Material 3. Targets Android + iOS. Scaffolded with `flutter create` (org `ch.schlierelacht`, package `schlierelacht_app`).
- Consumes the **same public `/api/**`** as the website and has **feature parity with it** (every website page has
  an app screen; the app adds Favoriten, push, onboarding and distances on top). When a website page gains a feature,
  mirror it in the app. Native clients are not subject to CORS. A **landing page** (`screens/landing_screen.dart`,
  the app entry point) shows the logo, the website home's countdown to the festival start (`festStart` in
  `config.dart`) + "Freier Eintritt" note, a grid linking to every section (website nav order, plus Favoriten and
  Locations), and a footer with the social links and Impressum/Datenschutz (`legal_screen.dart`, static Markdown
  copied from the website). Features:
    - **News** (`/api/v1/news`).
    - **Programm** (`/api/v1/programm`) — a flat, pre-sorted list of programm points (`ProgrammPointDTO` = a
      `ProgrammEntryDTO` joined with an `AttractionRefDTO`), grouped per day client-side, with the website's intro,
      act search + date + stage filters, and the upcoming / "Vergangene" split on the backend's `past` flag (`widgets/schedule_list.dart`, shared with Wirtschaft).
    - **Attraction detail** (`attraction_detail_screen.dart`, `/api/v1/attraction/{externalId}`) — shared by Programm,
      Gastro and Wirtschaft: main image, social links, description, files (`/attraction/{id}/files/{fileId}`, opened
      in the browser), additional images, and upcoming entries linking to the **location detail**.
    - **Gastro** — `/api/v1/attraction?type=FOOD`, sorted by Nr. (`externalId`), searched over Nr. + name + operator.
    - **Wirtschaft & Gewerbe** — `/api/v1/event` (EVENT attractions, kept out of `/programm`).
    - **Jahrgangstreffen** — opens straight into the participant list (`GET /api/v1/meetup`). Registration happens
      **only on the website**: the app has no form and does not link to it.
      `MeetupJahrgang` labels mirror the website's `meetup-jahrgang.ts`.
    - **Favoriten** — favorites are **per attraction** (`attraction.externalId`; programm points have no unique id). Toggled via `widgets/favorite_button.dart`, persisted in `shared_preferences`
      (`services/prefs.dart`, exposed as a
      `ValueNotifier`), and mirrored to a per-act FCM topic. `favorites_screen.dart` filters `/api/v1/programm` to the favorited acts.
    - **Festplan** (`festplan_screen.dart`) — a Mapbox map (`mapbox_maps_flutter`, the website's custom style
      `festMapStyleUri`) of the `/api/v1/location` entries with `showInFestplan`; labelled circles in the website's
      palette (`utils/location_labels.dart`) plus a legend; tap opens a sheet with distance, "Google Maps öffnen" and
      a link to the location detail.
    - **Locations** (`/api/v1/location`, name search) and **location detail** (`/api/v1/location/{externalId}`: image,
      map, Google Maps button, distance).
    - **Sponsoren** — `/api/v1/sponsoring` grouped by tier in the website's order, titles from `/sponsoring/type`,
      Gönner as a name list.
    - **Downloads** (`/api/v1/downloads`, by category), **Galerie** (`/api/v1/gallery`, fullscreen swipe viewer
      `widgets/image_viewer.dart`, Facebook video link) and **Über uns** (`/api/v1/ok`: contact, OK members, teams).
- **API paths go through `lib/api/api_paths.dart`**: the `ApiPaths` constants are the single place the app's API versions are pinned — the `lib/api/*_api.dart` clients never spell out `/api/...`. A
  released build keeps calling the versions it was compiled against, so the backend must keep serving them until that build is gone.
- **No code generator**: unlike the website's `rest.ts`, the Dart models under `lib/models/` are **hand-written and must be kept in sync with the Java DTOs** by hand. When a DTO changes, update the
  matching model + its `fromJson`.
- **Onboarding + permissions**: `screens/onboarding_screen.dart` runs once on first launch (gated by an
  `onboarding_complete` flag in `services/prefs.dart`) and requests **notification** + **location** permission.
- **Push (FCM)**: `services/push.dart` requests permission, gets the token, registers it with the backend (`POST /api/v1/push/register`), and subscribes to the `general` topic. Firebase init in
  `main.dart` is **guarded** — the app runs normally until `flutterfire configure` is run (`lib/firebase_options.dart` starts as a throwing placeholder that flutterfire overwrites). Uses options-based
  init, so no `google-services.json` / gms Gradle plugin is needed on Android.
- **Location**: `services/geolocation.dart` (`geolocator` + fail-soft permission handling) powers distance labels on the map and locations list.
- Base URL: `lib/config.dart` reads
  `String.fromEnvironment('API_BASE_URL', defaultValue: 'https://schlierelacht-api.ch')` — mirrors the website's env-override-then-prod-fallback. Cloudflare image URLs use the same account id
  (`cloudflareImageUrl`). The Mapbox **public** token is read from `String.fromEnvironment('MAPBOX_ACCESS_TOKEN')` (same value as the website's
  `MAPBOX_API_TOKEN`).
- Lightweight architecture: plain `http` + `FutureBuilder` (via `widgets/async_view.dart`; no Provider/Riverpod/Bloc). Markdown rendered with `flutter_markdown_plus`; images via `cached_network_image`.
  German formatting in
  `lib/utils/formatting.dart` (`de_CH`, initialized in `main.dart`).
- **Runtime prerequisites** (features stay inert-but-safe until supplied):
    - **Firebase** — run `flutterfire configure` (regenerates `lib/firebase_options.dart`). For iOS push, enable the Push Notifications capability + upload an APNs key in the Firebase console. Full
      step-by-step (client + backend service-account) in `schlierelacht-app/docs/firebase-setup.md`.
    - **Mapbox** — the **public** `pk.*` token via `--dart-define=MAPBOX_ACCESS_TOKEN=...`, and the **secret** downloads token in `android/gradle.properties` → `MAPBOX_DOWNLOADS_TOKEN=sk...` (needed
      to fetch the Android SDK; the Maven repo is configured in `android/build.gradle.kts`). `minSdk` is raised to 23 for Mapbox + Firebase.
- **App launcher icon**: generated by `flutter_launcher_icons` (config in `flutter_launcher_icons.yaml`) from `assets/app_icon.png` — the yellow "Schliere lacht" smiley extracted from the website favicon. Android
  uses an adaptive icon (brand-yellow
  `#FFF03F` background + inset foreground). Regenerate after changing the source with `dart run flutter_launcher_icons`.
  The Android 12+ splash screen draws only the icon's foreground in a circle, so `res/values(-night)-v31/styles.xml`
  set `windowSplashScreenIconBackgroundColor` to the same yellow — without it the foreground's square edges cut the circle.
- Key packages: `http`, `shared_preferences`, `flutter_svg`, `url_launcher`, `geolocator`, `permission_handler`,
  `firebase_core`, `firebase_messaging`, `mapbox_maps_flutter`, `cached_network_image`, `flutter_markdown_plus`, `intl`.
- **Agentic development** (per docs.flutter.dev/ai/get-started): the official `dart-flutter` Claude Code plugin (Dart MCP server `dart mcp-server` + Flutter/Dart skills) is enabled in **both** this
  workspace's
  `.claude/settings.json` and `schlierelacht-app/.claude/settings.json`, so it is available whether Claude Code is
  started here or inside the app. The Flutter team's proactive hot-reload rule lives in `schlierelacht-app/CLAUDE.md`,
  which Claude Code picks up as soon as it works on files in the app folder. Started from the workspace, the Dart MCP
  server's root is the workspace (not a Dart package) — pass `schlierelacht-app` as the root/project for its tools (analyze, tests, pub, `launch_app`).
- Common commands (run from `schlierelacht-app/`):

  ```bash
  flutter pub get
  flutter analyze
  flutter test
  dart format lib test   # page_width 160 from analysis_options.yaml = Android Studio's formatter; always use it
  # Provide the Mapbox public token so the map works; add API_BASE_URL for a local backend:
  flutter run --dart-define=MAPBOX_ACCESS_TOKEN=pk...
  flutter run --dart-define=API_BASE_URL=http://localhost:8080 --dart-define=MAPBOX_ACCESS_TOKEN=pk...  # iOS sim → local backend
  flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8080 --dart-define=MAPBOX_ACCESS_TOKEN=pk...    # Android emulator → local backend
  ```

## Working across the projects

A typical end-to-end change (e.g. a new public-facing field or endpoint):

0. Decide whether the change is breaking (see "API versioning"). If it is, add a new `/api/v<n+1>/<resource>` next to the existing one instead of changing it in place — the consumers then move one at
   a time.
1. Backend: add/modify the DTO and/or `*Endpoint`, plus service and any Flyway migration + jOOQ regen.
2. Build the backend so `typescript-generator` rewrites `schlierelacht-website/shared/types/rest.ts`.
3. Website: consume the updated type / endpoint in the relevant page or component (bump `app/utils/api-path.ts` if the resource was versioned up).
4. Mobile app (if affected): update the matching hand-written model in `schlierelacht-app/lib/models/` and the UI (bump
   `lib/api/api_paths.dart` if the resource was versioned up).
5. Run backend tests, website lint, and `flutter analyze`/`flutter test`.
