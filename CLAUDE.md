# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a static HTML/CSS/JS educational demo suite called **LLM Visual Demos** (`llmfundamentals/`). It contains 14+ self-contained interactive demos teaching LLM fundamentals: tokenization, embeddings, cosine similarity, attention, RAG, agentic patterns, etc. There is no build step, bundler, or package manager — all files are plain HTML with inline `<style>` and `<script>` blocks.

## Architecture

### File layout
- `llmfundamentals/index.html` — landing page with a sticky progress nav linking to all 14 demos
- `llmfundamentals/theme.css` — shared design tokens (CSS custom properties) and common components; **imported first on every demo page**
- `llmfundamentals/index.css` — styles specific to `index.html` only; imported after `theme.css`
- Each demo (e.g. `attention.html`, `tokenizer.html`) is fully self-contained: its own `<style>` block, inline JS, and a single `<link rel="stylesheet" href="theme.css">` for tokens

### Design system
Every page uses two Google Fonts loaded via `<link>` tag:
- `Space Grotesk` (sans-serif body/headings)  
- `Fira Code` (monospace labels/code)

Each demo page sets a `--page-accent` CSS variable in its own `:root` to give it a distinct color identity. All spacing, shadow, radius, and palette values come from `theme.css` custom properties — never use hardcoded values.

### Navigation
The sticky progress nav in `index.html` groups demos into 4 color-coded learning levels (indigo → amber → emerald → cyan). The same nav bar is expected to appear at the top of individual demo pages for wayfinding consistency.

## Development

No build tooling. Open any `.html` file directly in a browser, or serve the folder with any static server:

```bash
# Python
python -m http.server 8080 --directory llmfundamentals

# Node (npx)
npx serve llmfundamentals
```

## Adding a New Demo Page

1. Copy an existing demo as a starting point.
2. Set a unique `--page-accent` color in the page's `:root`.
3. Keep `<link rel="stylesheet" href="theme.css">` as the only external stylesheet.
4. Use only CSS custom properties from `theme.css` for colors, spacing, shadows, and typography.
5. Add a navigation dot to the progress nav in `index.html` (and optionally in other demo pages).
