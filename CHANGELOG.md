# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
