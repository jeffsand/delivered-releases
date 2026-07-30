# Delivered, product spec

Updated 2026-07-30 (1.3.0 candidate: the sending release). Supersedes
the framing in the original MLP plan (2026-07-11, on Jeff's machine);
architecture and decisions from that plan carry forward unchanged and
are restated here so this document stands alone. Version labels below
match shipped app versions: the ship pipeline landed inside the 1.2
series, and 1.3.0 is the release that answers back.

## Thesis

Your Messages history is one of the most valuable personal datasets you
own: years of family life, decisions, jokes, photos, logistics. Apple
gives you almost nothing to do with it, search is shallow, export does
not exist, there is no API, and the data is locked in an undocumented
SQLite file behind Full Disk Access.

**Delivered is the advanced view of Messages that Apple never wrote.**
Search everything in milliseconds. Browse the whole archive as one
continuous timeline. Answer from wherever the archive moved you, without
losing your place. Export any chat as clean Markdown or structured
JSON, in formats designed to drop straight into an LLM context window. And put
the whole thing on a TV as ambient art with memories mode.

The original framing ("a screensaver based on iMessage") is now the
showcase feature, not the identity. Memories mode is the emotional hook
and the demo; the archive tools are the durable daily-use product.

**The sherlock clause.** Apple will probably build some of this someday.
That is validation, not deterrence: the gap exists now, the data is ours
now, and a local-only tool that respects the user is worth having even in
the timeline where Apple ships theirs.

## Product pillars (priority order)

1. **Find**: FTS5 full-text search over every message with filters
   (chat, sender, has-photo). Sub-second over hundreds of thousands of
   rows. Operators (from:/in:/before:/after:/has:) and saved searches
   shipped in v1.1-v1.2.
2. **Browse**: the continuous transcript: one scroll through years,
   Messages' own visual language (clustering, blue/gray bubbles,
   avatars), a year/month jump index. The archive as a place you can
   actually walk through.
3. **Export**: per-chat Markdown (readability-first, Gruber's own
   format) and JSON (ISO8601 dates, typed tapbacks, attachment
   metadata), with a preview-before-save sheet. Explicitly designed as
   LLM context: "drop your family chat into Claude and ask it
   questions." The .delivered archive package is the machine-portable
   whole-library form.
4. **Answer** (1.3.0): the archive moves you, and you can act on it
   without leaving. A composer under every transcript, quick reply on
   Today cards and search hits, Cmd+N for a fresh conversation with
   anyone in Contacts, and Quick Send, a global hotkey panel that sends
   a message from any app without switching. Sending goes through
   Messages.app via Apple Events; the archive itself stays read-only
   forever, and nothing is ever claimed sent until the echoed row
   appears in the library.
5. **Memories**: fullscreen ambient mode: Ken Burns photos, ambient
   video clips, temporally coherent quote bubbles, tapback bursts,
   stats. The showcase, the demo, the reason someone installs it, and
   then the other pillars keep it installed.

## Privacy posture (non-negotiable, and the marketing headline)

100 percent local. No accounts, no analytics, no telemetry. The
ARCHIVE is read-only forever: Delivered reads a snapshot of chat.db
and never writes to it. Sending (v1.3) does not bend that: the
composer asks Messages.app to send via Apple Events, the same door
any automation uses, gated by the macOS Automation permission at
your first send. Delivered is the button, not the pipe; nothing
sends without a human pressing Return, and the MCP server stays
read-only so agents can read but only you can send. The one network
call remains the updater's appcast fetch, user-disableable.
Curation guarantees: nothing outside explicitly included chats appears in
memories mode; hide-forever is one keystroke; exports are user-initiated
files that go where the user says and nowhere else.

## Platform reality (decided, verified)

- iOS cannot read chat.db: no API, no entitlement. The Mac is the only
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
  key, one undo step per slide), via the window UndoManager.
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
  determinate bar; the finale reports what arrived, or that
  everything was already here, and celebrates when the archive's
  reach extends backward.
- The sidebar is the map: a Chats section lists every conversation with
  25+ messages (~400 of ~2,400, an archive never deletes, so the raw
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
  dependencies, zero network) with three tools, search_messages (full
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

## Shipped (1.2 series, 2026-07-17/18), the ship pipeline

- Signed: Developer ID Application (RF22UW723X), hardened runtime,
  inside-out signing including Sparkle's helpers (scripts/sign.sh).
  TCC grants are durable at last.
- Notarized and stapled, app and DMG both, via the "delivered"
  notarytool profile.
- The DMG: cream field, wordmark over the gold title-rule, drag to
  Applications. create-dmg with a generated background
  (scripts/make-dmg-background.swift).
- Unobtrusive auto-updates (the Rogue Amoeba adoption): Sparkle 2.9
  with a custom user driver, never a modal, never at launch.
  Background check + silent download; one quiet dismissible sidebar
  line ("1.2.1 ready · relaunch"); install-on-quit either way; manual
  Check for Updates keeps immediate capsule feedback; automatic checks
  toggleable in Settings > About.
- Permissions window (Settings > Permissions): the single honest map
 , FDA (required) and Contacts (optional), live-probed status, why,
  one button each.
- One-shot releases: ./scripts/release.sh <version> "<notes>", gate,
  bump, build, sign, notarize, DMG, appcast, GitHub release, verify.
  Public home: github.com/jeffsand/delivered-releases (code private).
- PROVEN 2026-07-18: 1.2.0 released and installed from the public DMG
  (Gatekeeper: accepted, Notarized Developer ID); 1.2.1 released; the
  installed 1.2.0 found it silently, staged it, and self-updated to
  1.2.1 on one click of the quiet line. The full loop works.

## Shipped (1.3.0, 2026-07-29), the release that answers back

The archive learned to hold a pen. Everything below was verified
against a live 284k-message library before release.

- Sending, built on one inviolable rule: the archive is read-only
  forever. SendService asks Messages.app to send via Apple Events, the
  same door any automation uses, gated by the macOS Automation
  permission at first send. The lifecycle never lies: a message lifts
  from the composer (with a 2 second Cmd+. catch window), hands to
  Messages, and is only marked delivered when the echoed row appears in
  the library itself. Failures surface amber with one-click retry.
  iMessage first, SMS fallback for text-only phone sends.
- The composer: under every transcript, Messages' own visual language,
  paste-a-photo support, emoji palette, drafts that survive navigation,
  send-scrolls-home. Reply in the very place the archive moved you.
- Quick reply: Today cards and search hits grow a reply button that
  answers in place, with the surrounding messages for context.
- New Message (Cmd+N): a compose window in Mail's lineage, with
  full-archive and full-Contacts people search, so the first text to a
  new plumber starts in Delivered too.
- Quick Send: a global hotkey (default Ctrl+Option+D, recordable in
  Settings) summons a Spotlight-height panel over any app: type a name,
  type the message, Return, back to work. Never activates the app, Esc
  preserves the draft.
- Live sync: a file watcher on chat.db means arriving messages appear
  in about 3 seconds while the app is open, and sends confirm
  themselves. Unread badges from the archive's own read state.
- On This Day, the true edition: each visit to Today past the Lately
  act is that date's edition across every year you have been texting,
  a hero year chosen by substance, vignettes with the conversation
  around each artifact, first-message milestones, and Cmd+G to open
  any date's edition. Chevrons page between days.
- Photos recovery: Messages evicts most attachment files from disk;
  many of those photos still live in the Photos library. Recovery
  matches by original filename and capture window (its own permission,
  asked honestly in Settings) and refills Today, memories, and exports.
- Ambient video clips in memories mode: muted, looped, era-correct.
- Same-human stitching: one person texting from a phone number and an
  email is one sidebar row, one transcript, one search scope,
  automatically (62 stitched families became 83 on the reference
  library).
- Settings, the honest map: four permission rows (Full Disk Access,
  Contacts, Automation, Photos), each live-probed with why and a
  button; Quick Send shortcut recorder; Photos recovery lives where
  the permission does.
- Help, answered the Gruber way: Delivered Help (Cmd+/) is a local,
  instant keyboard cheat sheet, and the project page link sits under
  it. No empty help book.
- Demo mode (DELIVERED_DEMO=1) boots a fully fictitious library for
  every public screenshot; it is hard-gated from real Contacts and
  real sending.
- Release safety: scripts/test-update.sh rehearses the real Sparkle
  update from the currently shipped public build to the local
  candidate before any user can receive it. Proven live: public 1.2.4
  updated itself to the 1.3.0 candidate through the genuine pipeline.
- 83 tests across 22 suites; perf budgets re-verified at 284k messages
  (search 1-2ms, edition build 1.5ms, whole-library stats 56ms).

Decided against, permanently: integrating the imsg CLI at runtime
(learn from its source, never depend on it), tier-3 injection
features (tapbacks, edits, unsend, typing indicators), and any path
that writes chat.db. The MCP server stays read-only: agents read,
humans send.

## Roadmap

Verification remaining (not construction):
- 4K memories perf check (hardware-blocked until a 4K display is
  available).

v2 (the big screen):
- tvOS/iPad companion consuming .delivered archives (iCloud Drive or
  local network sync). The TV is the endgame for memories mode.
- Considered and parked: Stats hero-card ranking, composer attachment
  folding. Taste calls, revisit with real tester feedback.

## Naming and trademark

Delivered (decided 2026-07-11). Never use "iMessage" in the name or
marketing copy; describe as working with "your Messages archive."
Delivered is not affiliated with Apple.

## Decision log

- 2026-07-11: name Delivered; repo private until ship; open source
  planned, final call later; fullscreen ambient not .saver; archive
  format in v1; club-sandquist archived (data separated from code).
- 2026-07-13: repositioned, archive power tool first, screensaver as
  showcase. Recents + contact-resolved titles. Continuous transcript
  replaces month pagination. Export preview before save.
- 2026-07-15: v1.1 construction complete (search operators, hide undo,
  photo drag, export filters). Transcript windowing deferred until All
  Messages demonstrably feels slow on real data, no speculative
  rework of the jump index.
- 2026-07-16: design review fixes (bubble contrast, honest scopes);
  Memories became a mode, not a place (one Chats list, sparkle +
  pinned); thread stitching healed the re-keyed family group; v1.2
  shipped in one push (MCP, copy-as-context, person unification, saved
  searches, idle start, video). Photos-library recovery and ambient
  clips deferred to v1.3 by choice.
- 2026-07-24: sending architecture decided. Apple Events to
  Messages.app only; never write chat.db; never integrate the imsg
  CLI at runtime (its source is a reference, not a dependency); tier-3
  injection features permanently out; no send is claimed delivered
  without the echoed library row.
- 2026-07-29: 1.3.0 gated on Jeff's hands-on sign-off before release.
  Auto-update rehearsal added as a standing pre-release step
  (scripts/test-update.sh) and proven against the live 1.2.4.
