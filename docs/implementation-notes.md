# Implementation notes from the HTML prototypes

**Audience:** Claude (planning) and Claude Code (implementation).
**Status:** these notes *extend* `angel-of-music-guide-v19-FINAL.md`. Where the guide and this file disagree about the scrapbook, **this file wins** — the prototypes went considerably deeper than §7 of the guide. Everything else in the guide stands.

Prototype files to read before building: `Scrapbook Builder.dc.html`, `Designer Profile.dc.html`, `Designer Template Studio.dc.html`, `Fan Profile.dc.html`. They are the visual and behavioural spec; this file is the translation into data and endpoints.

---

## 1. The core idea the guide does not state

A scrapbook is **not** a form with fixed photo slots. It is a **free canvas**: every element has a position and a size, both stored as **percentages of the page box**, and every element is owned by either the **designer** or the **fan**.

That single `owner` field drives the entire product:

| owner | designer's view | fan's view |
|---|---|---|
| `Design` | solid tinted wash, warm border, editable by the designer | rendered as final content, **not** selectable, not movable, not editable |
| `Fan` | blue dashed wash, empty | the only thing they can touch — an empty frame to drop a photo into, or a dashed line to write on |

A blueprint is therefore just *a scrapbook whose fan-owned elements are still empty*. Do not model blueprints and books as two unrelated shapes — a book is created by cloning a blueprint's element list.

---

## 2. Data model (replaces/extends guide §7)

```
Blueprint
  Id, DesignerId (FK AspNetUsers), Title, Musical (FK, nullable),
  Mood (short line), Status: Draft | InReview | Live | SentBack,
  ModeratorNote (nullable, required when SentBack),
  PaperCss, InkHex, AccentHex, SoftHex, TintHex,
  PageCount, FanSlotCount (computed), UseCount, AvgRating,
  SubmittedAt, DecidedAt, CreatedAt, UpdatedAt

BlueprintPage           Id, BlueprintId, Ordinal, Name ("Act One"), PaperCss (nullable override)
ScrapbookElement        Id, PageId (BlueprintPage or ScrapbookPage), Kind, Owner, X, Y, W, H, Z,
                        Text, FontStyle, FontSizePx, InkHex, Label, ShapeKey, MediaId (nullable)

Scrapbook               Id, FanId, BlueprintId, Title, Status: Draft | Published,
                        PaperCss/InkHex overrides, CreatedAt, PublishedAt
ScrapbookPage           Id, ScrapbookId, Ordinal, Name
BlueprintReview         Id, BlueprintId, FanId, Stars 1-5, Body, DesignerReply (nullable),
                        RepliedAt, CreatedAt   -- unique (BlueprintId, FanId)
ModerationEvent         Id, BlueprintId, AdminId, FromStatus, ToStatus, Note, CreatedAt
```

### Element kinds (exact list from the prototype)

- `photo` — a framed image slot. Design-owned: shows the designer's image. Fan-owned: dashed empty frame, drop target.
- `text` — editable prose. Carries font (Playfair for headings, Cormorant italic for prose), size, ink hex, and a dashed bottom rule when fan-owned.
- `label` — a fixed small-caps Cinzel title. Always design-owned; this is the "pinned" element type.
- `rule` — a hairline gradient divider. Always design-owned; has no owner badge and cannot be re-assigned.
- `keep` — a **keepsake frame**: a shaped image slot (rose, ticket stub, tragedy/comedy mask, candle, programme, playbill, pressed violet). `ShapeKey` picks the mask: `circle` or `rounded`.
- `shape` — pure decoration, no image: `ribbon`, `tape`, `rule`, `circle` (mount), `frame`, `seal` (wax). Each has a fixed CSS recipe — copy them verbatim from `Scrapbook Builder.dc.html`.

### Geometry rules

- `X, Y, W, H` are percentages (decimal). Move clamps to `x ∈ [-1, 99-w]`, `y ∈ [-2, 100-h]` — the small negatives are deliberate, they let a title sit above the page rule.
- Resize clamps to a minimum of `w ≥ 8`, `h ≥ 5`.
- Page height is a fixed aspect the client renders (prototype uses 700px, 760px for the Cats layout) — store the intended page aspect ratio on the blueprint so the PDF matches the screen.
- `Z` is the element's index in its page list; selecting an element lifts it visually only, not in storage.

---

## 3. Behaviour spec (the builder)

**Two modes, one screen.** A `Fan preview / Designing` toggle. Not two routes — the same canvas re-rendered.

Designing mode:
- Selected element gets a dashed outline plus three handles: a **grip pill above** (drag to move), a **corner handle bottom-right** (drag to resize), and an **✕ top-right** (delete). All pointer-events based — must work with touch (`touch-action: none`, `pointermove`/`pointerup` on `window`, not the element).
- Every element (except `rule`) shows an **owner badge top-left**: "Design" (brown `#7A6132`) or "Fan fills" (blue `#3F6076`). **Tapping the badge flips ownership.** There is no separate control for this.
- An **"New element belongs to"** toggle above the add buttons decides the owner of the *next* element added. Default `Design`.
- Add buttons: `+ photo slot`, `+ text`, `+ fixed title`, `apply blueprint layout`.
- **`apply blueprint layout`** seeds the current page with a stock spread (Phantom / Cats / Les Mis — the `seeds` object in the prototype). Seeding is a one-shot copy: afterwards the elements are ordinary and fully editable. Nothing stays bound to the blueprint.
- Shapes and keepsakes are dragged from the tray with a **pointer-drag ghost** (an element following the cursor), not HTML5 drag-and-drop, so it works on touch. Dropping over the canvas converts client coords to percentages.
- Paper and ink: six preset swatches **plus a full colour input**. Ink applies to the *selected* element; if nothing is selected, show the "Select an element first" toast. Paper applies to the page.

Fan preview mode:
- No handles, no owner badges, no add buttons, no page management, no property panel. The tray is replaced by a single explanatory card.
- Fans can only: switch pages, drop a photo into a fan-owned frame, type on a fan-owned text line.
- Attempting to delete a design-owned element raises the toast "That piece is part of the designer's layout."

**Pages:** start with one blank page. Tabs along the top. `+ page` appends. **Long-press (500 ms) or double-click a tab to rename inline**; Enter or blur commits. `← move` / `move →` reorder; `delete` removes (blocked at one page).

**Publish/draft:** publishing seals the layout — while `Published`, drag/resize/delete are all disabled and design-owned text is read-only. "Save draft" returns to editable. Feedback is a single gold toast line under the title bar, auto-dismissed after ~2.6s.

**Empty page state:** centred italic line — "A blank page. Add a slot, some text, a shape — or apply the blueprint layout."

---

## 4. Designer Profile (new page, `/designers/{slug}`)

Tabs: **Blueprints**, **Reviews**, and **Moderation** (Admin only — role-gated tab on the same page, not a separate Backstage screen).

- Blueprint card grid, each card tinted with its own palette and badged `Live` / `In review` / `Sent back`. Below the grid, a **paper panel** for the selected blueprint rendered in that blueprint's palette, listing pages, fan-slot count, books made, rating, and a "What fans fill" breakdown.
- Visibility: the public sees `Live` only; the owning designer also sees their `InReview` and `SentBack` items in the same grid; Admin sees everything.
- `SentBack` carries a required moderator note, and that note *is* the rejection UX — it renders as the blueprint's description on the designer's own card. No separate message or email thread.
- **Reviews**: one per fan per blueprint (unique constraint), 1–5 stars, one optional designer reply (a reply, not a thread). Only a fan who has completed a book from that blueprint may review it.
- **Moderation queue**: `Publish` → `Live`, `Send back` → `SentBack`, both reversible by Admin. Store each decision as a `ModerationEvent` row so `Undo` can restore the previous status — do not just overwrite `Status`.
- Header actions: `Open studio` (designer's own view) and `Start a book` on the selected blueprint (creates a `Scrapbook` from it and jumps to the builder).

---

## 5. API surface implied by the above

```
GET    /api/designers/{slug}                     profile + stats
GET    /api/designers/{slug}/blueprints          role-filtered by status
GET    /api/blueprints/{id}                      pages + elements
POST   /api/blueprints                           create (Designer)
PUT    /api/blueprints/{id}                      full page/element save (Draft or SentBack only)
POST   /api/blueprints/{id}/submit               Draft|SentBack -> InReview
POST   /api/blueprints/{id}/moderate             Admin: { verdict, note } -> Live|SentBack
POST   /api/blueprints/{id}/moderate/undo        Admin: restore previous status from last event
GET    /api/blueprints/{id}/reviews
POST   /api/blueprints/{id}/reviews              Fan, must own a completed book
POST   /api/reviews/{id}/reply                   Designer, own blueprint only
POST   /api/scrapbooks                           { blueprintId } -> clones elements
PUT    /api/scrapbooks/{id}                      fan may only write fan-owned elements
POST   /api/scrapbooks/{id}/publish
GET    /api/admin/moderation/blueprints          the queue
```

**Server-side guards that matter (do not trust the client):**
1. On a fan's `PUT /scrapbooks/{id}`, reject any change to an element whose `Owner = Design`, and reject any added or deleted element outright. The fan may set only `MediaId` and `Text` on fan-owned elements.
2. A designer may only edit a blueprint in `Draft` or `SentBack`. `Live` blueprints are versioned by copy, never edited in place — existing books must not change under their owners' feet.
3. `FanSlotCount` and `PageCount` are computed server-side from the element list.

---

## 6. Rendering and PDF

The canvas is absolutely-positioned percentage boxes — the same geometry renders identically in the browser, in the fan view, and in the PDF (guide §7 pipeline is unchanged: images in → JSON in the middle → PDF out). The PDF renderer needs the shape CSS recipes and the two image-slot masks (`circle`, `rounded`) to match; take them from the prototype rather than reinventing.

Palette values used throughout (they are per-blueprint columns, seeded with these three):

- Phantom — paper `#F4EFE4→#DAD6CE`, ink `#1C2033`, soft `#5B6076`, accent `#8C93AC`
- Cats — paper `#F8EFD8→#E4CFA2`, ink `#31240C`, soft `#7A6132`, accent `#B98F3A`
- Les Misérables — paper `#F5E9E2→#DCC6BD`, ink `#2A1013`, soft `#7A4048`, accent `#A9525E`

House chrome around the canvas stays the guide's palette (`#100E0B` ink, ivory `#F5F0E8`, champagne `#D9C9A3`).

---

## 7. Still unbuilt

- **Keepsake Wall** (guide Phase 5) — no design exists yet. When it is designed it should read from `ScrapbookElement` where `Kind = keep` and the parent book is `Published`.
- Blueprint versioning UI (see guard 2 above) — the rule is decided, the screen is not designed.
- Designer earnings / paid blueprints — the builder shows a "Free while in preview" badge and a "one day sell them" line. Keep the `Price` column nullable and out of the UI for now.
