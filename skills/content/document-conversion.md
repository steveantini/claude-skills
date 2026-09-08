# Document Conversion

| Field | Value |
|---|---|
| **Version** | 1.0.0 |
| **Last Updated** | 2026-09-07 |
| **Applicability** | Converting `.docx` (and similar office formats) to Markdown or HTML with pandoc |
| **Dependencies** | pandoc |

---

## pandoc Drops Headings from a `.docx` with No "Normal" Style

pandoc silently drops every heading from a `.docx` that defines no "Normal"
paragraph style. Word heading styles inherit from Normal; if Normal is
undefined, every heading converts as a plain paragraph, with no warning.

**Fix:**

1. Make a scratch copy of the `.docx`. Never modify the original.
2. In the scratch copy, add a default Normal style to `word/styles.xml`.
3. Remove duplicate heading style definitions from the same file.
4. Convert the scratch copy.

---

## Run `--extract-media=.` from Inside the Target Folder

Run pandoc with `--extract-media=.` from inside the target folder. Running it
from anywhere else produces a nested `media/media/` path.

```bash
cd path/to/target
pandoc scratch-copy.docx --extract-media=. -o output.md
```
