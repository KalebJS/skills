---
name: monospace-brutalist-web
description: UI design language using monospace typography throughout, minimal color, and content-first brutalist layout
---

# Skill: monospace-brutalist-web

## Overview

Monospace brutalist web is a design language where monospace font is applied universally — headings, body, captions, navigation, and code alike. There is no typographic hierarchy through font families; hierarchy is expressed through weight, spacing, and case only. The aesthetic is functional, raw, and reader-first. Color is a last resort. Decoration is absence.

Reference implementation: [int8.tech](https://int8.tech)

---

## Core Design Principles

1. **One font family, always monospace.** No mixing of serif and sans-serif. The monospace constraint is the identity.
2. **Content is the only element.** No sidebars, no hero sections, no widgets. The page is a document.
3. **Color means something.** Black on white is the default state. Any deviation from this — a dark highlight, a gray border — carries semantic weight.
4. **Whitespace does the layout work.** Margins and line-height replace decorative dividers and card shadows.
5. **Metadata is minimal.** Author, date (ISO 8601), and read time. Nothing else in the header.
6. **Navigation is positional.** `..` to go back. No hamburger menus. No dropdowns.

---

## Typography System

| Element | Weight | Size | Notes |
|---|---|---|---|
| Post title (`h1`) | Bold (700) | `1.4rem`–`1.6rem` | No uppercase, no tracking adjustments |
| Section heading (`h2`) | Bold (700) | `1rem` | Same size as body; weight alone signals heading |
| Subheading (`h3`) | Bold (700) | `1rem` | Optionally preceded by a blank line |
| Body text | Regular (400) | `1rem` | Monospace, ~65–72ch line length |
| Byline / metadata | Regular (400) | `0.85rem` | Author · read time · date |
| Image caption | Italic (400) | `0.85rem` | Centered beneath image |
| Nav link | Regular (400) | `0.85rem` | Plain text link, no underline decoration |
| Code (inline/block) | Regular (400) | `0.9rem` | Same family — distinguish via background or indent only |

**Font stack:**

```css
font-family: "Courier New", Courier, monospace;
/* Or a modern monospace alternative: */
font-family: "JetBrains Mono", "Fira Code", "Courier New", monospace;
```

**Line height:** `1.65`–`1.8` for body. Monospace fonts compress vertically; generous line-height is mandatory for readability.

---

## Color & Spacing

### Palette

| Role | Value | Usage |
|---|---|---|
| Background | `#ffffff` | Page background |
| Text | `#000000` or `#111111` | All body copy |
| Border (subtle) | `#cccccc` or `#e0e0e0` | Blockquote border only |
| Highlight bg | `#111111` | Inline key-phrase highlight |
| Highlight text | `#ffffff` | Text inside highlight |
| Link | `#000000` + underline | Default link style |
| Visited link | Same or slight gray | Optional |

No accent colors. No brand colors. If color must appear (e.g., a visited-state distinction), use a gray, not a hue.

### Spacing

```css
/* Page rhythm */
--body-max-width: 68ch;
--body-padding: 2rem 1rem;
--paragraph-gap: 1.25em;
--section-gap: 2.5em;
```

Spacing between paragraphs uses `margin-bottom` on `<p>`. Do not use `line-height` inflation as a substitute for spacing.

---

## Layout System

Single column. Centered. No grid system needed.

```css
body {
  max-width: 68ch;
  margin: 0 auto;
  padding: 2rem 1rem;
  font-family: "Courier New", Courier, monospace;
  font-size: 1rem;
  line-height: 1.7;
  color: #111;
  background: #fff;
}
```

**Key layout rules:**
- `max-width` in `ch` units — this scales with font size and preserves measure at any zoom level
- Images are `max-width: 100%` and `display: block; margin: 0 auto`
- No sticky headers. The page scrolls; the nav does not follow.
- No fixed footers.

---

## Component Patterns

### Blockquote

A simple bordered box. No background fill. No left-bar accent. Full border on all four sides.

```css
blockquote {
  border: 1px solid #ccc;
  padding: 1rem 1.25rem;
  margin: 1.5em 0;
  font-style: normal; /* body of quote stays upright */
}

blockquote cite,
blockquote .attribution {
  font-style: italic;
  display: block;
  margin-top: 0.75em;
}
```

```html
<blockquote>
  <p>Language disguises the thought...</p>
  <cite>— Ludwig Wittgenstein, <em>Tractatus Logico-Philosophicus</em></cite>
</blockquote>
```

**Key detail:** The attribution uses an em dash (`—`) inline, not a separate label. Author name in plain text; work title in `<em>`.

---

### Inline Highlight

Used sparingly to mark a key phrase — not for decoration, only when the phrase is the conceptual anchor of a paragraph.

```css
.highlight,
mark {
  background: #111;
  color: #fff;
  padding: 0 0.15em;
  font-style: normal;
}
```

```html
<p>...as Wittgenstein said: <mark>language disguises the thought</mark>.</p>
```

**Rule:** Maximum one highlight per paragraph. Never highlight an entire sentence. The highlight signals a term worth memorizing, not emphasis.

---

### Section Divider

Text-art, not an `<hr>`. The divider is content — it signals a tonal break, not just a layout separator.

```html
<p class="divider">///////////</p>
```

```css
.divider {
  text-align: center;
  color: #555;
  letter-spacing: 0.15em;
  margin: 2.5em 0;
  user-select: none;
}
```

Common divider glyphs: `///////////`, `* * *`, `—`, `∙ ∙ ∙`. Pick one and use it consistently across the site.

---

### Navigation

Minimal. A single back-link rendered as a text symbol, positioned top-left. Date or section context top-right.

```html
<nav class="site-nav">
  <a href="/">‥</a>
  <span class="post-date">2025-03-22</span>
</nav>
```

```css
.site-nav {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 2.5rem;
  font-size: 0.85rem;
}

.site-nav a {
  text-decoration: none;
  color: #111;
}

.site-nav a:hover {
  text-decoration: underline;
}
```

**Key detail:** The back link is `..` or `‥` (Unicode ellipsis `U+2025`). It reads as a filesystem path metaphor — you are navigating a directory, not a website.

---

### Footnotes

Numbered superscripts in body text, with a corresponding reference list at the bottom. Backlinks from each footnote back to the citation.

```html
<!-- In body -->
<p>...as Wittgenstein said<sup><a href="#fn1" id="ref1">1</a></sup>.</p>

<!-- At bottom of article -->
<ol class="footnotes">
  <li id="fn1">
    Wittgenstein, L. (1995). <em>Tractatus logico-philosophicus</em>. Routledge.
    <a href="#ref1" class="backlink">↩</a>
  </li>
</ol>
```

```css
.footnotes {
  font-size: 0.85rem;
  margin-top: 3rem;
  border-top: 1px solid #ccc;
  padding-top: 1rem;
}

sup a {
  text-decoration: none;
  color: #111;
}
```

---

### Image Caption

Italic, centered, monospace — same font as body. No bold. No "Figure X:" prefix.

```html
<figure>
  <img src="wittgenstein.webp" alt="Ludwig Wittgenstein" />
  <figcaption>Ludwig Wittgenstein</figcaption>
</figure>
```

```css
figure {
  margin: 1.5em 0;
}

figure img {
  display: block;
  max-width: 100%;
  margin: 0 auto;
}

figcaption {
  text-align: center;
  font-style: italic;
  font-size: 0.85rem;
  margin-top: 0.5em;
  color: #444;
}
```

---

## CSS Reference Implementation

Complete stylesheet for a minimal article page:

```css
*, *::before, *::after {
  box-sizing: border-box;
}

body {
  font-family: "Courier New", Courier, monospace;
  font-size: 1rem;
  line-height: 1.7;
  color: #111;
  background: #fff;
  max-width: 68ch;
  margin: 0 auto;
  padding: 2rem 1rem;
}

h1 { font-size: 1.5rem; font-weight: 700; margin: 0.5em 0 0.25em; }
h2 { font-size: 1rem; font-weight: 700; margin: 2em 0 0.5em; }
h3 { font-size: 1rem; font-weight: 700; margin: 1.5em 0 0.25em; }

p { margin: 0 0 1.25em; }

a { color: #111; }
a:hover { opacity: 0.6; }

blockquote {
  border: 1px solid #ccc;
  padding: 1rem 1.25rem;
  margin: 1.5em 0;
}

mark {
  background: #111;
  color: #fff;
  padding: 0 0.15em;
}

figure { margin: 1.5em 0; }
figure img { display: block; max-width: 100%; margin: 0 auto; }
figcaption { text-align: center; font-style: italic; font-size: 0.85rem; margin-top: 0.5em; color: #444; }

.site-nav {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  margin-bottom: 2.5rem;
}
.site-nav a { text-decoration: none; color: #111; }

.divider { text-align: center; color: #555; letter-spacing: 0.15em; margin: 2.5em 0; }

.footnotes { font-size: 0.85rem; margin-top: 3rem; border-top: 1px solid #ccc; padding-top: 1rem; }

code { font-family: inherit; background: #f4f4f4; padding: 0 0.2em; }
pre { background: #f4f4f4; padding: 1rem; overflow-x: auto; }
pre code { background: none; padding: 0; }
```

---

## Common Anti-Patterns

| Symptom | Cause | Fix |
|---|---|---|
| Headings feel invisible | `h2`/`h3` same weight as body | Set `font-weight: 700` and add top margin for breathing room |
| Page reads like a terminal dump | Line-height too tight | Set `line-height: 1.65` minimum for monospace body text |
| Highlights everywhere | Overusing `<mark>` | Limit to one per section maximum; treat as a term-of-art marker |
| Blockquote looks like a sidebar | Added background color | Remove background; border-only is the rule |
| Layout breaks on mobile | `max-width` in `px` not `ch` | Use `ch` units so the column scales with font size |
| Font fallback breaks aesthetic | Non-monospace fallback in stack | Every font in the stack must be monospace |
| Navigation feels like a product | Added hover effects, icons | Keep it plain text; underline-on-hover is the maximum decoration |
| Dates feel informal | Using formats like "March 22" | Always use ISO 8601: `YYYY-MM-DD` |
