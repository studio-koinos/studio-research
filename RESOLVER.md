# RESOLVER — Where Does This Doc Go?

Read this decision tree before creating any new document. Start at step 1.

## Decision tree

1. **Is this a viability assessment of a specific tool, product, or service?**
   - Does it answer "should we adopt X?" with a verdict (adopt/defer/reject)?
   - Does it evaluate a specific product against our stack?
   - YES → **evaluations/**
   - NO → continue

2. **Is this a deep dive into how something works, a tech comparison, or architecture analysis?**
   - Does it explain a technology, pattern, or approach in depth?
   - Does it compare alternatives without recommending one specific product?
   - YES → **references/**
   - NO → continue

3. **Is this a step-by-step guide for doing something?**
   - Does it have prerequisites, ordered steps, and verification?
   - Could someone follow it to accomplish a concrete task?
   - YES → **guides/**
   - NO → continue

4. **Is this a survey of a market, category, or tool ecosystem?**
   - Does it map out what exists in a space?
   - Does it include a comparison table across multiple options?
   - YES → **landscapes/**
   - NO → continue

5. **Is this obsolete, superseded, or no longer accurate?**
   - Has the tool been deprecated or the approach abandoned?
   - Has a newer doc replaced this one?
   - YES → **archive/** (move from original location, update index.md)
   - NO → re-read steps 1-4. If genuinely none fit, it's likely a **reference**.

## Disambiguation rules

| If it feels like both... | File in... | Because... |
|---|---|---|
| Evaluation AND reference | evaluations/ | The verdict (adopt/defer/reject) makes it an evaluation |
| Reference AND guide | guides/ | Actionable steps make it a guide |
| Landscape AND evaluation | landscapes/ | Multiple products surveyed = landscape |
| Guide AND evaluation | evaluations/ | Single product focus = evaluation |

## Filing checklist

After deciding the directory:

- [ ] Read that directory's `README.md` to confirm fit
- [ ] Check `index.md` — does a similar doc already exist? Update rather than duplicate.
- [ ] Use kebab-case for filenames: `elevenlabs-voice-api.md`, not `ElevenLabs Voice API.md`
- [ ] Include all required sections for the page type (see CLAUDE.md)
- [ ] Update `index.md` and append to `log.md`
