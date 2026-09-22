# Lymphedema Research Intelligence System

A personal, self-hosted research-tracking system for **primary (genetic / hereditary / developmental) lymphedema**. It continuously monitors the scientific literature and clinical-trial landscape, scores new findings for evidence quality and relevance, synthesizes them into living knowledge-base documents, and serves everything through a local web dashboard.

The system runs locally (built for a Mac Mini), uses [Claude Code](https://claude.com/claude-code) for autonomous research and synthesis, stores structured data as local JSON, and optionally syncs synthesized output to Google Drive.

> **Scope — primary lymphedema only.** Secondary lymphedema (breast-cancer-related, post-surgical, filarial, radiation-induced) is explicitly out of scope. Papers and trials are classified at ingestion and secondary-only items are filtered out.

> **Note:** This is a personal research tool, not medical advice. Findings are automatically summarized and may contain errors — always verify against primary sources and consult a qualified clinician.

---

## How it works

Two scheduled agents (macOS `launchd`) drive the pipeline:

| Agent | Schedule | Purpose |
|-------|----------|---------|
| `com.lymphedema.scan` | Tue / Thu / Sat at 08:00 | Light, fast sweep for new papers and trial updates |
| `com.lymphedema.deep` | Weekly (Sunday) at 09:00 | Comprehensive synthesis and digest generation |

Each agent invokes Claude Code with a mode-specific prompt. A typical run:

1. Fetches papers from PubMed / Semantic Scholar and checks ClinicalTrials.gov for updates
2. Assesses relevance and novelty against the existing knowledge base
3. Scores evidence on five axes (importance, evidence strength, novelty, decision usefulness, claim calibration) plus skepticism flags
4. Updates the structured data store (`data/*.json`)
5. Runs the schema guard (`scripts/normalize_data.py`) and updates synthesis documents, digests, and alerts

```
launchd (scan / deep)
        │
        ▼
   Claude Code  ──► fetch → assess → score → update data → synthesize
        │
        ▼
   Data layer (data/*.json)  ──►  Output (knowledge base, digests, alerts)
        │                                     │
        ▼                                     ▼
   Next.js dashboard  ◄───── reads ─────  Google Drive sync (optional)
```

---

## Research taxonomy

The agent tracks lymphedema research across six dimensions, each mapped to a living knowledge-base document:

1. **Pharmacological treatments** — VEGF-C / lymphangiogenic therapies, mTOR inhibitors (sirolimus), ketoprofen / anti-inflammatory approaches, repurposed drugs, gene therapy
2. **Dietary & lifestyle** — weight management, nutrition, exercise, conservative therapy
3. **Genetics & biomarkers** — FLT4/VEGFR3, FOXC2, SOX18, GATA2 and other primary-lymphedema variants; diagnostic biomarkers
4. **Clinical-trials pipeline** — active/recruiting trials and their status
5. **Disease management** — imaging (ICG lymphography), compression, surgical options (LVA, VLNT), staging
6. **Patient community** — patient-relevant developments and resources

See [`prompts/evaluation-framework.md`](prompts/evaluation-framework.md) for the full evidence-scoring specification.

---

## Repository layout

```
├── prompts/           # Claude Code system prompts (scan, deep) + evaluation framework
├── scripts/           # Fetchers (PubMed/Semantic Scholar), processing, schema guard, launchd plists, Drive sync
├── data/              # Structured JSON store: papers, trials, findings, run-log, user-state
├── output/
│   ├── knowledge-base/  # Six living synthesis documents
│   ├── digests/         # Weekly digests (YYYY-WXX)
│   ├── alerts/          # Dated significant-finding alerts
│   └── logs/            # Per-run logs (gitignored)
└── dashboard/         # Next.js web dashboard
```

### Data store (`data/`)

| File | Contents |
|------|----------|
| `papers.json` | Tracked papers with metadata, multi-axis scores, and summaries |
| `trials.json` | ClinicalTrials.gov records being followed |
| `findings.json` | Scored, synthesized findings |
| `run-log.json` | History of every scan/deep run |
| `user-state.json` | Dashboard read/seen state |

---

## Dashboard

A [Next.js](https://nextjs.org) app (Tailwind CSS, shadcn/ui, Recharts) that reads directly from `data/` and `output/`. Pages: overview, papers, trials, digests, and knowledge base.

```bash
cd dashboard
npm install
npm run build
npm run start -- --hostname 0.0.0.0 --port 3001   # serves on http://localhost:3001
```

> **Always run the dashboard in production mode** (`build` + `start`). Dev mode (Turbopack) has been observed to spawn runaway node processes.

---

## Setup

### Prerequisites

- macOS (uses `launchd` for scheduling)
- [Claude Code](https://claude.com/claude-code)
- Python 3 and Node.js
- (Optional) A [Semantic Scholar API key](https://www.semanticscholar.org/product/api) for higher rate limits

### 1. API keys

Create `data/.api-keys.json` (gitignored — never commit real keys):

```json
{
  "semantic_scholar": "YOUR_KEY_HERE",
  "ncbi": ""
}
```

### 2. Paths

The prompts, scripts, and `launchd` plists in this repo use absolute paths for one machine. Update them to your own project location before running.

### 3. Baseline

Run the baseline fetch/process to build the initial corpus:

```bash
python3 scripts/baseline_fetch.py
python3 scripts/baseline_process.py
```

### 4. Schedule the agents

Copy the plists to `~/Library/LaunchAgents/` and load them:

```bash
cp scripts/com.lymphedema.scan.plist ~/Library/LaunchAgents/
cp scripts/com.lymphedema.deep.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.lymphedema.scan.plist
launchctl load ~/Library/LaunchAgents/com.lymphedema.deep.plist
```

---

## Customize it for another disease

This repo is one instance of a reusable research-intelligence template — the same system also runs for ADPKD at [adpkd-research-intelligence](https://github.com/gzaal/adpkd-research-intelligence). To point it at a different disease or research domain:

1. **Scope & disease name** — update the disease name and the in/out-of-scope definition throughout `prompts/scan.md`, `prompts/deep.md`, and `.claude/CLAUDE.md`. The scope gate lives in `scripts/baseline_process.py` (`classify_*` functions) — adjust its keywords.
2. **Research taxonomy** — redefine the six dimensions and their keyword maps (`DIMENSION_KEYWORDS` / `SUBTOPIC_KEYWORDS` in `scripts/baseline_process.py`) and rename the six documents in `output/knowledge-base/`.
3. **Search queries** — change the PubMed / Semantic Scholar / ClinicalTrials.gov query terms in `scripts/baseline_fetch.py` and `scripts/pubmed_fetch.py`.
4. **Evaluation framework** — the 5-axis scoring in `prompts/evaluation-framework.md` is disease-agnostic, but the skepticism flags (e.g. lymphedema's "obesity not controlled") are domain-specific — edit them to match your field's common biases.
5. **Paths & schedule** — update absolute paths and the `launchd` plist labels/times (`com.<disease>.scan`, `com.<disease>.deep`) in `scripts/`.
6. **Dashboard** — change the title, port, and palette in `dashboard/` (this instance uses port 3001 and a green theme; ADPKD uses 3000 and blue).

The pipeline, dashboard, scoring model, and schema guard are all reusable as-is — only the domain-specific configuration above needs to change.

---

## License

The original source code and repository-authored documentation in this project
are licensed under the [MIT License](LICENSE).

Research publications, abstracts, article metadata, clinical-trial records,
datasets, APIs, and linked services are third-party material. They are not
relicensed by this repository and remain subject to their respective
copyright, database rights, terms of service, and licenses. Check the source
terms before copying or redistributing them.

This is a personal research tool shared for reference. Its automatically
generated summaries and findings may contain errors, are not medical advice,
and are provided without any warranty. Verify claims against primary sources
and consult a qualified clinician before making health decisions.
