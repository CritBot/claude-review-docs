# claude-review documentation

Source for the [claude-review documentation site](https://critbot.github.io/claude-review-docs).

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Deployed to GitHub Pages on every push to `main`.

## Local development

```bash
pip install mkdocs-material
mkdocs serve
```

Then open http://127.0.0.1:8000.

## Build

```bash
mkdocs build
```

Output goes to `site/`.

## Structure

```
docs/
├── index.md                    # Home page
├── getting-started/
│   ├── installation.md
│   ├── quickstart.md
│   └── configuration.md
├── commands/
│   ├── overview.md
│   ├── diff.md
│   ├── pr.md
│   ├── memory.md
│   ├── insights.md
│   └── install-hook.md
├── memory/
│   ├── how-it-works.md
│   ├── using-memory.md
│   └── daemon.md
├── guides/
│   ├── ci-integration.md
│   ├── output-formats.md
│   ├── pipeline.md
│   └── vs-anthropic.md
└── contributing.md
```

## Contributing

Open a PR with your changes. The site builds and deploys automatically when merged to `main`.
