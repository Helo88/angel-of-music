# Angel of Music 🎭

> A full-stack musical theatre archive, community, and booking platform.

**Stack**: Next.js (App Router) · TypeScript · ASP.NET Core 8 · SQL Server · EF Core · SignalR · Fabric.js · QuestPDF

---

## 🎨 Design Files

| Resource | Link |
|---|---|
| **Live UI Prototypes (Claude Design)** | [View all screens ↓](#live-prototypes) |
| **Figma (early wireframes, superseded)** | [Open in Figma](https://www.figma.com/design/FRs2Qm4QZ1jTgN1Y8jcoXF) |
| **Build Guide v19 (Final)** | [docs/angel-of-music-guide-v19-FINAL.md](docs/angel-of-music-guide-v19-FINAL.md) |
| **Implementation Notes (scrapbook spec — overrides guide §7)** | [docs/implementation-notes.md](docs/implementation-notes.md) |

### Live prototypes

All 15 screens, fully responsive, hosted via GitHub Pages:

| Screen | Live link |
|---|---|
| Landing | https://helo88.github.io/angel-of-music/designs/Landing_dc.html |
| Archive | https://helo88.github.io/angel-of-music/designs/Archive_dc.html |
| Musical Detail (Phantom) | https://helo88.github.io/angel-of-music/designs/Musical_Detail_dc.html |
| Booking + Checkout | https://helo88.github.io/angel-of-music/designs/Booking_dc.html |
| Tribute of the Month | https://helo88.github.io/angel-of-music/designs/Tribute_dc.html |
| Community | https://helo88.github.io/angel-of-music/designs/Community_dc.html |
| Thread Detail | https://helo88.github.io/angel-of-music/designs/Thread_Detail_dc.html |
| Auth (Sign in / Sign up) | https://helo88.github.io/angel-of-music/designs/Auth_dc.html |
| Fan Profile | https://helo88.github.io/angel-of-music/designs/Fan_Profile_dc.html |
| Designer Profile | https://helo88.github.io/angel-of-music/designs/Designer_Profile_dc.html |
| Designer Template Studio | https://helo88.github.io/angel-of-music/designs/Designer_Template_Studio_dc.html |
| Scrapbook Builder | https://helo88.github.io/angel-of-music/designs/Scrapbook_Builder_dc.html |
| Admin Backstage | https://helo88.github.io/angel-of-music/designs/Admin_dc.html |
| Back to Top (shared component) | https://helo88.github.io/angel-of-music/designs/BackToTop_dc.html |
| Master preview frame (Desktop/Tablet/Mobile) | https://helo88.github.io/angel-of-music/designs/Angel_of_Music_dc.html |

The Figma file above contains only the early wireframe pass and is superseded by the prototypes linked here — treat these HTML files as the canonical visual and behavioural spec.

---

## What this is

Angel of Music has three pillars:

1. **Archive** — musicals, songs, famous lines, galleries, credits, production histories, venues with accessibility profiles
2. **Community** — a scrapbook studio (designers craft templates/blueprints, fans fill them with photos and export printable PDFs), threaded forum, live Foyer chat, and the **Tribute of the Month** contest
3. **Booking** — a theatrical calendar with availability levels, seat map, mock payment, ticket-stub confirmation, and Google Calendar integration

---

## Project structure (to be scaffolded)



---

## Roles

| Role | Can do |
|---|---|
| Admin | Everything — seeded, never self-registered |
| Fan | Browse, book, build scrapbooks, submit tributes |
| Designer | Everything a fan can do + submit scrapbook blueprints |

---

## Key decisions

- **Next.js server components** for SEO and ISR caching (archive pages cached weekly + on-demand revalidation)
- **One Admin** — no sub-admin hierarchy at this stage
- **SignalR** for real-time (not Firebase) — same JWT, same SQL Server
- **Fabric.js v6** for the canvas scrapbook editor — free-canvas element model (see implementation notes)
- **QuestPDF Community** licence for PDF export (free)
- **MailKit** for transactional email (free tier / console sink in dev)
- **Tribute of the Month** — admin picks a Musical of the Month + uploads its iconic reference pose each month; fans recreate that pose (not a fixed site-icon pose)
- Every paid service has a free alternative — this project costs £0 to run locally

---

## Status

🎨 **Design phase complete** — 15 fully responsive Claude Design prototypes live above, guide v19 finalized
🏗️ **Implementation starting** — Phase 1: backend foundation
