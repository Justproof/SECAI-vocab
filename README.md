# SecAI Vocab

245 security-and-AI terms, each with a short definition and a pre-written four-part explainer card. One JSON file, no build step, no API key, no surprises.

It is designed to be swallowed whole by an AI application: load it, index it, ship it.

## The file

`secai-vocab-terms.json` (~253 KB, UTF-8, single JSON object)

```json
{
  "source": "public/SECAI_glossary_ordered.json",
  "note": "data-term must match term byte-for-byte ...",
  "term_count": 245,
  "terms": [ /* 245 objects */ ]
}
```

| Top-level key | Type | What it is |
| --- | --- | --- |
| `source` | string | Where the glossary was ordered from. Provenance, not a live path. |
| `note` | string | Authoring contract for the shortcut that renders these cards. |
| `term_count` | integer | `245`, and it matches `len(terms)`. Cheap sanity check on ingest. |
| `terms` | array | The goods. See below. |

## A term object

Every one of the 245 entries has exactly these five keys. No optional fields, no nulls, no ragged shapes to defend against.

```json
{
  "term": "artificial intelligence",
  "definition": "The science of creating machines with the ability to develop problem-solving and analysis strategies without significant human direction or intervention.",
  "phase": 1,
  "phase_name": "What AI Is: Core Concepts and Paradigms",
  "explanation": "Plain English: ...\n\nExample: ...\n\nWhy it matters: ...\n\nHook: ..."
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `term` | string | The lookup key. Unique across the file. Verbatim casing and punctuation: mostly lowercase, but proper nouns keep their capitals (`EU AI Act`, `ISO/IEC 42001:2023`, `DevSecOps`). Match byte-for-byte. Do not slugify. |
| `definition` | string | One formal sentence. 36 to 315 characters, median 129. |
| `phase` | integer | `1`–`15`. The learning stage the term belongs to. |
| `phase_name` | string | Human-readable title for that phase. Fully determined by `phase`, so it is safe to denormalize or drop. |
| `explanation` | string | The Explain-it card. Four labeled sections joined by blank lines. 88 to 130 words, always under 150. |

### Parsing `explanation`

Always four sections, always in this order, always separated by exactly `\n\n`, always prefixed with the label and a colon:

1. `Plain English:` the definition without the jargon tax
2. `Example:` one concrete scenario
3. `Why it matters:` the security consequence
4. `Hook:` a single sentence worth remembering

```python
def sections(explanation):
    parts = explanation.split("\n\n")
    return dict(p.split(": ", 1) for p in parts)
```

That parser was run against all 245 entries and returns the same four keys every time, so you can split with confidence rather than hope. The prose also carries no em or en dashes by design, which keeps text-to-speech and downstream formatting tidy.

## Phases

`terms` is already sorted by `phase`, so slicing by stage needs no sort. Within a phase, order is curated rather than alphabetical.

| Phase | Terms | Name |
| ---: | ---: | --- |
| 1 | 13 | What AI Is: Core Concepts and Paradigms |
| 2 | 12 | How Models Are Built: Architectures and Learning Mechanics |
| 3 | 14 | Data Fundamentals: Types, Pipelines, and Preparation |
| 4 | 14 | Data Integrity, Governance, and Provenance |
| 5 | 17 | Interacting with Models: Prompting, APIs, and Retrieval |
| 6 | 20 | Model Quality, Ethics, and Responsible AI Principles |
| 7 | 16 | AI-Specific Runtime Controls: Guardrails, Limits, and Enforcement |
| 8 | 19 | Governance, Risk, and Compliance Frameworks |
| 9 | 11 | Roles, Teams, and Organisational Accountability |
| 10 | 20 | Identity, Access, and Cryptographic Foundations |
| 11 | 34 | Threat Landscape: Attack Vectors and Adversarial Techniques |
| 12 | 21 | Defensive Technologies and Secure Development Practices |
| 13 | 10 | Detection, Monitoring, and Threat Intelligence |
| 14 | 16 | Incident Response, Evaluation, and Knowledge Bases |
| 15 | 8 | MLOps, Continuous Delivery, and Operational Resilience |

## Ingesting it

The whole file is somewhere near 60k to 70k tokens depending on your tokenizer, so it fits in a modern context window in one gulp. That said, most applications do better with one of these three shapes.

**Lookup table.** The fastest thing that works.

```python
import json

data = json.load(open("secai-vocab-terms.json"))
by_term = {t["term"]: t for t in data["terms"]}

by_term["prompt injection"]["explanation"]
```

**RAG chunks.** One term is one chunk. Do not split further, since each entry is already a self-contained 100-word card and cutting it in half throws away the half that explains why anyone should care.

```python
chunks = [
    {
        "id": t["term"],
        "text": f"{t['term']}\n\n{t['definition']}\n\n{t['explanation']}",
        "metadata": {"phase": t["phase"], "phase_name": t["phase_name"]},
    }
    for t in data["terms"]
]
```

Embed `text`, filter on `metadata.phase` when a user is working through a specific stage, and return the matching term object rather than a raw excerpt.

**Tool or function call.** Hand a model a `lookup_term(term)` function backed by `by_term`, and pass `list(by_term)` as the enum so it can only ask for terms that exist. Grounded answers, no invented vocabulary.

```javascript
const data = await fetch(url).then((r) => r.json());
const byTerm = new Map(data.terms.map((t) => [t.term, t]));
```

## Stability

- Field names and the five-key shape are the contract. Treat them as fixed.
- `term` strings are identifiers. If one changes, that is a breaking change, not a typo fix.
- New terms may be appended to a phase. Code that assumes a fixed count will be sad, so read `term_count` instead of hardcoding 245.
- The four `explanation` labels are part of the format, not decoration.

## License and provenance

Derived from the SECAI glossary (`public/SECAI_glossary_ordered.json`). Explanation cards are pre-rendered for the SecAI Vocab shortcut. Check with the repository owner before redistributing.
