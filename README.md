# enterprise-architecture — a Claude Code Agent Skill

A unified **enterprise & software architecture** skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code),
grounded in four open-source standards instead of one. It helps Claude produce architecture
work the way the standards intend it: **diagrams as code, documentation as code, decisions
as records, and one traceable model underneath.**

🌐 **Live site & eval examples:** https://gauravs19.github.io/enterprise-architecture/

| Framework | Answers | Used for |
|---|---|---|
| **C4 model + Structurizr DSL** | "How is *this system* built?" | Context / Container / Component diagrams |
| **ArchiMate 3.x** | "How does the *enterprise* fit together?" | Capabilities → apps → technology |
| **TOGAF ADM** | "How do we *deliver* the change?" | Engagement structure, roadmaps, portfolio |
| **arc42 + ADR/MADR** | "How do we *write it down*?" | System docs & decision records |

## Why this exists

The open-source ecosystem has excellent single-purpose skills — C4+ArchiMate plugins,
arc42 toolkits, ADR generators — but **none unify all four frameworks, and none cover
TOGAF or architecture *review*.** This skill fills that gap with four modes:

1. **Diagram** — C4 / Structurizr DSL / Mermaid / PlantUML, at the right altitude.
2. **Document** — arc42 sections and ADRs/MADRs, docs-as-code.
3. **Review** — assess a design against ISO/IEC 25010 quality attributes + EA principles,
   returning severity-categorized findings and a verdict.
4. **Model the enterprise** — capability → application → technology, ArchiMate + TOGAF.

A stable URN-style ID scheme (`ea:org:system:kind:name`) keeps one model under every
diagram, doc, and decision so they stay consistent.

## Layout

```
enterprise-architecture/
├── SKILL.md                       # router: pick framework + mode, shared workflow
├── references/
│   ├── c4-structurizr.md          # C4 levels, Structurizr DSL, Mermaid/PlantUML
│   ├── archimate.md               # 6 layers, element catalog, relationships, viewpoints
│   ├── togaf-adm.md               # 10 ADM phases, deliverables, TIME portfolio model
│   ├── arc42.md                   # 12 sections + LEAN/ESSENTIAL/THOROUGH detail knob
│   ├── adr-madr.md                # ADR & MADR templates and conventions
│   └── review-rubric.md           # quality attributes, severity scheme, verdict format
├── assets/templates/              # ADR, arc42 section, Structurizr workspace starters
└── scripts/
    └── ea_lint.py                 # cross-artifact consistency checker
```

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/<you>/enterprise-architecture \
  ~/.claude/skills/enterprise-architecture
```

Claude Code discovers it automatically. Invoke it implicitly ("draw the container diagram
for this service", "write an ADR for choosing Kafka", "review this architecture", "map our
capabilities to our apps") or explicitly via the skill name.

## The consistency linter

`scripts/ea_lint.py` checks the connective tissue between artifacts — undefined EA IDs,
ADR numbering gaps / missing status / dangling "superseded by", vague (unmeasurable)
quality goals, empty arc42 decision sections, and orphan diagram files.

```bash
python scripts/ea_lint.py path/to/docs/architecture
python scripts/ea_lint.py path/to/docs/architecture --strict   # exit 1 on Major+
```

It's advisory — it reports, it doesn't rewrite your intent.

## Credit / prior art

Patterns studied and adapted from the open-source community:
- [kristjanakkermann/archimate-c4-plugin](https://github.com/kristjanakkermann/archimate-c4-plugin) — URN artifact registry, Mermaid C4/ArchiMate generation
- [MSiccDev/arc42-toolkit](https://github.com/MSiccDev/arc42-toolkit) — per-section discover→generate→validate loop, consistency linting, a dedicated review skill
- [bitsmuggler/arc42-c4-...-example](https://github.com/bitsmuggler/arc42-c4-software-architecture-documentation-example) — Structurizr-DSL-as-source-of-truth + ADR docs-as-code pipeline

## License

MIT — see [LICENSE](LICENSE).
