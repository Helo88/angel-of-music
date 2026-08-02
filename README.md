# Angel of Music 🎭

> A full-stack musical theatre archive, community, and booking platform.

**Stack**: Next.js (App Router) · TypeScript · ASP.NET Core 8 · SQL Server · EF Core · SignalR · Fabric.js · QuestPDF

---

## What this is

Angel of Music has three pillars:

1. **Archive** — musicals, songs, famous lines, galleries, credits, production histories, venues with accessibility profiles
2. **Community** — a scrapbook studio (designers craft templates, fans fill them with photos and export printable PDFs), threaded forum, live Foyer chat, and the **Tribute of the Month** contest
3. **Booking** — a theatrical calendar with availability levels, mock payment, ticket-stub confirmation, and Google Calendar integration

---

## Project structure (to be scaffolded)

```
angel-of-music/
├── backend/          # ASP.NET Core 8 Web API
│   ├── Controllers/
│   ├── Services/
│   ├── Models/
│   ├── DTOs/
│   └── Data/
├── frontend/         # Next.js App Router + TypeScript
│   ├── app/
│   ├── components/
│   ├── styles/
│   └── public/
└── docs/             # Planning documents
    └── angel-of-music-guide-v17-FINAL.md
```

---

## Build guide

The full plan is in [`docs/angel-of-music-guide-v17-FINAL.md`](docs/angel-of-music-guide-v17-FINAL.md) — a single consolidated document covering data model, API, caching strategy, design system, motion language, and all build phases. Read it before writing any code.

---

## Roles

| Role | Can do |
|---|---|
| Admin | Everything — seeded, never self-registered |
| Fan | Browse, book, build scrapbooks, submit tributes |
| Designer | Everything a fan can do + submit scrapbook templates |

---

## Key decisions

- **Next.js server components** for SEO and ISR caching (archive pages cached weekly, invalidated on admin edits)
- **One Admin** — no sub-admin hierarchy at this stage
- **SignalR** for real-time (not Firebase) — same JWT, same SQL Server
- **Fabric.js v6** for the canvas scrapbook editor
- **QuestPDF Community** licence for PDF export (free)
- **MailKit** for transactional email (free tier / console sink in dev)
- Every paid service has a free alternative — this project costs £0 to run locally

---

## Status

🎨 **Design phase complete** — see `docs/` for the full build guide  
🏗️ **Implementation starting** — Phase 1: backend foundation
