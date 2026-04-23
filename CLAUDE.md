# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
cargo build           # compile
cargo run             # run
cargo test            # run all tests
cargo test <name>     # run a single test by name
cargo clippy          # lint
cargo fmt             # format
```

## Architecture

`hwp2pdf` is a Rust HTTP API that converts HWP/HWPX (Korean document format) files to PDF using the `rhwp` engine.

**Planned tech stack**: Axum (or Actix) web framework, Tokio async runtime, rhwp conversion engine, local filesystem storage.

**Request flow**:
```
POST /convert (multipart HWP/HWPX file)
  → save to temp
  → rhwp: HwpDocument::from_bytes() → render_to_svg_pages() → svg_to_pdf()
  → store result PDF
  → return { file_id, download_url }

GET /files/{file_id}.pdf → stream PDF response
```

**rhwp pipeline** (internal): binary/XML parser → Document Model → SVG-per-page renderer → PDF assembler. The PDF output path in rhwp is not yet fully stabilized; layout breakage on some documents is a known risk.

**Font dependency**: PDF quality depends heavily on Korean fonts. Fonts (Noto Sans KR / Noto Serif KR) must be present at the path rhwp expects (e.g. `/app/fonts/`). Environment differences in font paths cause rendering divergence.

**Scope (initial phase)**: synchronous single-node conversion only — no async worker queues, no distributed processing, no OCR fallback.
