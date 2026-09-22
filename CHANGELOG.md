# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- react-patterns.md 1.1.0: an interpolated value that shares a JSX text run with an HTML entity loses the space after it in the production build; write `{value}{" "}word`, and scan for the pattern.
- ui-patterns.md 1.1.0: one exported constant for a label that renders in several places, with identifiers (routes, keys, params, tables) never renamed; the product name as one config value rendered through one casing-pinned component, with a scan for wrong casings and for the bare name inside any uppercasing element, including uppercasing applied through a class helper.
- supabase.md 1.2: a data-row change is a migration too, written as a guarded no-op that says so when already applied, with its seed updated in the same commit; never edit a merged migration.
- ux-writing.md 1.1.0: vendor-neutral help text when connectors are stand-ins (fence any unavoidable vendor-named label in a test); read-only recon that prints every rendered string verbatim before a copy rewrite.
- prompt-engineering.md 1.2.0: audience framing stated once in the shared preamble, never in task prompts, pinned by test.

### Changed
- SKILLS_GUIDE.md section 3: descriptions follow the additions.

### Added
- supabase.md 1.1: applying a migration by SQL Editor paste (the ledger row and the body in one transaction; one atomic DO block; guards that raise; safe to run twice; rehearsing with rollback; never probing read-only access with a write). New gotcha: triggers that read `auth.uid()` see null in the SQL Editor, so a seed can insert a guarded row but cannot update one.
- database-patterns.md 1.2: retiring rows. Soft delete is a per-query filter that leaks into secondary reads (pickers, imports, rollups); one marker only; unique keys still count retired rows; rows with no history are hard deleted behind a guard that raises.
- nextjs.md 1.1.0: stale generated route types after deleting a route (read the error path before judging; rebuild or typegen; the dev types directory is only rewritten by a running dev server). `NEXT_PUBLIC_` values need the literal expression and a rebuild.
- api-security.md 1.1.0: testing authorization. The gate lives in the action, not the page around it; test the real predicate over a fake data layer; cover every tier and assert refusal before side effects; mutate the gate once to prove the test bites; for open endpoints the limits are the whole defense, and in-memory counters are stated honestly.
- ci-cd.md 1.1.0: a content denylist guard for names and identifiers (owner-held gitignored list, path:line output only, skip-not-pass when absent, exemptions printed on every run, tested with a nonsense token).
- CLAUDE.template.md: "Before removing something that works": name both readings of an ambiguous removal before acting, and report a wrong premise.

### Changed
- SKILLS_GUIDE.md section 3: descriptions follow the additions; six reference docs moved out of the "not yet used in a project" list.

### Added
- document-conversion.md 1.0.0 (new Content category): pandoc silently drops every heading from a `.docx` with no "Normal" paragraph style; fix a scratch copy's `word/styles.xml`, never the original. Run `--extract-media=.` from inside the target folder to avoid a nested `media/media/` path. From the antini-studios repo setup (2026-09-07).

### Changed
- vercel-deployment.md 1.1.0: Vercel Password Protection is not included on Pro (needs the Advanced Deployment Protection add-on), Vercel Authentication covers previews only, and the default `.vercel.app` production domain is public. Gate in-app if the add-on is not wanted.
- CLAUDE.template.md: verify the working directory matches the project before the first command of every session.

### Added
- "Engineering Standard" section in CLAUDE.template.md: the bar every change is held to (foundation never simple, no shortcuts to undo later, right way over cheap way, documented). From the jam-city-music-center site build (2026-09-07).

### Changed
- Decision-log bullet in CLAUDE.template.md now states the immutability rule: accepted decisions are never edited except for their Status line; changes get a superseding entry. Lesson from the jam-city-music-center content migration (2026-09-07).

### Added
- Optional "Content Ownership Rules" section in CLAUDE.template.md: where to state which text is owner-authored and off limits, which files live outside the repo, and where plain-English explanations go. Lesson from the jam-city-music-center setup (2026-09-07).

## [1.0.0] - 2026-04-05

### Added
- MIT License
- CHANGELOG
- Generic path references in CLAUDE.template.md for broader usability

### Note
- Repo was previously public without a formal license. Adding MIT now to clarify permissions going forward.
