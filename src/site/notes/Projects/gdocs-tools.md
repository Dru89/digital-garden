---
{"dg-publish":true,"permalink":"/projects/gdocs-tools/","tags":["tooling","google-drive"],"updated":"2026-03-19T22:04:30.151-07:00"}
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
  "original_markdown": "... exact markdown generated on fetch ..."
}
```

This captures the state at fetch time. On push, we diff the original against the edited version to determine what changed.

**Sync command**: `gpush --sync Document.md`

1. Read the `.gdoc.json` sidecar
2. Check `modifiedTime` on the Google Doc — if someone edited it since fetch, warn about conflicts
3. Diff original markdown vs edited markdown
4. Map changes to Google Docs API operations
5. Apply changes surgically
6. Update the sidecar with new state

#### Diff and Apply Strategy

Use **section-level detection with sub-paragraph precision**.

**Detection phase**: Diff the markdown at the section level (split on `##` headings) to identify which sections changed. Headings serve as stable structural anchors for alignment. Unchanged sections are skipped entirely — zero API calls, comments fully preserved.

**Precision phase**: Within each changed section, diff at the paragraph level to find which paragraphs changed. Within each changed paragraph, diff the text content to find the specific character ranges that differ. Apply edits at the finest granularity possible.

**Applying edits**: Use the Docs API `batchUpdate` with:
- `deleteContentRange` to remove changed text
- `insertText` to add new text
- Apply all changes **from the end of the document backward** so index shifts don't cascade

**Blast radius per change type**:

| Change | What gets touched | Comments preserved |
|--------|------------------|-------------------|
| Edit text within a paragraph | Only the changed character range | All except those anchored to the changed text |
| Add new paragraph | Insert at the right index | Everything — nothing existing is touched |
| Delete a paragraph | Delete that range | Everything except comments on the deleted text |
| Add a new section | Insert at the right index | Everything |
| Delete a section | Delete that range | Everything except comments on the deleted section |
| Reorder sections | Treated as delete + insert (lossy) | Comments on moved content are lost |

The reorder case is the known limitation. Detecting that content was moved (vs deleted and new content added) requires fuzzy matching with no clear correct answer. Treating it as delete + insert is honest about what the API can do.

#### Conflict Detection

Before pushing, check `modifiedTime` on the Google Doc:
- If unchanged since fetch → safe to push
- If changed → warn the user with who edited it and when
- Require `--force` to push over remote changes
- Future: `gfetch --merge Document.md` for three-way merge

#### Status Command

`gpush --status Document.md` — shows:
- Which sections/paragraphs changed since fetch
- Whether the remote doc has been modified
- Whether there are conflicts

Like `git status` for a Google Doc.

#### Implementation Phases

**Phase A: Sidecar infrastructure**
- Save `.gdoc.json` on fetch with original markdown and document metadata
- `gpush --status` reads sidecar and shows local changes
- Conflict detection (check remote `modifiedTime`)

**Phase B: Section-level sync**
- Diff at section level using heading anchors
- Fetch document structure (element indices) via `documents.get()`
- Apply section-level paves for changed sections only
- Unchanged sections fully preserved

**Phase C: Sub-paragraph precision**
- Within changed sections, diff at paragraph level
- Within changed paragraphs, diff at character range level
- Apply surgical edits with minimal blast radius
- This is the north star

**Phase D: Advanced features**
- `gfetch --merge` for three-way merge
- `gpush --status` showing paragraph-level diffs
- Lock/checkout semantics (leave a comment on the doc: "Being edited offline")
- Undo: `gpush --revert` to restore to the fetched state

#### Key Technical Challenges

1. **Index mapping**: Google Docs uses character offsets for everything. We need to map markdown structure back to doc indices. This requires fetching the full document structure on push.
2. **Backward application**: All edits must be applied from end of document to start, so earlier operations don't shift indices for later ones.
3. **Heading level shift**: Our `--shift-heading-level-by` means `##` in markdown = Heading 1 in Google Docs. The diff/apply logic needs to account for this.
4. **Rich formatting**: Inline formatting (bold, italic, links, colors) within a paragraph is lost when we rewrite that paragraph's text. Sub-paragraph precision minimizes this but can't fully eliminate it.
5. **Images and tables**: These are structural elements that don't round-trip through markdown. Edits near them need careful handling to avoid corruption.
6. **List nesting**: Markdown and Google Docs have different list models. Nested lists are a known pain point for pandoc conversion.

## Notes

### 2026-03-19 — Project created

Set up the gdocs-tools and transcribe repos. Built gfetch, gpush, transcribe, config system, skills, install scripts. Shipped with custom reference.docx and reference.pptx (Lato/Roboto fonts). Used in the [[Projects/Commerce Delta Notes\|Commerce Delta Notes]] project for summit note-taking workflow.

## Tasks
- [ ] Implement Drive folder listing, caching, and `--folder` flag for gpush ➕ 2026-03-19
- [ ] Add shell autocomplete scripts (bash/zsh) for gpush and gfetch ➕ 2026-03-19
- [ ] Phase A: Save `.gdoc.json` sidecar on fetch, add `gpush --status` and conflict detection ➕ 2026-03-19
- [ ] Phase B: Section-level sync for `gpush --sync` using heading anchors ➕ 2026-03-19
- [ ] Phase C: Sub-paragraph precision editing for `gpush --sync` (north star) ➕ 2026-03-19
