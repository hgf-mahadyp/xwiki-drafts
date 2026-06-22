## Converting to XWiki format
- Install pandoc:
```bash
brew install pandoc
```

- Run the pandoc command:

```bash
pandoc -f markdown DatadogIntegration.md -t xwiki -o datadogintegration.xwiki
```

