# Compliance Clause Comparator — Architecture & Data Flow

**Author:** Shivshankar Ganapuram
**Created:** 2025-11-08

---

## 1. Objective
Build a RegTech tool that compares two versions of a regulatory/legal document and highlights changes in obligations, permissions, and risks — including subtle semantic shifts (e.g., *may → must*), added/removed obligations, and newly introduced parties or responsibilities.

---

## 2. High-level architecture (components)

1. **Ingestion Layer**
   - Document upload (PDF, DOCX, TXT)
   - Version metadata (version id, source, date, jurisdiction)
   - Preprocessing pipeline (OCR for scanned PDFs)

2. **Preprocessing & Parsing**
   - Document normalization: remove headers/footers, normalize whitespace
   - Structural parsing: extract sections, headings, clause numbers using heuristics + regex + ML
   - Sentence/Clause chunking: split into clause-level units

3. **Indexing & Storage**
   - Clause store (Postgres / Elasticsearch): clause_id, text, metadata, doc_version
   - Vector store (for embeddings): PostgreSQL+pgvector / FAISS / Pinecone

4. **Alignment & Matching**
   - Compute clause embeddings
   - Nearest-neighbour matching (cosine similarity)
   - Candidate pairs generation with similarity score and fallback heuristics

5. **Semantic Comparator (LLM + Rules)**
   - Use an LLM for semantic diffing and explanation (compare two clauses and report differences in obligations/risks)
   - Rule-based layers to flag explicit keywords: `must`, `shall`, `may`, `should`, `required`, `prohibited`.
   - Hybrid output: structured JSON + human-readable summary

6. **Risk Scoring & Classification**
   - Rule-based scoring for certain keywords & numeric changes (e.g., retention period increase)
   - Supervised ML model for contextual risk prediction (fine-tuned transformer classifier)

7. **UI / Visualization**
   - Web UI (React): side-by-side clause view, color-coded highlights, change summary dashboard
   - Downloadable reports (PDF/CSV)

8. **Audit & Explainability**
   - Store LLM prompts & responses for auditability
   - Confidence score, provenance (which model / rule triggered), timestamp

9. **Security & Compliance**
   - Data encryption at rest/in transit, RBAC, tenant isolation (multi-tenant)

---

## 3. Data flow (step-by-step)

1. **Upload**: User uploads `doc_v1` and `doc_v2` (or selects two versions from a repository).
2. **Preprocess**: Run OCR (if needed), normalize, extract headings and clause-level chunks.
3. **Indexing**: Store clauses and compute embeddings. Insert embeddings into vector store.
4. **Alignment**: For each clause in `doc_v2`, find top-k similar clauses in `doc_v1` using cosine similarity. Create match candidates with similarity score.
5. **Comparator**: For each candidate pair run:
   - Rule-based quick checks (exact keywords, numeric comparisons)
   - LLM prompt to generate semantic diff and classify change (added/removed/modified/relocated/merged/split)
6. **Risk Scoring**: Aggregate signals (keyword severity, LLM classification, historical risk models) to compute `risk_level`.
7. **Post-processing**: Deduplicate, order by impact, generate UI-ready structure and exportable report.
8. **Feedback loop**: Users can accept/correct classifications to create labeled data for model retraining.

---

## 4. Clause alignment algorithm (detailed)

1. **Embedding model choices**
   - Off-the-shelf: OpenAI `text-embedding-3-large`, Hugging Face Sentence-BERT variants (all-mpnet-base-v2), or custom legal-domain embeddings.
2. **Similarity search**
   - Use FAISS / pgvector / Pinecone to find top-3 candidate matches per clause.
   - Use both *semantic* (embedding) and *token overlap* (Jaccard, edit distance) scores; compute a weighted score.
3. **Alignment heuristics**
   - If IDs or headings match, boost score.
   - If one clause maps to multiple clauses (split/merge), mark special `merge/split` candidate.

---

## 5. LLM-driven semantic comparator — prompt design

**Prompt template (concise):**
```
You are a legal-translation and comparison assistant.
Input:
 - OLD_CLAUSE: "<old text>"
 - NEW_CLAUSE: "<new text>"
Task: Compare OLD_CLAUSE and NEW_CLAUSE and return a JSON with following fields:
 - change_type: one of [added, removed, modified, relocated, merged, split]
 - obligation_changes: list of {entity, old_obligation, new_obligation, severity}
 - permissions_changes: list similar
 - numeric_changes: list of {field, old_value, new_value, significance}
 - risk_level: [low, medium, high]
 - human_summary: short text (1-2 sentences)
 - confidence: float 0-1
Important: explain if the change alters legal duties (e.g., may -> must = increased obligation).
```

**Usage notes:**
- When calling, prepend clause metadata: clause ID, section heading, doc version.
- Include a short instruction to prioritize `must/shall` verbs and time/value changes.

---

## 6. Risk scoring model (prototype)

**Hybrid approach:**
1. **Rule-based tier** — immediate flags for critical keywords: `must`, `shall`, `required`, `prohibited`, `penalty`, `fine`.
2. **Numeric heuristics** — large increases in retention period, monetary limits, or threshold changes.
3. **ML classifier (optional)** — train a transformer-based classifier on labeled clause-pairs with labels (risk: low/med/high).
   - Input features: embedding of concatenated pair, keyword counts, delta of numeric values, clause length change, similarity score.
   - Model: fine-tuned BERT / RoBERTa for classification.

**Output:** `risk_level` + confidence.

---

## 7. Input schema / API

**POST /compare**
Request JSON:
```json
{
  "doc_old": {"id": "v1", "source": "uploaded", "clauses": [{"id":"1.1","text":"..."}, ...]},
  "doc_new": {"id": "v2","clauses": [{"id":"1.1","text":"..."}, ...]},
  "options": {"similarity_threshold":0.78, "top_k":3, "risk_model":"hybrid"}
}
```

Response JSON:
```json
{
  "matches": [
    {
      "old_clause_id":"1.1",
      "new_clause_id":"1.1",
      "old_text":"...",
      "new_text":"...",
      "similarity":0.92,
      "change":{...},
      "risk_level":"medium"
    }
  ],
  "summary": {"total_added":1, "total_removed":0, "total_modified":3}
}
```

---

## 8. Mock data (usable)

### Document V1 (JSON):
```json
{
  "version":"policy_v1",
  "clauses":[
    {"id":"1.1","text":"The company may collect user data for analytics."},
    {"id":"1.2","text":"The company must delete user data after 6 months."},
    {"id":"2.1","text":"Employees should follow security guidelines."}
  ]
}
```

### Document V2 (JSON):
```json
{
  "version":"policy_v2",
  "clauses":[
    {"id":"1.1","text":"The company must collect user data for analytics and compliance."},
    {"id":"1.2","text":"The company must delete user data after 12 months."},
    {"id":"2.1","text":"Employees shall strictly follow updated security guidelines."},
    {"id":"3.1","text":"Vendors must sign a data protection agreement before accessing any personal data."}
  ]
}
```

### Expected comparator output (example):
```json
{
  "matches":[
    {"old_clause_id":"1.1","new_clause_id":"1.1","change_type":"modified","summary":"may -> must; added compliance purpose","risk_level":"medium"},
    {"old_clause_id":"1.2","new_clause_id":"1.2","change_type":"modified","summary":"retention increased 6 -> 12 months","risk_level":"high"},
    {"old_clause_id":"2.1","new_clause_id":"2.1","change_type":"modified","summary":"should -> shall strictly; stronger enforcement","risk_level":"medium"},
    {"old_clause_id":null,"new_clause_id":"3.1","change_type":"added","summary":"New vendor obligation","risk_level":"high"}
  ]
}
```

---

## 9. UI/UX wireframe (brief)

- **Left pane**: Document tree (version, sections, clause IDs)
- **Center pane**: Side-by-side clause viewer with highlights (+ inline diff)
- **Right pane**: Change summary and risk dashboard (filters: severity, type, section)
- **Top**: Compare controls, thresholds, export button
- **Actions**: Accept/Reject/Annotate change (for feedback labeling)

---

## 10. Training / Labeling strategy

1. **Bootstrapping**: Use rule-based heuristics and synthetic edits (swap verbs, change numbers) to produce labeled pairs.
2. **Human-in-the-loop**: Legal reviewers verify and correct LLM outputs; collected corrections feed supervised fine-tuning.
3. **Active learning**: Prioritize low-confidence or high-impact pairs for reviewer labeling.

---

## 11. Deployment & infra

- **Core services**: Containerized microservices (FastAPI / Node.js)
- **Models**: Hosted LLM (OpenAI / Anthropic / self-hosted Llama 2 or larger), embeddings in vector DB
- **Storage**: PostgreSQL + pgvector or Elasticsearch + FAISS
- **Orchestration**: Kubernetes (EKS / GKE / AKS)
- **Monitoring**: Prometheus + Grafana

---

## 12. Security & privacy considerations

- Data residency and encryption; consider private LLM or bring-your-own-key for external LLMs.
- Redaction option for PII before sending to external LLMs.
- Detailed audit logs for every comparison and reviewer action.

---

## 13. Prototype roadmap (3 sprints)

**Sprint 0 (1 week)**: PoC ingestion, clause chunking, embeddings, simple similarity matching, small UI to upload and view matched pairs.

**Sprint 1 (2–3 weeks)**: Integrate LLM comparator, produce structured JSON diffs, add risk heuristics, basic UI highlights.

**Sprint 2 (2–3 weeks)**: Feedback loop, model retraining pipeline, export reports, RBAC and encryption, performance tuning.

---

## 14. Risks & mitigations

- **False positives/negatives**: Use hybrid rules + LLM + human review to reduce errors.
- **Data leakage to LLM providers**: Use private endpoints or on-premises LLMs for sensitive docs.
- **Complex clause mapping (split/merge)**: Provide manual linking UI for ambiguous cases and record them as training data.

---

## 15. Next actions (pick one)
- Build the Sprint 0 PoC (I can provide starter code + API routes).
- Design UI screens (React components + sample CSS/Tailwind).
- Prepare a small dataset and annotation guideline for labeling.


---

## 16. Guardrails Integration (added)

Guardrails provide safety, consistency, and auditability across the entire comparator pipeline. This section documents practical guardrails to implement, where they are applied, and examples of enforcement patterns.

### 16.1 Guardrail Categories

- **Input Validation Guardrails** — validate uploaded files, detect OCR issues, remove hidden content, and enforce allowed formats.
- **Normalization Guardrails** — ensure clause text normalization (canonical date formats, number normalization, contract-entity mapping).
- **Alignment Logic Guardrails** — enforce thresholds and heuristics for clause matching; detect ambiguous mappings and mark for human review.
- **Prompt / Output Schema Guardrails** — require LLM outputs to conform to a strict JSON schema (change_type, risk_level, obligation_changes, confidence, human_summary). Any non-conforming response triggers a retry or fallback.
- **Content Policy Guardrails** — block disallowed assertions (legal advice, unverifiable claims, PII leakage) and redact sensitive data before external model calls.
- **Confidence & Human-review Guardrails** — if the model confidence is below threshold or the risk level is high, route the result to a human reviewer and flag it in the UI.
- **Audit & Provenance Guardrails** — log model version, prompt hash, timestamp, input clause IDs, and any human overrides for traceability.
- **Display/UX Guardrails** — visually mark results as `verified`, `low-confidence`, or `human-review required` and prevent export of unverified decisions.

### 16.2 Technical Patterns & Examples

**Schema enforcement (example):**
- Use Pydantic or Guardrails AI to define the expected JSON output and validate responses automatically.

**Retry & recursive prompting:**
- On schema failure or obviously inconsistent outputs, re-prompt the LLM with additional context and stricter instructions. Limit retries to a configurable count (e.g., 2 retries).

**Redaction before LLM calls:**
- Run a PII detection pass and replace detected values with placeholders (`<REDACTED_EMAIL>`). Keep mapping locally so the UI can rehydrate if needed under secure conditions.

**Hybrid fallback:**
- If LLM output is missing critical fields or confidence is low, fall back to deterministic rules (keyword-diff, numeric delta detection) and mark the result as `partial`.

**Human-in-the-loop gating:**
- For `risk_level == high` or `change_type` in critical categories (e.g., `obligation_reversal`), the system requires explicit legal reviewer approval before finalizing the report.

### 16.3 Where to place Guardrails in the pipeline

- **Ingestion**: Input Validation Guardrails
- **Preprocessing**: Normalization Guardrails
- **Alignment**: Alignment Logic Guardrails
- **Comparator (LLM layer)**: Prompt/Output Schema, Content Policy, Confidence Guardrails
- **Post-processing**: Audit & Provenance, Human-review Gate
- **UI**: Display/UX Guardrails

### 16.4 Operational & Governance Notes

- **Model management**: Maintain a registry of approved model versions and quarantine any unapproved model outputs until reviewed.
- **Settings & tunables**: Admin UI for thresholds (similarity, confidence, retry counts). Changes to these settings are auditable.
- **Testing**: Add synthetic test-cases that represent common failure modes (e.g., `may -> must`, split/merge clause mapping, numeric drift) and run them in CI.
- **Privacy**: For highly sensitive documents, enable an on-premises-only processing mode which disables external LLM calls and uses local models only.

---

*End of document*

