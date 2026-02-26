# START HERE — Instructions for AI

You are given a repository codedump split into multiple volume files.

## Goal
Answer questions by opening the minimum necessary content.

## Format notes
- Use `==== FILE_INDEX ====` first (lines starting with `ENTRY`).
- Then jump to `----- FILE BEGIN -----` with matching `path="..."`.
- For big files, prefer `--- CHUNK BEGIN ---` blocks.


## How to navigate this dump
1) Open `Index.txt` (master index) and use it to locate the right volume.
2) Pick the relevant volume file.
3) Use the per-volume index section to locate the path.
4) Read the exact file content (or required chunks only).
5) Expand cautiously (imports / calls / routes), 1–2 hops unless needed.

## Rules
- Do NOT try to read the entire dump.
- Prefer docs/diagrams/indices if present.
- When answering, cite file paths and the volume filename.

## Files
- Instructions (this): `00_START_HERE.instructions.md`
- Master index: `Index.txt`
- Volumes:
- kristal-docs-v4_20260226_182027_01_02-schemas.txt  —  FOLDER: 02-schemas
- kristal-docs-v4_20260226_182027_02_05-profiles.txt  —  FOLDER: 05-profiles
- kristal-docs-v4_20260226_182027_03_10-examples.txt  —  FOLDER: 10-examples
- kristal-docs-v4_20260226_182027_04_00-overview.txt  —  FOLDER: 00-overview
- kristal-docs-v4_20260226_182027_05_06-integration.txt  —  FOLDER: 06-integration
- kristal-docs-v4_20260226_182027_06_03-reproducibility.txt  —  FOLDER: 03-reproducibility
- kristal-docs-v4_20260226_182027_07_01-core-spec.txt  —  FOLDER: 01-core-spec
- kristal-docs-v4_20260226_182027_99_OTHERS.txt  —  OTHERS (Misc Folders)

