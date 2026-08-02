# Angel of Music 🎭

> A full-stack musical theatre archive, community, and booking platform.

**Stack**: Next.js (App Router) · TypeScript · ASP.NET Core 8 · SQL Server · EF Core · SignalR · Fabric.js · QuestPDF

---

## 🎨 Design Files

| Resource | Link |
|---|---|
| **Figma Design System & UI Screens** | [Open in Figma](https://www.figma.com/design/FRs2Qm4QZ1jTgN1Y8jcoXF) |
| **Build Guide v17 (Final)** | [docs/angel-of-music-guide-v17-FINAL.md](docs/angel-of-music-guide-v17-FINAL.md) |

The Figma file contains:
- 🎨 **Design System** — colour palette, typography, tokens
- 🏛 **Navbar variants** — Guest, Fan, Designer, Admin (desktop + mobile)
- 🏠 **Landing page** — hero, musical cards, Now Showing section
- 🎭 **Archive page** — Phantom atmosphere, accordion gallery, sub-navbar, songs
- 🗓 **Booking calendar** — availability filter chips, colour-coded performance cards
- 🌹 **Tribute keepsake wall** — scattered ivory cards, crowned winner
- ⚙️ **Admin dashboard** — occupancy bars, week chart, action queue

---

## What this is

Angel of Music has three pillars:

1. **Archive** — musicals, songs, famous lines, galleries, credits, production histories, venues with accessibility profiles
2. **Community** — a scrapbook studio (designers craft templates, fans fill them with photos and export printable PDFs), threaded forum, live Foyer chat, and the **Tribute of the Month** contest
3. **Booking** — a theatrical calendar with availability levels, mock payment, ticket-stub confirmation, and Google Calendar integration

---

## Project structure (to be scaffolded)



---

## Roles

| Role | Can do |
|---|---|
| Admin | Everything — seeded, never self-registered |
| Fan | Browse, book, build scrapbooks, submit tributes |
| Designer | Everything a fan can do + submit scrapbook templates |

---

## Key decisions

- **Next.js server components** for SEO and ISR caching (archive pages cached weekly + on-demand revalidation)
- **One Admin** — no sub-admin hierarchy at this stage
- **SignalR** for real-time (not Firebase) — same JWT, same SQL Server
- **Fabric.js v6** for the canvas scrapbook editor
- **QuestPDF Community** licence for PDF export (free)
- **MailKit** for transactional email (free tier / console sink in dev)
- Every paid service has a free alternative — this project costs £0 to run locally

---

## Status

🎨 **Design phase complete** — Figma file linked above, HTML demos in 
🏗️ **Implementation starting** — Phase 1: backend foundation
