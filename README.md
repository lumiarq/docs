# LumiARQ Documentation

This repository contains the versioned documentation source for [LumiARQ](https://github.com/lumiarq/lumiarq).

## Structure

Each version of the documentation lives on its own branch:

| Branch | Version |
|--------|---------|
| `1.x`  | v1.x (current) |

## Format

Every page is a Markdown file with YAML frontmatter:

```md
---
title: Page Title
description: Short description
section: Section Name
order: 1
draft: false
---

# Page content...
```

## Manifest

Each branch contains a `docs.json` manifest listing all published pages with their metadata. The LumiARQ website reads this manifest to build navigation and discover pages.

## Contributing

To contribute to the docs, open a PR targeting the relevant version branch (e.g. `1.x`).
