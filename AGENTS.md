# AGENTS.md — Hugo Documentation Site Authoring Workflow

## Project Overview

Bilingual (English + Vietnamese) Hugo documentation site using the [hugo-theme-learn](https://github.com/matcornic/hugo-theme-learn) theme with GitHub Pages deployment. The site documents a MarTech multi-agent workshop built on AWS AgentCore services.

- **Hugo version:** 0.134.3 extended
- **Theme:** `hugo-theme-learn` (Git submodule at `themes/hugo-theme-learn`)
- **Theme variant:** `workshop` (dark-blue AWS styling from `static/css/theme-workshop.css`)
- **Config:** `config/_default/hugo.toml` (production), `config/development/hugo.toml` (local)
- **CI/CD:** `.github/workflows/hugo.yml` — builds and deploys to GitHub Pages on push to `main`

---

## Authoring Workflow

### 1. Start Authoring

```bash
# Launch development server
hugo serve -D --disableFastRender --navigateToChanged
```

The site runs at `http://127.0.0.1:1313/`. Use the devcontainer (`.devcontainer/devcontainer.json`) for a fully tooled environment with Hugo, Python diagramming, OCR, and image optimization.

### 2. Content Creation Rules

#### Naming Convention
- Use **lowercase, hyphenated** folder names: `03-my-topic/`
- Section files: `_index.md` (English) + `_index.vi.md` (Vietnamese) in the same folder
- Standard pages: `index.md` (English) + `index.vi.md` (Vietnamese) as folder page bundles
- Never mix spaces or uppercase in folder names

#### Front Matter (REQUIRED for every page)

All content files must use **YAML front matter** (`---` delimiters):

```yaml
---
title: "Page Title"
date: YYYY-MM-DD
weight: N
chapter: false
pre: " <b> N. </b> "
---
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Page title (English for `_index.md`, Vietnamese for `_index.vi.md`) |
| `date` | Yes | Publication date in `YYYY-MM-DD` format |
| `weight` | Yes | Menu ordering (lower = earlier in sidebar). Use section number as weight |
| `chapter` | No | Set `true` only for top-level chapter headers (stylized display) |
| `pre` | No | HTML prefix before menu link. Use `" <b> N. </b> "` for numbered sections |
| `draft` | No | Set `true` for unpublished pages (used in archetype) |
| `menuTitle` | No | Short alternative title for sidebar menu (when title is too long) |

#### Content Structure Pattern

```
content/
  _index.md              # Home page (English)
  _index.vi.md           # Home page (Vietnamese)
  01-section-name/
    _index.md            # Section landing page (English)
    _index.vi.md         # Section landing page (Vietnamese)
    some-page/           # Page bundle (when page needs assets)
      index.md           # English content
      index.vi.md        # Vietnamese content
      _static/
        diagram.svg
        screenshot.png
```

### 3. Bilingual Rules

- **Every page and section must have both languages.** Create paired files in the same directory.
- English uses suffix `.md` (e.g., `_index.md`), Vietnamese uses suffix `.vi.md` (e.g., `_index.vi.md`).
- Both language files must share identical front matter structure except for `title`.
- Translate all visible content (headings, body text, image alt text) but NOT directory names, file names, or code blocks.
- The English version is the canonical/primary version; write it first, then translate to Vietnamese.
- Use `{{% children description="true" /%}}` in the home page to list child sections.

### 4. Assets and Images

#### Page-Bundled Assets (PREFERRED)
- Place page-specific assets in a `_static/` subfolder alongside `index.md`
- Reference with relative Markdown: `![alt](_static/my-image.svg)`
- **Do NOT use root-relative `/images/...` paths** in new content

```text
content/03-my-topic/my-page/
  index.md
  index.vi.md
  _static/
    architecture-diagram.svg
    screenshot.png
```

#### Shared/Global Assets
- Place shared images in `static/images/`
- Reference with root-relative: `![alt](/images/AWS_Logo.svg)`

#### Legacy Path Converter
A custom partial (`layouts/partials/utils/static-image-path-converter.html`) handles legacy `/images/...` paths. New content must not rely on this converter.

### 5. Available Shortcodes

Always prefer these shortcodes over raw HTML for consistent styling:

| Shortcode | Usage |
|-----------|-------|
| `children` | `{{% children description="true" /%}}` — list child pages with descriptions |
| `notice` | `{{% notice info %}}Text{{% /notice %}}` — info/warning/error/tip callout boxes |
| `expand` | `{{% expand "Title" %}}Content{{% /expand %}}` — collapsible sections |
| `button` | `{{< button href="/path" >}}Label{{< /button >}}` — styled link button |
| `mermaid` | `{{< mermaid >}}graph LR; A-->B;{{< /mermaid >}}` — Mermaid diagrams |
| `tabs` / `tab` | `{{< tabs >}}{{< tab name="Tab1" >}}Content{{< /tab >}}{{< /tabs >}}` — tabbed content |
| `attachments` | `{{% attachments /%}}` — list file attachments in page bundle |
| `ref` | `{{< ref "path/to/page" >}}` — link to another Hugo page by source path |
| `ghcontributors` | `{{< ghcontributors "repo-url" >}}` — GitHub contributor list |

### 6. Image and Diagram Authoring

The devcontainer includes these tools. Use them when creating visuals:

- **Mermaid diagrams:** Use the `mermaid` shortcode (rendered client-side, no build dependency)
- **Python diagrams:** `diagrams` library for AWS architecture diagrams. Generate with `python generate_diagram.py` in devcontainer, output PNG/SVG to `_static/`
- **Screenshots:** Capture with whatever tool; optimize with `optipng` or `pngquant` before committing
- **SVG is preferred** for diagrams (scalable, smaller size); PNG for screenshots only

---

## Commit and Review Standards

### Commit Message Convention

Use conventional commits:

```
feat(section-name): add bilingual page for XYZ
fix(assets): correct image paths in ABC
docs: update authoring guidelines
```

### Pre-Commit Checklist

- [ ] Both English and Vietnamese files created for every new page
- [ ] Front matter present with `title`, `date`, and `weight`
- [ ] `draft` removed from front matter for published pages
- [ ] All image paths are relative (`_static/...`) not root-relative (`/images/...`)
- [ ] `hugo serve` runs without errors and page renders correctly in both languages
- [ ] Images optimized (PNGs through `optipng`/`pngquant`, SVGs cleaned)
- [ ] No broken internal links — use `hugo` shortcode `{{< ref >}}` for internal links, never hardcode URLs
- [ ] Section `pre` prefixes are consistent across language pairs

### PR Requirements

- One logical change per PR (one page, one section, one set of edits)
- Describe changes in both languages if the content is bilingual
- Include screenshots of layout changes if modifying shortcodes, partials, or CSS
- Verify deployment preview on GitHub Pages (comment the PR with the preview URL)

---

## Deployment

### Production Build

```bash
hugo --gc --minify --baseURL "$BASE_URL"
```

GitHub Actions (`.github/workflows/hugo.yml`) automatically builds and deploys on push to `main`. The base URL is computed dynamically:
- User/org site repos (`{owner}.github.io`): `https://{owner}.github.io/`
- Project repos: `https://{owner}.github.io/{repo}/`

### Local Build Test

```bash
hugo --gc --minify --baseURL "/"
```

---

## Hugo Operation Reference

| Command | Purpose |
|---------|---------|
| `hugo serve -D --disableFastRender` | Dev server with drafts, navigate-to-changed |
| `hugo new content/03-section/index.md` | Create new page from archetype |
| `hugo --gc --minify` | Production build (garbage collect + minify) |
| `hugo --templateMetrics` | Show template execution times |
| `hugo --printI18nWarnings` | Check for missing translations |

---

## Templating Rules

When modifying layouts, partials, or shortcodes:

1. **Do not modify files in `themes/hugo-theme-learn/`** — create overrides in `layouts/` instead
2. Follow Hugo's lookup order: `layouts/` overrides `themes/hugo-theme-learn/layouts/`
3. Match the directory structure exactly (e.g., override `themes/.../layouts/_default/list.html` at `layouts/_default/list.html`)
4. Always provide both English-language fallbacks in template logic for any localized UI strings
5. Test template changes across both language pages before committing

### Existing Override Map

| Override Path | Replaces Theme | Purpose |
|---------------|---------------|---------|
| `layouts/partials/logo.html` | `partials/logo.html` | AWS logo in sidebar |
| `layouts/partials/custom-footer.html` | Empty hook | Google Analytics |
| `layouts/partials/menu-footer.html` | `partials/menu-footer.html` | Dynamic last-updated date + team links |
| `layouts/_default/list.html` | Theme `_default/list.html` | Image path converter for section pages |
| `layouts/shortcodes/children.html` | Theme `children` shortcode | Enhanced sorting, depth, and styling |
