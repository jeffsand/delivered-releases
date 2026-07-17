# Delivered

**The advanced view of Messages that Apple never wrote.**

Native macOS, 100 percent Swift and SwiftUI. No Electron, one dependency
(GRDB), about 6,000 lines. It snapshots the Messages database
(read-only) into its own local library and turns years of iMessage
history into an archive you can actually use:

- **Today**: what happened lately, plus "on this day" across every
  year. Different every morning.
- **Find**: full-text search over hundreds of thousands of messages in
  milliseconds, with operators like `from:mom`, `before:2024`,
  `has:photo`. Every hit opens the conversation at that day.
- **Browse**: the whole archive as one continuous Messages-style
  transcript with a year/month jump index.
- **Export**: any chat as clean Markdown or JSON, filtered by person or
  date range. Built to drop straight into an LLM context window.
- **MCP server**: `delivered --serve-mcp` gives Claude and any MCP
  client local search and transcript tools over your own archive.
- **Memories**: fullscreen ambient mode with Ken Burns photos and
  era-matched quotes from your own history.
- **Add Messages**: merge another Mac's database. Dedupes, unifies
  people, extends the archive backward.

100 percent local: no network calls, no accounts, no analytics. The
only network traffic the app will ever make is checking this repo for
updates. Syncs itself silently whenever it comes to the front.

Built entirely from the terminal with swift build and a Makefile. Xcode
never opened. macOS 14+.

## Downloads

Signed and notarized DMGs ship on the
[Releases](../../releases) page. Installed copies update themselves via
Sparkle.

## This repository

This is the public home for Delivered: releases, the product spec, and
update feeds. The source lives in a private repository for now; open
sourcing is planned but not final.

- [Product spec](product-spec.md)

Delivered is not affiliated with Apple. It works with your Messages
archive; your data never leaves your Mac.
