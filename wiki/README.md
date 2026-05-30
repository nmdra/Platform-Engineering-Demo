# FinEdge Platform Engineering Wiki

> An Obsidian-style knowledge base for learning platform engineering concepts.

## Getting Started

### Open in Obsidian (Recommended)

1. [Download Obsidian](https://obsidian.md)
2. Open Obsidian → "Open folder as vault"
3. Select this `wiki/` directory
4. Enable the **Graph View** plugin for visual navigation

### Browse Without Obsidian

All pages are standard Markdown files with `[[wikilinks]]`. You can read them in any text editor, VS Code, or GitHub's file browser — the wikilinks just won't be clickable.

## Wiki Structure

```
wiki/
├── _index.md              # Main Map of Content (start here!)
├── glossary.md            # A-Z definitions
├── concepts/              # Core concepts (Crossplane, K8s, GitOps)
├── tools/                 # Tool guides (LocalStack, ArgoCD, kind)
├── patterns/              # Architecture patterns
├── troubleshooting/       # Issue resolution guides
└── journal/               # Daily learning notes
```

## Conventions

### Page Format
Every page uses:
- **YAML frontmatter** with `tags` and `created` date
- **`# Title`** as the first heading (matches the `[[wikilink]]` name)
- **`## Related`** section at the bottom linking to related pages
- **`[[wikilinks]]`** throughout the content for navigation

### Naming
- File names use **kebab-case**: `infrastructure-as-code.md`
- Wikilink names use **Title Case**: `[[Infrastructure as Code]]`
- The page title (`# Infrastructure as Code`) must match the wikilink name

### Adding New Pages

1. Create a `.md` file in the appropriate directory (`concepts/`, `tools/`, etc.)
2. Add YAML frontmatter with tags and created date
3. Add `[[wikilinks]]` to connect it to existing pages
4. Add the page to `_index.md` (Map of Content)
5. Add a `## Related` section at the bottom

### Tags

Common tags used:
- `concept` — Core concepts
- `tool` — Tool guides
- `pattern` — Architecture patterns
- `troubleshooting` — Issue resolution
- `journal` — Learning notes
- `crossplane`, `kubernetes`, `security`, `gitops` — Topic tags

## Contributing

When you learn something new or solve an issue:
1. **New concept?** → Create a page in `concepts/`
2. **Solved an issue?** → Add to `troubleshooting/`
3. **Daily learning?** → Copy `journal/_template.md` to `journal/YYYY-MM-DD.md`
4. **Always** update `[[wikilinks]]` in related pages
