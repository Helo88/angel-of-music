# Angel of Music — Build Guide v17 (CONSOLIDATED FINAL)

> This document supersedes all previous versions (v1–v16). Every decision appears **once**, in final form.
> Read this file top to bottom before writing any code. Build in the phase order given.
> **Cost rule: every tool, service and package named here is free or free-tier. Nothing in this plan requires payment.**

---

## 0. What we are building

**Angel of Music** — an archive, community, and booking platform for musical theatre. Three pillars:

1. **Archive** — musicals, their songs, famous lines, galleries, credits, origins; productions by production house at specific venues; cast, milestones, statistics.
2. **Community** — a scrapbook studio (designers craft templates, fans fill them with photos and export printable PDFs), a forum with images and threaded replies, a live "Foyer" chat, and the **Tribute of the Month** contest where fans photograph themselves in the muse's pose and The Muse crowns a monthly winner.
3. **Booking** — a calendar of performances with availability levels, mock payment checkout, ticket-stub confirmation, and add-to-calendar.

**Three roles only**: Admin (single, seeded), Fan, Designer. Designers inherit everything fans have.

---

## 1. Tech stack (final)

| Layer | Choice | Notes |
|---|---|---|
| Frontend | **Next.js (App Router) + TypeScript** | Server components for SEO/caching |
| Styling | **SCSS modules** | No Tailwind |
| Animation | **Framer Motion** | Respect `prefers-reduced-motion` |
| Canvas editor | **Fabric.js v6** | Pin to v6 — v5 API differs |
| State | **Zustand** | Player store, UI state |
| Charts | **Recharts** | Admin dashboard only |
| Tables | **@tanstack/react-table** | Admin lists |
| Dates | **date-fns** | Calendar is custom-built, NOT FullCalendar |
| Real-time | **SignalR** (`@microsoft/signalr`) | NOT Firebase |
| Backend | **ASP.NET Core 8 Web API** | Single backend |
| ORM / DB | **EF Core + SQL Server** | LocalDB / Express is free |
| Auth | **ASP.NET Identity + JWT** | Identity roles only |
| PDF | **QuestPDF** (Community licence — free) | `QuestPDF.Settings.License = LicenseType.Community;` at startup |
| Images | **ImageSharp** | Thumbnails, validation |
| Email | **SMTP via MailKit** — Gmail app-password or Mailtrap free tier; console-log sink in dev | No paid service |
| Storage | **Local disk** behind an `IFileStorage` interface | Free; interface allows later swap without rework |

**Integration**: Next.js is only a frontend. It calls the ASP.NET API over HTTP (`API_URL` env). Login sets the JWT in an **httpOnly cookie** via a Next route handler; server components forward it as Bearer; SignalR connects client-side with `access_token`. CORS allows the Next origin. Deployment = two processes behind one reverse proxy, `/api/*` → ASP.NET.

---

## 2. Data model (final)

**Musical**: Id, Title, Slug (unique), Synopsis, Era, PremierDate, HeroImageUrl, BannerImageUrl, AccentHex, AccentSoftHex, AccentDimHex, BgTopHex, BgBaseHex, SpotifyAlbumId (nullable), SourceMaterial (nullable), HistoryText (nullable), **Status { Draft, Published }**
Children (Cascade): **MusicalSong** (TrackNumber, Title, Act, Description, NotesImageUrl), **FamousLine** (Quote, Character, Act, Scene), **MusicalGalleryImage** (ImageUrl, ThumbUrl, Caption?, AltText *(required)*, SortOrder, IsHero), **MusicalCredit** (Role, Name, SortOrder — SortOrder 1 = masthead eyebrow)

**ProductionHouse**: Id, Name, Slug (unique), City, Country, Description, LogoUrl, FoundedYear — *independent entity*
**Venue**: Id, Name, Slug (unique), City, Country, Address, Capacity, Description, ImageUrl, HighlightsText, ConsiderationsText — *independent entity*
Children (Cascade): **VenueAccessibilityFeature** (Category { Mobility, Hearing, Vision, Families, Sensory }, Feature, Details?), **VenueFaq** (Question, Answer, SortOrder)

**Production** — *the association entity* carrying the Musical↔ProductionHouse many-to-many with payload: Id, MusicalId, ProductionHouseId, VenueId, StartDate, EndDate?, IsActive
Children (Cascade): **ProductionCastMember** (ActorName, Role, PhotoUrl, IsFeatured), **ProductionMilestone** (Type, Title, Description, Date), **ProductionStat** (StatType, Value *decimal*, DisplayFormat, Label)
> Do NOT add MusicalProductionHouse or MusicalVenue join tables — Production already expresses both relationships, with history preserved (a transfer to a new theatre = a new Production row).

**Performance**: Id, ProductionId, DateTime, IsBookable, TotalSeats, AvailableSeats, PriceMin, PriceMax, **RowVersion** (concurrency token)
**Booking**: Id, PerformanceId, UserId, SeatsBooked, TotalPrice, BookingReference (unique, `AOM-{year}-{6 uppercase alphanumerics}`), Status { Confirmed, Cancelled }, PaymentStatus { MockPaid }, AmountPaid, CreatedAt

**ApplicationUser** : IdentityUser + DisplayName, AvatarUrl (nullable — optional at registration), Bio. **No Role column** — Identity roles are the single source of truth.

**ScrapbookTemplate**: Id, DesignerId, MusicalId?, Title, PreviewImageUrl, BackgroundImageUrl, **TemplateDataJson** (single source of truth for layout), SlotCount, Status { Draft, Submitted, Approved, Rejected }, ReviewNote?, CreatedAt
> No separate PhotoSlot / DecorativeElement tables. Approved templates are **immutable**.
**Scrapbook**: Id, FanUserId, TemplateId, Title, PreviewImageUrl, IsPublished, CreatedAt
**ScrapbookPhoto**: Id, ScrapbookId, SlotKey, ImageUrl, CropX, CropY, CropZoom, RotationDeg
**ExportLog**: Id, ScrapbookId, ExportedAt (feeds designer insights)

**ForumPost**: Id, UserId, Title, Content, ImageUrl?, PostType { General, ScrapbookShare, Tribute }, ScrapbookId?, CreatedAt
**ForumComment**: Id, PostId, UserId, ParentCommentId? (one reply level), Content, CreatedAt
**ForumPostLike**: Id, PostId, UserId — **unique index (PostId, UserId)**
**Report**: Id, ReporterUserId, TargetType { Post, Comment, Tribute }, TargetId, Reason, Status { Open, Resolved, Dismissed }, CreatedAt

**TributeContest**: Id, Year, Month, StartsAt, EndsAt, WinnerPostId?, Status { Open, Judging, Closed }
**ContestPrize**: Id, ContestId, PrizeType { FreeTicketVoucher, Poster, FoyerFeature, Custom }, Description, Quantity
**Membership**: Id, UserId, ContestId, Type { AngelOfTheMonth }, AwardedAt, ExpiresAt, PerksJson
> Tribute submissions: **unique index (UserId, ContestId)** — one per member per month, enforced in the database, not just the UI.

**ChatMessage**: Id, UserId, Content, SentAt (single public Foyer room)
**Notification**: Id, UserId, Type { CommentOnYourPost, TributeShortlisted, MuseCrownedYou, BookingConfirmed }, PayloadJson, IsRead, CreatedAt

**Cascade rules**: `Cascade` only for owned children listed above. `Restrict` everywhere else (Musical→Production, Production→Performance, Performance→Booking, User→anything, Template→Scrapbook). Deleting a musical with productions, or a performance with bookings, must fail loudly.

---

## 3. Build phases (follow this order)

### Phase 1 — Foundation
Scaffold ASP.NET Core 8 API + EF Core + SQL Server + Identity with JWT; seed three Identity roles (Admin, Fan, Designer) and one Admin account. Swagger with **JWT Bearer security definition**. Signing key in **user-secrets**, never appsettings. Then all entities above + Fluent API config (unique indexes, RowVersion, cascade rules) + `InitialCreate` migration.
*Ask Claude Code to explain the Fluent API and cascade decisions as it writes them.*

### Phase 2 — Core API
Service pattern (controllers delegate to services), DTOs not entities, pagination where lists can grow.
- **Auth**: register (role whitelist `{Fan, Designer}` **validated server-side**; optional avatar upload), login, me, **forgot-password / reset-password** (token emailed).
- **Musicals / Venues / ProductionHouses / Productions**: public GETs, admin CRUD.
- **Performances**: upcoming, by musical, `calendar?productionId&year&month` returning date-grouped performances with `AvailabilityLevel` computed from AvailableSeats/TotalSeats (Good >50%, Medium 20–50%, Low <20%, SoldOut 0).
- **Bookings** (creation lives ONLY here): validate seats, decrement, save in a transaction; on `DbUpdateConcurrencyException` **retry once**, then return 409 "seats just sold out". Generate reference; regenerate once on collision. Send confirmation email.
- **Templates**: browse Approved; `mine`; create (Draft); update (Draft/Rejected only); `submit`; admin `status` (Approved/Rejected + ReviewNote). Approved = immutable.
- **Scrapbooks**: mine, published, create, save, publish, **export PDF**.
- **Forum**: posts (with optional image), comments (+replies), likes, **report**.
- **Tributes**: current contest, submit (one per user per contest), list, admin crown.
- **Files**: authenticated upload, ≤10MB, jpg/png/webp validated by **content** (ImageSharp load), 480px thumbnail generated, served behind `IFileStorage`.
- **Admin dashboard**: single aggregate DTO — occupancy today, week series, pending templates, contest state, recent activity.

**Ownership rule (applies everywhere)**: any endpoint reading or mutating a user-owned record must verify ownership (or Admin) and return 403/404 otherwise.

### Phase 3 — Frontend foundation
Next.js App Router + TypeScript + SCSS modules. Root layout: navbar, breadcrumbs, **persistent player**, notification bell. Auth via httpOnly cookie route handlers. Then landing, archive list, archive detail.

### Phase 4 — Booking
Custom calendar grid + availability filter chips (All/Good/Medium/Low/Sold out) + musical filter; performance page with **"Know before you go"** (venue accessibility grouped by category, FAQs, honest considerations — **required, not optional**); mock checkout; ticket-stub confirmation with Google Calendar link + `.ics` download.

### Phase 5 — Community
Scrapbook studio (designer) and builder (fan), forum, tributes, keepsake wall, profiles (tabbed), SignalR notifications + Foyer.

### Phase 6 — Admin
Dashboard (clickable cards → dedicated pages), musical form (tabbed: Basics / Credits / History & Origins / Songs / Famous Lines / Gallery / Music & Theme), production form, venue form, performances & bookings tables, template review queue, contest & prizes, **reports queue**.

### Phase 7 — Polish
SEO metadata, sitemap, JSON-LD, caching tags, accessibility pass, empty/error states, seed data, demo rehearsal.

---

## 4. Caching (final)

**Tier 1 — Archive family**: `{ next: { revalidate: 604800, tags: [...] } }` — one-week floor **plus** tag invalidation.
Tags: `musicals`, `musical-{slug}`, `venues`, `venue-{slug}`, `houses`, `landing`, `templates`.
Invalidation map (admin mutations run as server actions; call `revalidateTag` on API success):
- Add musical → `musicals` + `landing`
- Edit musical → `musical-{slug}` (+ `musicals`, `landing` if title/hero/status changed)
- Delete/unpublish → `musical-{slug}` + `musicals` + `landing`
- Venue add/edit/delete → `venue-{slug}` + `venues`
- Production/cast/milestone/stat → parent `musical-{slug}` (+ `venue-{slug}`)
- Template approved/rejected → `templates`
`generateStaticParams` pre-builds Published musicals and venues. Secret-protected `POST /api/revalidate` as safety valve.

**Tier 2 — Live**: calendar & availability `revalidate: 60`; "Now Showing" `revalidate: 300`; seat counts near purchase `revalidate: 30`; a booking mutation revalidates `performance-{id}`.
> Caching is never the correctness guard for seats — RowVersion is.

**Tier 3 — Dynamic (`no-store`)**: profiles, my bookings/scrapbooks, notifications, forum feed, contest state while Open/Judging, Foyer, all admin views.

**ASP.NET layer**: OutputCache + ETag on anonymous content GETs; uploaded images with immutable headers (GUID filenames).

---

## 5. SEO

`generateMetadata` per page from DB content; canonical URLs from slugs; Open Graph + Twitter cards using hero/preview images; JSON-LD (**BreadcrumbList** + CreativeWork on musicals, **Event** on bookable performances with startDate/location/offers, **PerformingArtsTheater** on venues); `app/sitemap.ts` + `app/robots.ts`; one `h1` per page; `next/image` everywhere; **AltText required on every gallery upload** (form field).

---

## 6. Design system

**House palette**: `--ink #100E0B` / `--ink2 #171412` (warm candlelit charcoal), ivory `#F5F0E8`, dim `#B8B2A4`, champagne gold `#D9C9A3`, velvet crimson `#8B1A2B`. Gold belongs to all theatres, not to Phantom.

**Per-musical full-vibe theming**: each musical supplies accent + background tokens; the archive page sets them as CSS variables and the whole atmosphere changes (background gradient, accents, glows) with a ~0.6s transition. Phantom = cold midnight blue + pearl silver; Les Mis = maroon/crimson; Cats = umber/amber. Common pages always keep the house palette.

**Typography**: **Cinzel Decorative** — wordmark ONLY. Playfair Display — headings. Cormorant Garamond italic — quotes/taglines. Inter — body/UI.

**Wordmark lockup**: carved treatment (vertical gradient `#F7EFD9 → #D9C9A3 → #9C8A5C`, hairline top-light + dark under-shadow, soft gold glow). **Position: Variant A** — lockup top-left as masthead, muse alone at center.

**Navbar = the proscenium arch**: sticky, highest z-index, **nothing ever renders over it**. The muse image (real marble artwork, no circle/ring/border) rises from the bar's center, base masked into it, footlight glow beneath, shadow for depth. Desktop: lockup + primary links left, muse centre, role links right. Mobile: muse + small lockup left, burger right opening a full navigation drawer (staggered link cascade, muse crowned at top). Admin: slim crimson "Backstage" bar, small unmasked muse, no theatrics.

**Musical sub-navbar**: on archive detail pages, hero shows credit above title and contributors beneath; scrolling past condenses that identity into its own slim sticky bar under the main navbar; scrolling up releases it.

**Listen drawer + gramophone**: drawer is `position:fixed`, top edge = navbar height (`--nav-height`), never scrolls with the page; any click on page content closes it. Player lives in the **root layout** (Zustand store) so music persists across all navigation. While playing with the drawer closed, a spinning vinyl pins top-left below the navbar on every page; click reopens the drawer; hover reveals a ✕ stop chip (mobile: stop inside drawer). Use **Spotify iFrame Embed API** for real play/pause state.

**Archive gallery**: vertical accordion — closed pleats show a sliver of image behind a veil with the title on the band; hover/tap opens one at a time (~0.6s ease), veil lifts, caption rises. Mobile identical, tap-driven.

**Tribute keepsake wall**: scattered rotated ivory keepsake cards, handwritten-style italic names, heart counts as crimson wax-seal pills, candlelight glow; hover straightens and lifts; winner's card gilded with crown and "THE MUSE HAS CHOSEN". **Plain grids forbidden here.**

**Ornamental divider**: hairline with diamond finials, optional letterspaced label, tinted by the current accent.

**Motion language** — each effect owns exactly one role:
- **Glint** — button hover (skewed light sweep ~0.55s) + gold glow; `scale(0.96)` press.
- **Sparkle orbit** — saving: two gold stars + one red rose orbit *outside* the button border via `offset-path` with `offset-rotate: auto`, staggered; button stays gold and readable, label dims.
- **Light rays** — grand success (publishing a musical): rotating conic gradient + twinkles.
- **Roses** — tribute success only.
- **Curtain** — page transitions.
- **Bellows** — gallery accordion.
All wrapped in `useReducedMotion()`.

**Confirmation dialogs, three tiers**: (1) **Grand** — publishing a musical: rays + twinkles, "The house welcomes a new musical", next-step actions (+ Add a production / + Link a venue / View its page). (2) **Standard** — production/venue/template saves: "Saved — the stage is set". (3) **Tribute** — "The Muse has received your tribute" with rising roses.

**Auth dialog**: muse image at top, "Welcome back to the house" / "Take your seat", switch link between login and register, optional avatar upload on register with the line *"You may remain a silhouette in the house — no photo required."*

**Crowned avatar**: ONE shared Avatar component taking `isAngelOfMonth`; when true, a small gold crown badge overlays the corner. Used **everywhere** the member appears — posts, comments, replies, chat, galleries, shortlists.

**Breadcrumbs**: desktop only (hidden on mobile except admin pages), under the navbar, 11px letterspaced uppercase, gold diamond separators, current page plain. Same data emits BreadcrumbList JSON-LD.

**Profiles are tabbed, never long scrolls**: Fan = Tribute / Scrapbooks / Bookings. Designer = Templates / My fan life. Paginate inside tabs past ~10 items.

---

## 7. Scrapbook pipeline (definitive)

**Images in → JSON in the middle → PDF out.**

- **Designer uploads**: one background artwork (JPG/PNG/WebP, A4 portrait, min 1240×1754) + optional decorative elements as **transparent PNGs**. Never PDFs.
- **Designer draws data**: photo slots (shape, position, size, rotation) and text areas (position, font, size, colour) on the Fabric canvas. Template = asset URLs + one layout JSON.
- **Fan fills**: opens template (background + ornaments locked), taps a slot, uploads their photo — clipped to the slot shape via Fabric `clipPath`, pan/zoom to reframe — fills text areas, saves (photo URLs + crop data + texts + preview PNG from `canvas.toDataURL`).
- **PDF is output only**: `GET /scrapbooks/{id}/export` composes background → cropped photos → ornaments → texts via QuestPDF at print resolution; writes an ExportLog row.
- **Validation**: reject non-portrait or low-resolution backgrounds with a friendly message; ornaments must be PNG with alpha.

**Three seed templates** (Approved, owned by Admin, so day-one fans have choices):
1. *Music of the Night* — midnight-blue parchment, silver filigree corners, faded lace-mask motif, gothic **arch** slot + oval **mirror** slot, single crimson rose divider, Cormorant caption area.
2. *At the Barricade* — distressed parchment over deep maroon, red flag sweep, rough wooden-frame rectangular slots, stencil headline block, candle motif footer.
3. *Jellicle Moon* — warm amber night sky, one large **circular moon** slot, three tilted polaroid slots, scattered stars, soft fur-texture border.

**Designer insights** (per approved template): scrapbooks created, hearts earned, PDF exports, 6-week usage sparkline. `GET /api/templates/mine/insights`.

---

## 8. Community & contest

**Tribute of the Month**: fans post a photo mimicking the muse's pose (one hand raised to the sky, a crimson rose held to the chest). Community hearts build the shortlist; the Admin, **in persona as The Muse**, crowns the winner — "The Muse is watching…" during judging, "The Muse has chosen" on crowning. Winner receives that contest's configured prizes (free ticket voucher, poster, foyer feature, custom surprise), the **Angel of the Month** badge on their avatar site-wide, and a landing-page feature for the month.

**One tribute per member per contest** — enforced by unique index; the UI shows three states: not yet (instructions + upload), submitted (banner with their photo, date, hearts, next window), crowned (gilded banner with perks).

**Forum**: posts with optional image, comments with one reply level, likes (one per user), report button.

**Real-time (SignalR)**: `/hubs/notifications` (comment on your post, tribute shortlisted, Muse crowned you, booking confirmed) and `/hubs/foyer` (single public lounge, last 50 messages on join). Authenticate with the same JWT.

---

## 9. NEW — the seven gaps, closed

### 9.1 Email (free)
MailKit + SMTP (Gmail app password or Mailtrap free tier; in development, an `IEmailSender` implementation that writes to console/log). Emails: **booking confirmation** (reference, musical, venue, date, seats, .ics attached), **password reset** (tokenised link, 1-hour expiry), **contest crowning** ("The Muse has chosen you"), **template approved/rejected** (with the review note). All templated in the house style, plain-text fallback. `IEmailSender` interface so the provider can change without touching call sites.

### 9.2 File storage abstraction (free, local)
All uploads go through `IFileStorage` with a `LocalDiskFileStorage` implementation writing to `wwwroot/uploads/{type}/{guid}.{ext}`. Keeps the assessment free and simple while making a future move to blob/S3 a one-class change. Document in the README that local disk does not survive redeploys on ephemeral hosts.

### 9.3 Moderation & safety
- **Report** button on posts, comments, and tributes → creates a Report row; admin **Reports queue** in Backstage with Resolve / Dismiss / Remove content actions.
- Removing a tribute that is currently crowned: the Membership stays (the person was chosen in good faith) but the photo is replaced with a discreet placeholder and the foyer feature is cleared.
- Upload rules shown at the tribute form: your own photo only, no other identifiable people without consent, no offensive content.
- Admin can soft-hide any user content (`IsHidden` flag on ForumPost/ForumComment) rather than hard-deleting, preserving thread structure.

### 9.4 Our own accessibility
- Full keyboard navigation: all interactive controls reachable and operable; visible focus rings in gold (never `outline:none` without replacement).
- Dialogs (auth, celebrations, mobile nav drawer) trap focus, close on Escape, return focus to the trigger.
- The **canvas editor needs a non-canvas fallback path**: a list-based "assign photo to slot 1/2/3" flow usable by keyboard and screen readers, since canvases are inherently inaccessible.
- ARIA: the gramophone is a labelled button with `aria-pressed` state; the accordion uses `aria-expanded` on each pleat; live regions announce booking success and new chat messages.
- Contrast: check ivory-on-charcoal and gold-on-charcoal against WCAG AA; the dim ivory (`#B8B2A4`) must not be used for essential small text on the darkest backgrounds.
- `prefers-reduced-motion` disables spin, sparkle orbit, rays, and parallax; content still conveys state.
- All images require alt text; decorative ornaments use `alt=""`.

### 9.5 Error, empty and loading states (interactive flows)
Design and implement for: upload rejected (wrong type/size/resolution — say exactly why), PDF export failure (retry action, don't lose the scrapbook), booking losing the race for the last seats (409 → "those seats have just gone; here are the next available performances"), payment mock declined (re-enter card), SignalR disconnected (a quiet "reconnecting…" chip, messages queue locally), template submission failure, session expired mid-form (preserve entered data, prompt re-login). Every list has a designed empty state in house voice ("The stage is quiet tonight…"); every async action has a loading state (sparkle orbit for submits, skeletons for content).

### 9.6 Seed data volume (for a convincing demo)
At minimum: **3 musicals** fully populated (10+ songs, 4+ famous lines, 5+ gallery images with alt text, 3+ credits, origins text), **2 production houses**, **3 venues** with full accessibility profiles and 3+ FAQs each, **4 productions** with cast/milestones/stats, **20+ performances** across two months with varied availability levels, **3 seed templates**, **4 published scrapbooks**, **8 forum posts** with comments and replies, **6 tributes** in the current contest with one crowned previous winner, **6 demo users** (admin@aom.demo, fan@aom.demo, designer@aom.demo + 3 community members) with a documented password.

### 9.7 Verification story
- **Backend**: xUnit tests for the three logic-heavy services — booking concurrency (simulate two simultaneous bookings for the last seats and assert exactly one succeeds), availability-level computation boundaries, and tribute one-per-contest enforcement. Plus a smoke test that every endpoint returns the expected status for anonymous / fan / admin callers (authorization matrix).
- **Frontend**: no heavy test framework required for the assessment; instead a written **manual QA checklist** covering the demo path and the error states in 9.5.
- **Demo rehearsal**: landing → Phantom archive (switch musicals to show theming) → play music, close drawer, show persistent gramophone → build a scrapbook → export PDF → book a ticket → ticket stub + add to calendar → submit a tribute → admin: dashboard, crown the winner, review a template. Rehearse once end-to-end before presenting.

---

## 10. Content & copyright

Short famous lines only (a few words) — never full lyrics or long passages. Prefer public-domain or self-generated imagery over official production stills. Write original synopses. This protects the project and makes it feel authored rather than copied.

---

## 11. Scope guard (cut in this order if time runs short)

1. Web Push notifications
2. Private member-to-member messaging
3. Interactive seat picker (quantity booking ships first)
4. Designer template studio → seed templates only
5. Foyer live chat → notifications only
6. Forum tab → scrapbook gallery + comments only
7. Booking cancellation, admin booking views

**Never cut**: landing page, archive with theming, scrapbook builder + PDF export, booking calendar, Tribute contest, accessibility basics.

---

## 12. Working with Claude Code

Build in small increments and ask for a short "what I did and why" after each. Verify the migration runs before moving on. When a page looks wrong, describe the specific problem, not "make it nicer". Keep the seed data rich — a beautiful site with three rows of data still demos thin. Ask it to explain the Fluent API, the caching tags, and the concurrency handling as it writes them — that explanation is where the learning sticks.
