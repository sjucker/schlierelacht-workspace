# Schliere lacht — Workspace

This repository is a **workspace wrapper**. It exists only to host a root
[`CLAUDE.md`](./CLAUDE.md) so that Claude Code can work on all Schliere-lacht
projects at the same time — they are tightly coupled through REST endpoints and
shared DTOs.

The actual projects live in their own Git repositories and are **not**
tracked here (see `.gitignore`). Clone them as siblings inside this directory:

```bash
git clone https://github.com/sjucker/schlierelacht-admin.git
git clone https://github.com/sjucker/schlierelacht-website.git
git clone https://github.com/sjucker/schlierelacht-app.git
```

Resulting layout:

```
schlierelacht-workspace/
├── CLAUDE.md                 # shared guidance (versioned here)
├── README.md                 # this file
├── schlierelacht-admin/      # Spring Boot backend + Vaadin admin UI (own repo)
├── schlierelacht-website/    # Nuxt public website (own repo)
└── schlierelacht-app/        # Flutter mobile app (own repo)
```

| Project                 | Repository                                       | Tech                               |
|-------------------------|--------------------------------------------------|------------------------------------|
| `schlierelacht-admin`   | https://github.com/sjucker/schlierelacht-admin   | Java / Spring Boot / Vaadin / jOOQ |
| `schlierelacht-website` | https://github.com/sjucker/schlierelacht-website | Nuxt 4 / Vue 3                     |
| `schlierelacht-app`     | https://github.com/sjucker/schlierelacht-app     | Flutter / Dart                     |
