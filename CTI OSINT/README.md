# CyberIntel Daily Digest (CTI OSINT)

An [n8n](https://n8n.io) workflow that turns a day's worth of public cyber security reporting into one consolidated threat intelligence brief.

Every morning it pulls 42 security news and vendor research feeds, drops articles it has already covered, groups articles about the same story, and has an LLM merge each group into a structured brief. It then validates and enriches the results (CVEs, IOCs, MITRE ATT&CK) and publishes:

- a **Markdown digest** for people to read
- a **STIX 2.1 bundle** for threat intel platforms
- a **full JSON run record**
- optionally, a **MISP event per story** and a **Discord post**

The workflow runs on free tiers and local models: free OpenRouter and Groq models do the synthesis, and LM Studio covers embeddings, reranking and fallback synthesis.

| File | Description |
|---|---|
| `cyberintel-daily-digest.public.json` | The n8n workflow export. Import it into n8n. |

---

## How it works

```
Schedule (07:00) / Manual
        │
     Config ──► rank free OpenRouter + Groq models ──► ensure Qdrant collection ──► load ATT&CK map
        │
   42 RSS feeds ──► Normalize (36h window, URL canonicalisation, dedupe)
        │
   Qdrant lookup ──► drop already-digested articles
        │
   Fetch article ──► Extract text + regex CVEs / IOCs / ATT&CK IDs
        │
   Embed (LM Studio) ──► search prior stories in Qdrant ──► Cluster (+ reranker)
        │
   Build prompt ──► Synthesize (OpenRouter → Groq → LM Studio) ──► Validate
        │
   IOCs ──► VirusTotal + AlienVault OTX ──► verdicts
        │
   CVEs ──► CISA KEV + FIRST EPSS + NVD
        │
   MISP sync (one event per story) ──► Build outputs ──► write files ──► Qdrant upsert
                                                      └─► Discord
```

### 1. Collect and deduplicate
- **Feeds:** 42 sources listed in the `Config` node, including BleepingComputer, The Hacker News, CISA, Unit 42, Cisco Talos, Microsoft, CrowdStrike, SANS ISC and CERT/CC.
- **Normalize:** keeps items published in the last `lookback_hours` (36h), strips tracking parameters, and removes duplicates by canonical URL and by title.
- **Filter New:** every article gets a deterministic UUID. Anything already stored in Qdrant was digested on an earlier run and is skipped.
- **Fetch Article / Extract Text:** downloads the full page and extracts the article body. If the site blocks the request or the page is thin, it falls back to the RSS excerpt. It also regex-extracts CVE IDs, ATT&CK technique IDs and defanged or plain indicators: hashes, URLs, IPs and domains.

### 2. Cluster into stories
- Articles are embedded with **Qwen3-Embedding-4B** (2560-d) through LM Studio.
- Two articles join the same story when:
  - cosine similarity is ≥ `similarity_threshold_hard` (0.92), **or**
  - cosine similarity is ≥ `similarity_threshold` (0.78) **and** the articles share a CVE or at least 2 distinctive title tokens.
- **Borderline pairs** (0.70 – 0.92 with no lexical overlap) go to a cross-encoder judge, **Qwen3-Reranker-4B**.
- **Prior-story linking:** each story is searched against earlier digests (≥ 0.85). A confirmed match marks the story as an *update*, and the LLM is asked to fill in `whats_new`.
- If embeddings are unavailable, clustering falls back to lexical evidence only, and the run still ships.

### 3. Synthesize
One LLM call per story, returning JSON that matches a strict schema: headline, category, severity, summary, key facts, what's new, affected products, CVEs, IOCs, TTPs, threat actors, malware, recommended actions and source discrepancies.

- **Provider order:** `openrouter → groq → lmstudio`. The first provider that answers wins.
- **Live model ranking:** at the start of each run, the workflow fetches the full OpenRouter `:free` catalogue and the Groq model list and ranks them. Preferred models come first, then models with structured output support, then context size.
- **Circuit breakers:**
  - A model that fails sits out a cooldown that doubles on each strike. A 404 or 400 benches it for the rest of the run.
  - Account-level limits (401, 402, daily quota) bench the whole provider for the run.
- **Time budget:** `synth_budget_ms` (45 min) is shared across stories. Once it is spent, the remaining stories go straight to LM Studio.
- The Synthesize node never throws. A story that no provider answers is rendered from raw excerpts.

### 4. Validate
Model output is treated as untrusted:
- Key aliases are mapped back to the schema, and fields in the wrong shape are coerced.
- **IOCs:** refanged, checked per type (valid TLD, public IPs only, hash lengths), and filtered against a denylist of news, vendor and platform domains and placeholder values. Indicators found by regex are always kept.
- **ATT&CK:** techniques are resolved **by name** against the local technique map. A model-supplied ID is used only when its name agrees. Proposals that can't be resolved are dropped and counted in the run notes.
- **Actors and malware:** generic filler such as "Unknown", "attackers" or country names is removed.
- **Severity guardrails:**
  - A `critical` rating needs evidence.
  - Research, policy and tooling stories are capped at `medium`.
  - Vulnerability stories with no sign of exploitation are capped at `medium`.

### 5. Enrich
- **IOCs:** unique indicators, ordered by story severity, are looked up in **VirusTotal** and **AlienVault OTX**. The results produce a verdict (`malicious`, `suspicious`, `clean` or `unverified`) and a STIX confidence score. Lookups are throttled to fit free-tier limits (VirusTotal: 4 requests/min).
- **CVEs:** **CISA KEV** status, due date and ransomware use; **FIRST EPSS** score and percentile; **NVD** CVSS score, vector, CWE and description. NVD lookups are throttled to the public 5 requests / 30 s limit.

### 6. MISP (optional)
Each story gets a MISP event that serves as its running record. When a later story continues an earlier one, new attributes are added to the **same event**, reached through the prior-story link stored in Qdrant. Event UUIDs are deterministic, so re-running the same day is idempotent.

Events carry:
- CVEs
- source links
- ATT&CK galaxy tags
- IOCs tagged with their verdict

`to_ids` is set only for `malicious` and `suspicious` indicators. The workflow never deletes anything in MISP. After syncing, it reads each event back so the digest can show the **cumulative** indicator set for developing stories.

### 7. Outputs
Files are written to `output_dir` (default `/data/intel/output`):

| File | Contents |
|---|---|
| `YYYY-MM-DD-digest.md` | Human-readable digest (details below). |
| `YYYY-MM-DD-stix.json` | STIX 2.1 bundle (details below). |
| `YYYY-MM-DD-data.json` | Full run data: stats, config, stories, enrichment, IOC intel and notes. |
| `latest-digest.md` | Copy of the most recent digest. |

The **digest** contains:
- Priorities
- Stories (critical, high and medium)
- "Also noted" (low and info)
- Appendix A: CVEs
- Appendix B: IOCs, defanged, with VirusTotal and OTX results
- Appendix C: ATT&CK techniques
- Run notes

The **STIX bundle** contains report, vulnerability, indicator, attack-pattern, intrusion-set, malware and relationship objects. All objects are marked TLP:CLEAR and use deterministic IDs, so a threat intel platform that ingests daily bundles updates existing objects instead of creating duplicates.

**Discord** gets two messages when `DISCORD_WEBHOOK_URL` is set: a summary embed with the digest attached, then the STIX bundle.

Qdrant is updated only **after** the files are written. If a run fails partway, its articles are picked up again on the next run instead of being silently marked as seen.

---

## Requirements

| Component | Purpose | Required? |
|---|---|---|
| **n8n** (self-hosted) | Runs the workflow. Needs Code node v2, HTTP Request v4.2 and If v2. | Yes |
| **Qdrant** | Stores article vectors for dedup and prior-story search. | Yes |
| **LM Studio** (OpenAI-compatible API) | Embeddings, reranker and fallback chat model. | Yes, for embeddings. The workflow degrades without it. |
| **OpenRouter** API key | Primary synthesis (free models). | Recommended |
| **Groq** API key | Secondary synthesis (free tier). | Recommended |
| **VirusTotal** API key | IOC enrichment. | Optional |
| **AlienVault OTX** API key | IOC enrichment. | Optional |
| **MISP** instance + API key | Per-story event record. | Optional |
| **Discord** webhook | Posts the digest. | Optional |
| **ATT&CK technique map** | Local JSON used to resolve TTPs. | Yes |

LM Studio models used by default (change them in `Config`):

| Setting | Model |
|---|---|
| `embed_model` | `text-embedding-qwen3-embedding-4b` (2560-d) |
| `reranker_model` | `qwen3-reranker-4b` |
| `chat_model` | `google/gemma-4-e2b` (fallback only) |

---

## Setup

### 1. Environment variables
Set these on the n8n container or process:

```bash
LMSTUDIO_URL=http://<lmstudio-host>:1234/v1   # OpenAI-compatible base URL
QDRANT_URL=http://qdrant:6333
OPENROUTER_API_KEY=sk-or-...
GROQ_API_KEY=gsk_...
MISP_URL=https://misp.example.com             # optional
MISP_API_KEY=...                              # optional; empty = MISP sync skipped
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...   # optional; empty = no post

# n8n settings the workflow depends on
N8N_BLOCK_ENV_ACCESS_IN_NODE=false   # Code nodes read $env
NODE_FUNCTION_ALLOW_BUILTIN=crypto   # deterministic UUIDs (a pure-JS fallback exists)
EXECUTIONS_TIMEOUT_MAX=10800         # the workflow sets a 3h execution timeout
GENERIC_TIMEZONE=America/Los_Angeles
```

### 2. n8n credentials
Create these credentials in n8n and attach them to the matching nodes:
- **VirusTotal API**, on the `VirusTotal Lookup` node
- **AlienVault API**, on the `OTX Lookup` node

### 3. ATT&CK technique map
The `Validate` node reads `attack_map_path` (default `/data/intel/data/attack-techniques.json`). It expects this shape:

```json
{
  "techniques": {
    "T1190": {
      "name": "Exploit Public-Facing Application",
      "tactics": ["initial-access"],
      "url": "https://attack.mitre.org/techniques/T1190/",
      "stix_id": "attack-pattern--3f886f2a-874f-4333-b794-aa6075009b1c",
      "deprecated": false
    }
  }
}
```

The file is not included in this folder. You can generate it from MITRE's Enterprise ATT&CK STIX data:

```python
import json, urllib.request
src = "https://raw.githubusercontent.com/mitre-attack/attack-stix-data/master/enterprise-attack/enterprise-attack.json"
bundle = json.load(urllib.request.urlopen(src))
out = {}
for o in bundle["objects"]:
    if o.get("type") != "attack-pattern" or o.get("revoked"):
        continue
    ref = next((r for r in o.get("external_references", []) if r.get("source_name") == "mitre-attack"), None)
    if not ref:
        continue
    out[ref["external_id"]] = {
        "name": o["name"],
        "tactics": [p["phase_name"] for p in o.get("kill_chain_phases", []) if p.get("kill_chain_name") == "mitre-attack"],
        "url": ref.get("url"),
        "stix_id": o["id"],
        "deprecated": bool(o.get("x_mitre_deprecated")),
    }
json.dump({"techniques": out}, open("attack-techniques.json", "w"), indent=1)
```

### 4. File paths
The n8n process must be able to read the ATT&CK map and write to `output_dir`. With Docker, mount a volume at `/data/intel`:

```
/data/intel/data/attack-techniques.json
/data/intel/output/
```

If the n8n version you run restricts file access (`N8N_RESTRICT_FILE_ACCESS_TO`), add `/data/intel` to the allowed paths.

### 5. Import and run
1. In n8n, go to **Workflows → Import from File** and select `cyberintel-daily-digest.public.json`.
2. Attach the credentials from step 2.
3. Review the `Config` node: timezone, feeds, models, thresholds and paths.
4. Click **Manual Run** to test.
5. Activate the workflow. The `Daily 07:00` trigger then runs it every day at 07:00 in the workflow's timezone (America/Los_Angeles).

The Qdrant collection (`cyberintel_articles_v2`) is created automatically on the first run.

---

## Configuration

Most behaviour is set in the **`Config`** Code node. Every other node reads its values.

| Key | Default | Notes |
|---|---|---|
| `lookback_hours` | `36` | RSS item age window. |
| `feeds` | 42 sources | `{ source, url }` list. |
| `provider_order` | `['openrouter','groq','lmstudio']` | Synthesis failover order. |
| `openrouter_preferred` / `groq_preferred` | curated lists | Tried first when present in the live catalogue. |
| `synth_budget_ms` | 45 min | Total synthesis time budget for the run. |
| `embed_model` / `embed_dim` / `collection` | Qwen3-Embedding-4B / 2560 / `cyberintel_articles_v2` | **Change the collection name whenever you change the embedding dimension.** |
| `similarity_threshold` / `_hard` | `0.78` / `0.92` | Story clustering thresholds. |
| `prior_similarity_threshold` | `0.85` | Link a story to earlier digests. |
| `reranker_enabled` | `true` | Set to `false` to cluster on thresholds only. |
| `max_cluster_chars` | `36000` | Article text sent to the LLM per story. |
| `max_cves_to_enrich` | `60` | Capped because of the NVD rate limit. |
| `max_iocs_to_enrich` | `200` | Capped because of the VirusTotal free tier (500/day). |
| `misp_distribution` / `misp_tags` | `0` / `tlp:clear`, … | MISP event settings. |
| `misp_to_ids_verdicts` | `['malicious','suspicious']` | Which verdicts get `to_ids=true`. |
| `output_dir` / `attack_map_path` | `/data/intel/...` | File locations. |

---

## Runtime and limits

- Most of the run time comes from rate limits. VirusTotal lookups run at about 15.5 s each, NVD at about 6.5 s each, and synthesis is capped at 45 minutes. A busy day can take over an hour, so the workflow's execution timeout is set to 3 hours.
- OpenRouter free models allow about 50 requests per day without credits. Groq applies per-model tokens-per-day limits. Stories that neither can handle go to LM Studio.
- Stories are tagged **TLP:CLEAR** because every input is public reporting. Review `misp_distribution` before syncing to a shared MISP instance.

---

## Caveats

- The output is machine-generated from news reporting. Treat briefs as leads to verify, not as ground truth. Validation reduces hallucinated CVEs, IOCs and TTPs but cannot remove them entirely.
- An IOC verdict reflects VirusTotal and OTX results at lookup time. `unverified` means the lookup found nothing, not that the indicator is benign.
- Some sites block automated fetches. Those articles are summarized from their RSS excerpt, and the digest marks them as such.
