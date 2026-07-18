# Delivered -- product spec

Updated 2026-07-16 (v1.2 shipped). Supersedes the framing in the original
MLP plan (2026-07-11, on Jeff's machine); architecture and decisions from
that plan carry forward unchanged and are restated here so this document
stands alone.

## Thesis

Your Messages history is one of the most valuable personal datasets you
own: years of family life, decisions, jokes, photos, logistics. Apple
gives you almost nothing to do with it -- search is shallow, export does
not exist, there is no API, and the data is locked in an undocumented
SQLite file behind Full Disk Access.

**Delivered is the advanced view of Messages that Apple never wrote.**
Search everything in milliseconds. Browse the whole archive as one
continuous timeline. Export any chat as clean Markdown or structured JSON
-- formats designed to drop straight into an LLM context window. And put
the whole thing on a TV as ambient art with memories mode.

The original framing ("a screensaver based on iMessage") is now the
showcase feature, not the identity. Memories mode is the emotional hook
and the demo; the archive tools are the durable daily-use product.

**The sherlock clause.** Apple will probably build some of this someday.
That is validation, not deterrence: the gap exists now, the data is ours
now, and a local-only tool that respects the user is worth having even in
the timeline where Apple ships theirs.

## Product pillars (priority order)

1. **Find** -- FTS5 full-text search over every message with filters
   (chat, sender, has-photo). Sub-second over hundreds of thousands of
   rows. Operators (from:/in:/before:/after:/has:) and saved searches
   shipped in v1.1-v1.2.
2. **Browse** -- the continuous transcript: one scroll through years,
   Messages' own visual language (clustering, blue/gray bubbles,
   avatars), a year/month jump index. The archive as a place you can
   actually walk through.
3. **Export** -- per-chat Markdown (readability-first, Gruber's own
   format) and JSON (ISO8601 dates, typed tapbacks, attachment
   metadata), with a preview-before-save sheet. Explicitly designed as
   LLM context: "drop your family chat into Claude and ask it
   questions." The .delivered archive package is the machine-portable
   whole-library form.
4. **Memories** -- fullscreen ambient mode: Ken Burns photos, temporally
   coherent quote bubbles, tapback bursts, stats. The showcase, the
   demo, the reason someone installs it -- and then pillars 1-3 keep it
   installed.

## Privacy posture (non-negotiable, and the marketing headline)

100 percent local. No accounts, no analytics, no telemetry. Read-only
snapshot of chat.db; never writes to Messages. One honest exception to
"no network calls," arriving with v1.3: the updater fetches the appcast
and DMG from the public delivered-releases repo, and only that -- and
it can be switched off in Settings.
Curation guarantees: nothing outside explicitly included chats appears in
memories mode; hide-forever is one keystroke; exports are user-initiated
files that go where the user says and nowhere else.

## Platform reality (decided, verified)

- iOS cannot read chat.db -- no API, no entitlement. The Mac is the only
  possible home. iOS/tvOS become consumers of the .delivered archive
  (format_version 1, shipped in v1) in a future release.
- Mac App Store is impossible (sandbox forbids Full Disk Access).
  Distribution: Developer ID signed + notarized DMG, direct download.
- A real .saver bundle is off the table for v1 (legacyScreenSaver hosts
  third-party savers under its own TCC identity, no FDA inheritance);
  memories mode is fullscreen-in-app with start-after-idle as a future
  option.

## Architecture summary

Swift + SwiftUI (macOS 14+), GRDB + FTS5, SPM + Makefile, no Xcode ever.
One dependency (GRDB). Importer snapshots chat.db read-only into an
app-owned library.db; everything downstream reads library.db. Decoding
knowledge (typedstream bodies, tapback codes, Apple epoch, contact
matching) is captured in CLAUDE.md's domain-knowledge section and in the
services themselves. The memories sequencer is a pure, seeded, unit-
tested engine reusable by future archive consumers.

## Shipped (v1.0, 2026-07-11 through 2026-07-13)

- Import: full (310k messages in 33s) + incremental; zero typedstream
  decode failures against real data; tapback removal folding.
- Continuous transcript with clustering, iMessage visual language,
  contact avatars, year/month jump index.
- Toolbar search (Cmd+F) -> live FTS5 results with filters.
- Per-chat export: Markdown/JSON with preview sheet, include-photos
  folder option, CLI flags. .delivered archive export.
- Memories mode: full port of the club-sandquist design, group-aware
  wordmark, hide-forever key, Reduce Motion variant.
- Curation: included chats, keyword blocklist, hidden items + unhide.
- Recents sidebar with contact-resolved titles everywhere.
- Stats cards. FDA onboarding + in-app banner. Programmatic app icon.
- Release chain: make dev/release/sign/notarize/dmg. 24 tests.

## Shipped (v1.1, 2026-07-15)

- Search operators: from:, in:, before:/after: (YYYY[-MM[-DD]]),
  has:photo, from:me, quoted values; operator-only queries; the parsed
  interpretation shown above results; unknown names explain themselves.
- Cmd+Z undo for hide, everywhere hide exists (context menu, memories H
  key -- one undo step per slide), via the window UndoManager.
- Drag a photo out of the transcript to Finder (original file).
- Export filters: per-person and date-range on the preview sheet (seeded
  from the chat's real span) and CLI (--from, --after, --before).
  Partial exports declare their slice in the Markdown preamble and a
  filteredTo JSON field.
- VoiceOver: bubbles read as one element; photo and tapback labels.
- Today: the default pane, in two acts. Lately: the last few days with
  anything worth showing (photos, wall-worthy quotes, Today/Yesterday/
  weekday labels). On this day: this calendar day across every earlier
  year, newest first. Both acts draw from conversations (the sidebar's
  message-count floor plus included chats), never the one-off SMS tail;
  QualityFilter, blocklist, and hidden are respected everywhere. A
  daily-rotating stat in serif; every cluster clicks through into the
  transcript at that day. Contacts access is requested automatically
  the first time the library appears, and resolves to full names
  (nickname wins when set).
- Search hardening from live testing: debounced off-main queries,
  generation-counter last-write-wins (stale results can never stick).
- Sync is ambient: the library catches up silently on launch and every
  app activation (at most once a minute), so staleness has no indicator
  because it has no opportunity. Cmd+Shift+R now syncs quietly too,
  reporting in the bottom capsule ("214 new messages" / "Up to date")
  and fading; the full-screen import remains first-run only. Settings >
  About carries a quiet "Last synced" line.
- Add Messages (File menu + --merge-db): merge another Mac's chat.db or
  a copied Messages folder into the library. Chats match by guid,
  messages dedupe by guid (idempotent, cancel-safe), people unify via
  handle keys, attachments found beside the source copy into app-owned
  storage. Progress is content: a serif numeral counting up and a
  determinate bar; the finale reports what arrived -- or that
  everything was already here -- and celebrates when the archive's
  reach extends backward.
- The sidebar is the map: a Chats section lists every conversation with
  25+ messages (~400 of ~2,400 -- an archive never deletes, so the raw
  list is mostly one-off SMS receipts), recency-sorted, contact-titled.
  Syncs and merges badge the chats they touched, Mail-style; visiting
  clears the badge. The long tail stays reachable through search, whose
  hits now open that chat's transcript at that day.
- Search field responsiveness: the filter pickers rebuild per keystroke,
  so they now draw from small cached arrays (sidebar chats, resolved
  contacts) instead of all 2,100 people; operators keep the full reach.
- Thread stitching: Messages re-keys group threads, splitting one human
  conversation across chats (and one person across phone and email
  threads). Right-click a chat, Combine with its same-named sibling: the
  larger thread becomes canonical, every read (transcript, search,
  export, memories, Today) expands to the family, the sidebar shows one
  row ("2 threads, 55,558 messages"), and Separate undoes it. Fully
  non-destructive: message rows keep their real chatIds, so incremental
  imports never notice.
- Video, first stage: transcript videos show cached poster frames with
  a play badge (frame extraction through MediaPipeline, cached like
  photo renditions); click plays in the default player, drag copies the
  original out, and evicted files fall back to the file chip.
- 55 tests; make test works on CLT-only machines.

## Shipped (v1.2, 2026-07-16)

- MCP server mode: delivered --serve-mcp speaks MCP over stdio (zero
  dependencies, zero network) with three tools -- search_messages (full
  operator grammar, unified names), get_transcript (name-matched chat,
  date range, Markdown out), list_chats. Delivered is the bridge
  between the Messages archive and every MCP client. Register:
  claude mcp add delivered --
  /Applications/Delivered.app/Contents/MacOS/Delivered --serve-mcp
- Copy as context: search results (clipboard button) or a whole chat
  (context menu) as clean Markdown under a ~16k-token budget that trims
  history, never the present; the capsule reports the copied size.
- Person unification: one human across every handle. From pickers show
  one row per contact; from: in search, export, and CLI expands to all
  of a person's handles; stats leaders group by human.
- Saved searches in the search bar's star menu; the query is the name.
- Memories starts itself after N idle minutes (optional, Settings >
  Chats > Ambient); any input ends an idle-started session.
- Video, second stage: has:video operator, video posters on Today,
  videos copied as-is in media exports.
- 61 tests.

## Roadmap

v1.1 remaining (verification, not construction):
- Developer ID signing live (cert pending -- Jeff's portal step), then
  notarized DMG builds.
- VoiceOver and keyboard-navigation audit; 4K memories perf check;
  frame-restoration check (docs/hig-checklist.md unchecked items).
- All Messages transcript windowing if the ~284k-row load feels slow.

## Shipped (v1.3, 2026-07-17/18) -- the ship pipeline

- Signed: Developer ID Application (RF22UW723X), hardened runtime,
  inside-out signing including Sparkle's helpers (scripts/sign.sh).
  TCC grants are durable at last.
- Notarized and stapled, app and DMG both, via the "delivered"
  notarytool profile.
- The DMG: cream field, wordmark over the gold title-rule, drag to
  Applications. create-dmg with a generated background
  (scripts/make-dmg-background.swift).
- Unobtrusive auto-updates (the Rogue Amoeba adoption): Sparkle 2.9
  with a custom user driver -- never a modal, never at launch.
  Background check + silent download; one quiet dismissible sidebar
  line ("1.2.1 ready · relaunch"); install-on-quit either way; manual
  Check for Updates keeps immediate capsule feedback; automatic checks
  toggleable in Settings > About.
- Permissions window (Settings > Permissions): the single honest map
  -- FDA (required) and Contacts (optional), live-probed status, why,
  one button each.
- One-shot releases: ./scripts/release.sh <version> "<notes>" -- gate,
  bump, build, sign, notarize, DMG, appcast, GitHub release, verify.
  Public home: github.com/jeffsand/delivered-releases (code private).
- PROVEN 2026-07-18: 1.2.0 released and installed from the public DMG
  (Gatekeeper: accepted, Notarized Developer ID); 1.2.1 released; the
  installed 1.2.0 found it silently, staged it, and self-updated to
  1.2.1 on one click of the quiet line. The full loop works.

v1.3 candidates (deferred deliberately from v1.2):
- Photo recovery via the Photos library: many evicted attachments exist
  in Photos; capture-date matching could refill Today, memories, and
  exports. Needs PhotoKit and its own permission conversation.
- Ambient video clips inside memories mode (muted AVPlayer layers in
  the sequencer) -- the TV dream, done properly.

v2 (the big screen):
- tvOS/iPad companion consuming .delivered archives (iCloud Drive or
  local network sync). The TV is the endgame for memories mode.

## Naming and trademark

Delivered (decided 2026-07-11). Never use "iMessage" in the name or
marketing copy; describe as working with "your Messages archive."
Delivered is not affiliated with Apple.

## Decision log

- 2026-07-11: name Delivered; repo private until ship; open source
  planned, final call later; fullscreen ambient not .saver; archive
  format in v1; club-sandquist archived (data separated from code).
- 2026-07-13: repositioned -- archive power tool first, screensaver as
  showcase. Recents + contact-resolved titles. Continuous transcript
  replaces month pagination. Export preview before save.
- 2026-07-15: v1.1 construction complete (search operators, hide undo,
  photo drag, export filters). Transcript windowing deferred until All
  Messages demonstrably feels slow on real data -- no speculative
  rework of the jump index.
- 2026-07-16: design review fixes (bubble contrast, honest scopes);
  Memories became a mode, not a place (one Chats list, sparkle +
  pinned); thread stitching healed the re-keyed family group; v1.2
  shipped in one push (MCP, copy-as-context, person unification, saved
  searches, idle start, video). Photos-library recovery and ambient
  clips deferred to v1.3 by choice.
