---
title: "Hugo Documentation Template"
date: 2026-04-29
weight: 1
chapter: false
---

# Hugo Documentation Template

Use this template to start a bilingual Hugo documentation site with GitHub Pages deployment, page-bundled assets, and a ready-to-use authoring devcontainer.

## What This Template Provides

- Bilingual English and Vietnamese content structure
- GitHub Pages Actions deployment
- Page-bundled asset conventions
- Legacy `/images/...` compatibility for older content
- Devcontainer tooling for Hugo, diagrams, screenshots, OCR, and image optimization

## Example Sections

{{% children description="true" /%}}

## Authoring Direction

New pages should use page bundles and relative Markdown paths for page-specific media. Shared static assets may still be used when appropriate, but new content should avoid root-relative `/images/...` paths.
