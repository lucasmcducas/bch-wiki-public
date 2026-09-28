---
pageType: concept
id: concept.llm-wiki-pattern
aliases:
  - Karpathy LLM Wiki
  - LLM Wiki
  - AI-maintained wiki
description: Pattern where an AI agent maintains a structured knowledge base from curated sources
claims:
  - id: claim.llm-wiki.origins
    text: The LLM Wiki pattern was popularized by Andrej Karpathy as a way for AI agents to maintain structured, interlinked knowledge bases from curated source documents.
    status: supported
    confidence: 0.9
    evidence:
      - kind: direct-read
        sourceId: source.psf-llm-wiki
        path: sources/psf-llm-wiki.md
        weight: 1.0
---

# LLM Wiki Pattern

**Popularized by**: Andrej Karpathy

---

An LLM Wiki is a structured, interlinked knowledge base maintained by an AI agent. The pattern works as follows:

- Human curates source documents
- Agent reads sources and discusses key takeaways with human
- Agent creates or updates wiki pages for concepts and entities
- Pages are interlinked with wiki-links
- A table of contents and operation log are maintained
- Sources are archived as immutable records after processing

## Standard structure

```
input/        -- Source documents to be processed
processed/    -- Immutable archived source documents
wiki/         -- Markdown pages maintained by AI
wiki/index.md -- Table of contents
wiki/log.md   -- Append-only operation log
AGENTS.md     -- Instructions for AI on maintenance
```

## Example implementations

- [PSF LLM Wiki](entities/psf-llm-wiki.md) — BCH/PSF/Cash Stack knowledge base (120+ pages)

## Related pages

- [PSF LLM Wiki Project](entities/psf-llm-wiki.md)
- [Source: PSF LLM Wiki](sources/psf-llm-wiki.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
