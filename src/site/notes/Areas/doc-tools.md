---
{"dg-publish":true,"permalink":"/areas/doc-tools/","tags":["tooling","google-drive","ai-tooling","work-adjacent"],"updated":"2026-04-27T16:58:18.672-07:00"}
---

## Overview

CLI tools for working with Google Docs from the command line. Download Google Docs/Slides/Sheets to local markdown (`gfetch`), upload markdown to Google Docs with custom styling and multi-tab support (`gpush`), and transcribe audio/video files locally (`transcribe`).



### What exists today

- `gfetch` — Download Google Docs → markdown, Slides → `file.pptx` + `file.slides.md`, Sheets → `file.xlsx` + `file.comments.md`. Comments, multi-tab docs, frontmatter with author/url/tags.
- `gpush` — Upload markdown → Google Docs (with custom pandoc reference doc), markdown → Google Slides, native docx/pptx/xlsx upload. Multi-tab docs, pageless mode, sharing, post-upload frontmatter update.
- `transcribe` — whisper.cpp wrapper for local audio/video transcription with Metal acceleration.
- Config system, skills for OpenCode and Claude Code, install scripts.

## Goals

The north star is a **local-first Google Docs workflow**: fetch a doc, edit it in markdown, push the changes back with minimal loss of comments, formatting, and collaboration state. Think of it like `git` for Google Docs — checkout, edit, commit.

## Roadmap

### 1. Drive & Folder Support

Give users the ability to organize uploads into specific Drive folders by name, with shell autocomplete.

#### Features

- **Folder listing and caching**: `gfetch folders` hits the Drive API, lists all accessible Shared Drives and folders, saves to `~/.local/share/gdocs-tools/folders.json`. This is the only command that hits the API for folder metadata — everything else reads from cache.
- **Folder creation**: `gfetch folders create "New Folder"` (optionally under a parent folder).
- **`--folder` flag on gpush**: Accepts a folder name (resolved from cache), folder URL, or folder ID. `gpush notes.md --folder "Commerce Delta"`.
- **Default folder in config**: `gfetch config set default_folder "My Folder"` so you don't have to pass `--folder` every time.
- **Shared Drive support**: `supportsAllDrives=True` on all upload calls. Handle the `driveId` requirement for Shared Drive destinations.

#### Shell Autocomplete

Completion scripts for bash and zsh that read from the cached `folders.json`:
- Tab-complete folder names for `--folder`
- Tab-complete config keys for `config set`/`config get`
- Ship with the repo, sourced via install script or manually by the user

### 2. Fetch / Edit / Push Cycle

The big feature. Enable round-trip editing of Google Docs through markdown with minimal loss.

#### The Problem

Google Docs are rich structured documents. Markdown is a lossy text representation. A naive round-trip (fetch → edit → pave) destroys comments, suggestions, inline formatting, and collaboration state. The goal is to make the push step surgical — only touching what actually changed.

#### Architecture

**Sidecar state file**: On `gfetch`, save a `.gdoc.json` alongside the markdown:

```json
{
  "file_id": "abc123",
  "fetched_at": "2026-03-19T...",
  "modified_time": "2026-03-15T14:30:00.000Z",
  "original_markdown": "... exact markdown generated on fetch ...",
  "sections": {
    "Introduction": "hash_of_section_content",
    "Architecture": "hash_of_section_content"
  }
}
```

This captures the state at fetch time. The per-section hashes enable content-aware conflict detection — comparing section content rather than relying solely on the document's `modifiedTime` timestamp. On push, we diff the original against the edited version to determine what changed.

**Sync command**: `gpush --sync Document.md`

1. Read the `.gdoc.json` sidecar
2. Fetch the current Google Doc and extract section content per heading
3. Compare each section's remote content against the sidecar hashes — flag only sections that changed both locally and remotely as conflicts (not the whole document)
4. Parse both original and edited markdown to pandoc JSON AST; diff the ASTs at the section level using heading titles as anchors
5. For changed sections: delete the content range between headings, re-insert using the pandoc AST → Docs API request builder (preserves bold, italic, links, lists, heading styles)
6. Unchanged sections are never touched — zero API calls, comments fully preserved
7. Update the sidecar with new state

#### Diff and Apply Strategy

Use **AST-level diffing with section-level application**.

**Detection phase**: Parse both the original and edited markdown through `pandoc -t json` to produce structured ASTs. Diff the ASTs at the section level using heading titles as stable anchors. AST diffing gives structural awareness — "this paragraph was added," "bold was applied to this phrase" — rather than trying to reverse-engineer changes from raw text diffs. Unchanged sections are skipped entirely — zero API calls, comments fully preserved.

**Section boundary detection**: Fetch the Google Doc via `documents.get()`, find each HEADING_1 paragraph, and match by title to the markdown's `##` headings. The range between two headings defines a section. This avoids needing a full element-by-element index mapping — just section boundaries.

**Precision phase**: Within each changed section, delete the content range and re-insert using the pandoc AST → Docs API request builder (`_DocsContentBuilder`). Because the builder generates proper formatting requests (bold, italic, links, lists, heading styles, line spacing), rewriting a section restores all markdown-representable formatting. The only loss is formatting that exists in Google Docs but not in markdown (text color, highlights, manual font overrides).

**Applying edits**: Use the Docs API `batchUpdate` with:
- `deleteContentRange` to remove changed sections
- `insertText`, `updateTextStyle`, `updateParagraphStyle`, `createParagraphBullets` to insert new content with formatting
- Apply all changes **from the end of the document backward** so index shifts don't cascade

**Tables, images, and other non-roundtrippable elements**: Represent with stable passthrough markers in the markdown on fetch (e.g., `<!-- gdoc:element:objectId -->`). On sync, match markers to their Google Docs elements and leave them untouched. The AST builder already skips these element types — the same principle applies in reverse.

**Applying edits**: Use the Docs API `batchUpdate` with:
- `deleteContentRange` to remove changed sections
- `insertText`, `updateTextStyle`, `updateParagraphStyle`, `createParagraphBullets` to insert new content with formatting
- Apply all changes **from the end of the document backward** so index shifts don't cascade

**Tables, images, and other non-roundtrippable elements**: Represent with stable passthrough markers in the markdown on fetch (e.g., `<!-- gdoc:element:objectId -->`). On sync, match markers to their Google Docs elements and leave them untouched. The AST builder already skips these element types — the same principle applies in reverse.

**Blast radius per change type**:

| Change | What gets touched | Comments preserved |
|--------|------------------|-------------------|
| Edit text within a section | Section is deleted and re-inserted with formatting | All except those anchored within the changed section |
| Add new paragraph to a section | Section is re-inserted | Comments on other sections preserved |
| Add a new section | Insert at the right index | Everything — nothing existing is touched |
| Delete a section | Delete that range | Everything except comments on the deleted section |
| Reorder sections | Treated as delete + insert (lossy) | Comments on moved content are lost |

The reorder case is the known limitation. Detecting that content was moved (vs deleted and new content added) requires fuzzy matching with no clear correct answer. Treating it as delete + insert is honest about what the API can do.

> [!note] Sub-paragraph precision
> A future refinement could diff within changed sections at the paragraph or character level to reduce the blast radius further. In practice, section-level sync with AST-generated formatting preserves enough that this is low priority — the main loss is comments anchored to specific text within a section you edited, which is an inherent tradeoff of editing offline.

#### Conflict Detection

Before pushing, fetch the current Google Doc and compare per-section:
- Extract each section's text content, keyed by heading title
- Compare against the sidecar's per-section hashes
- Sections unchanged both locally and remotely → skip (no conflict, no update)
- Sections changed locally but not remotely → safe to push
- Sections changed remotely but not locally → skip (preserve remote changes)
- Sections changed both locally and remotely → conflict — warn the user with what changed and where
- Require `--force` to push over conflicting sections
- Future: `gfetch --merge Document.md` for three-way merge

This is more precise than a `modifiedTime` check — someone adding a comment or editing section 3 doesn't block your push to section 7.

#### Status Command

`gpush --status Document.md` — shows:
- Which sections/paragraphs changed since fetch
- Whether the remote doc has been modified
- Whether there are conflicts

Like `git status` for a Google Doc.

#### Implementation Phases

**Phase A: Sidecar infrastructure**
- Save `.gdoc.json` on fetch with original markdown, per-section hashes, and document metadata
- `gpush --status` reads sidecar and shows local changes (section-level diff)
- Per-section conflict detection (fetch remote, compare against sidecar hashes)

**Phase B: Section-level sync**
- Parse original and edited markdown to pandoc JSON AST; diff at section level using heading titles as anchors
- Fetch document structure via `documents.get()` to find section boundary indices (HEADING_1 paragraphs matched by title)
- For changed sections: `deleteContentRange` + re-insert using `_DocsContentBuilder` (the pandoc AST → Docs API request builder already exists from the multi-tab work)
- Apply backward (last section first) to avoid index cascading
- Unchanged sections fully preserved

**Phase C: Refinements**
- Passthrough markers for tables, images, and other non-roundtrippable elements
- `gpush --status` showing paragraph-level diffs within changed sections
- Lock/checkout semantics (leave a comment on the doc: "Being edited offline")
- Undo: `gpush --revert` to restore to the fetched state
- `gfetch --merge` for three-way merge

#### Key Technical Challenges

1. **Section boundary detection**: Find HEADING_1 paragraphs in the Google Doc and match them by title to the markdown's `##` headings. Edge cases: duplicate heading titles, headings added/removed since fetch.
2. **Backward application**: All section edits must be applied from end of document to start, so earlier deletions/insertions don't shift indices for later ones. *(Approach is clear; implementation is straightforward.)*
3. **Heading level shift**: Our `--shift-heading-level-by` means `##` in markdown = Heading 1 in Google Docs. The AST builder already handles this — sync needs to use the same shift when matching headings. *(Solved.)*
4. **Rich formatting within rewritten sections**: When a section is deleted and re-inserted, inline formatting is regenerated from the pandoc AST (bold, italic, links, lists, code). Only non-markdown formatting (text color, highlights, manual font overrides) is lost. *(Acceptable for a markdown-first workflow.)*
5. **Tables and images**: These don't round-trip through markdown. Passthrough markers preserve them in place; edits near them need careful index handling to avoid corruption.
6. **Pandoc AST → Docs API translation**: The `_DocsContentBuilder` handles this for fresh content. For sync, it needs to insert at arbitrary indices rather than always starting at index 1. *(Minor adaptation of existing code.)*

## Notes

### 2026-03-25 — Multi-tab fixes and AST content builder

Fixed multi-tab Google Docs: corrected API field names (`addDocumentTab`/`updateDocumentTabProperties` instead of the invented `addTab`/`updateTabProperties`), added 50-char tab title truncation, fixed first-tab always being named "Tab 1". Replaced the raw-text `_insert_markdown_into_tab` with a pandoc-AST-based content builder (`_DocsContentBuilder`) that generates proper Docs API requests for headings, bold, italic, links, lists, code blocks, etc. Added heading level shift (-1), 1.15 line spacing, NORMAL_TEXT named styles on body paragraphs. Fixed `set_pageless_mode` to apply per-tab (the API's `updateDocumentStyle` has a `tabId` field that defaults to first tab only).

Revisited the sync/round-trip roadmap based on what we learned. The AST builder is the same translation layer sync needs — the remaining work is section boundary detection, AST diffing, and targeted delete+insert. Deprioritized sub-paragraph precision in favor of section-level sync with AST-generated formatting, which preserves enough for a markdown-first workflow. Updated conflict detection approach to per-section content comparison instead of document-level `modifiedTime`.

### 2026-03-19 — Project created

Set up the gdocs-tools and transcribe repos. Built gfetch, gpush, transcribe, config system, skills, install scripts. Shipped with custom reference.docx and reference.pptx (Lato/Roboto fonts). Used in the [[Projects/Commerce Delta Notes\|Commerce Delta Notes]] project for summit note-taking workflow.

## Tasks
- [ ] Implement Drive folder listing, caching, and `--folder` flag for gpush ➕ 2026-03-19
- [ ] Add shell autocomplete scripts (bash/zsh) for gpush and gfetch ➕ 2026-03-19
- [ ] Phase A: Save `.gdoc.json` sidecar on fetch with per-section hashes, add `gpush --status` and per-section conflict detection ➕ 2026-03-19
- [ ] Phase B: Section-level sync for `gpush --sync` — AST diffing, section boundary detection, backward delete+insert using `_DocsContentBuilder` ➕ 2026-03-19
- [ ] Phase C: Passthrough markers for tables/images, `gpush --revert`, `gfetch --merge` ➕ 2026-03-19
- [ ] Implement `--tabs + --update` for gpush (clear and replace tabs in an existing multi-tab doc) ➕ 2026-03-25
- [x] Update gcat to use docsread.py instead of the old Drive export pipeline (currently calls _export_doc_as_markdown which requires drive.readonly) 🔼 ➕ 2026-04-10 ✅ 2026-04-17
- [-] gcomments is broken without drive.readonly — decide whether to document as limitation, remove, or gate behind custom credentials 🔼 ➕ 2026-04-10
- [-] Test gpush end-to-end with the new public OAuth credentials and scopes 🔼 ➕ 2026-04-10
- [-] Set up GitHub Pages for doc-tools (privacy policy, TOS, homepage) — needed for OAuth verification 🔼 ➕ 2026-04-10
- [-] Submit doc-tools OAuth app for Google verification (sensitive scopes only, no CASA needed) 🔼 ➕ 2026-04-10
- [-] Update README with documented limitations: comments and Drive metadata (author/date) unavailable without drive.readonly custom credentials 🔼 ➕ 2026-04-10
- [-] Tombstone the enterprise doc-tools repo (github.twdcgrid.net) with README pointing to the new public repo and Disney OAuth setup instructions 🔽 ➕ 2026-04-10
- [-] Make the doc-tools repo public once OAuth verification is complete 🔽 ➕ 2026-04-10
- [-] Add scopes to GCP console (drive.file, documents.readonly, presentations.readonly, spreadsheets.readonly) and enable Slides + Sheets APIs 🔼 ➕ 2026-04-10
- [-] Multi-account / per-domain token storage for doc-tools (Option C from open-source planning) 🔽 ➕ 2026-04-10
- [-] Investigate Google Picker API for per-file drive.file grants (would enable comments on arbitrary docs without drive.readonly) 🔽 ➕ 2026-04-10
- [x] Merge `docsread-primary` branch to main in doc-tools — test against a few known docs first ⏳ 2026-04-20 📅 2026-04-24 ➕ 2026-04-17 ✅ 2026-04-17
