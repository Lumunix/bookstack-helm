# BookStack Helm Chart – Hacks & Customizations

This chart can optionally enable **BookStack hacks** — unsupported customizations that extend or alter BookStack behavior. Apply at your own risk.

## Contents

- [Mermaid Viewer](#mermaid-viewer)

---

## Mermaid Viewer

The [Mermaid Viewer hack](https://www.bookstackapp.com/hacks/mermaid-viewer/) enables interactive Mermaid diagrams on page view. Mermaid code blocks (language `mermaid` in the editor or Markdown code fences) are rendered with pan/zoom, copy, and other controls.

This is an **unsupported customization**; apply at your own risk. It uses BookStack's Visual Theme System and loads Mermaid.js and Font Awesome from external CDNs.

**Enable in values:**

```yaml
mermaidViewer:
  enabled: true
```

When enabled, the chart installs the hack files into a dedicated theme folder and sets `APP_THEME` accordingly. The default theme name is `mermaid_theme`; you can change it with `themeName` if needed.

**Usage:** Add Mermaid code blocks to pages using the WYSIWYG editor (code block, language `mermaid`) or Markdown (```` ```mermaid ```` ... ```` ``` ````).
