# archive/

Obsolete or superseded research. Preserved for provenance — never deleted.

## What goes here

- Research docs where the tool was deprecated or approach abandoned
- Docs superseded by newer research (link to the replacement)
- Point-in-time snapshots that are no longer accurate but have historical value

## When to archive

- The tool/product evaluated is discontinued
- A newer doc in the same directory covers the same ground
- The `status` field was set to `deprecated` and staleness threshold exceeded

## How to archive

1. Move the file from its original directory to `archive/`
2. Set frontmatter `status: archived`
3. Add a note at the top: `> Archived YYYY-MM-DD. Superseded by [newer doc](../references/newer.md).`
4. Update `index.md` — move entry from original category to Archive section
5. Append to `log.md`: `## [YYYY-MM-DD] archive | Title`
