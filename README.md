# UCOS-COM Ω∞ — Complete Implementation Corpus (CIC)

The authoritative engineering source of truth for the Universal Commerce Operating Substrate.

## Layout

```
corpus.yaml                     Manifest: every corpus, every artifact, status, dependency order
schemas/                        JSON Schemas for artifact headers and domain specifications
corpora/CORPUS-NNN-<slug>/      Normative artifacts (Markdown + YAML front matter)
corpora/CORPUS-002-domain/domains/
    registry.yaml               Every constitutional domain: code, bounded context, owner, tier, status
    DOM-<CODE>.yaml             Full 25-section domain specification
tools/validate.py               Corpus validator (headers, dependencies, traceability, domain completeness)
```

## Reading order

Follow the corpora in dependency order (`corpus.yaml` → `order`). CORPUS-000 defines the terms
"normative", "MUST", artifact identity, and the certification model used by every other corpus.

## Validating

```
python3 tools/validate.py            # full validation, non-zero exit on any violation
python3 tools/validate.py --coverage # coverage report: which corpora/domains are specified
```

The validator is the gate defined in `CIC-000-004` (CR-000-004-01): no artifact is normative unless it validates.
