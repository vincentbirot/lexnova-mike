# Brief 11 — Firm-Wide RAG Memory

## Context

Today, Lexnova AI's chat assistant has a hard wall around every matter: it can only see the documents and chats inside the current matter. A new associate cannot ask "how does Lex Nova usually structure share transfers" and get a useful answer drawn from the firm's 200+ closed matters. The institutional knowledge built up over years of practice is invisible to the AI. This is the single biggest gap separating Lexnova AI from Harvey or Legora at the same price point.

This brief introduces a firm-wide retrieval-augmented generation (RAG) layer. Every document uploaded to any matter becomes part of a searchable embedding index. When a lawyer asks the assistant a question, the system retrieves the most relevant chunks from across the firm's history (subject to matter access rules and conflict walls) and injects them as context. The assistant can now cite "in the Acme deal we structured this as ___, see Acme matter dated ___" with confidence.

This is the most strategically important feature in the build plan. It is also the most technically and ethically complex due to confidentiality and conflict-of-interest concerns.

## Audience

Every Lex Nova lawyer. Particularly valuable for: new hires (instant access to firm precedent), partners (citing past work), and lawyers crossing practice areas (e.g., a corporate lawyer needing real estate context).

## Primary user stories

A new associate joins the firm. On day one, she asks the assistant "Show me how Lex Nova typically structures FBA-compliant property holding companies in Bangkok." The assistant retrieves chunks from 8 past matters where this structure was used, summarizes the pattern, and cites each source matter and document.

A partner is starting a new BOI application matter. He asks "What was the BOI position we took on import-substitution arguments in the past?" The assistant retrieves three past BOI matters, summarizes the firm's evolving stance, and notes which arguments were accepted by the BOI Board.

A junior associate is drafting a Share Purchase Agreement and asks "What governing-law and dispute-resolution clauses do we typically use for cross-border deals where the buyer is European and the target is Thai?" The assistant retrieves 12 past SPAs, identifies the 3 most common pairings, and shows verbatim language from anonymized past matters.

A conflict-of-interest scenario: Lex Nova represented Buyer Acme in a 2024 deal and is now considering representing Seller Beta in a 2026 deal involving Acme. When the lawyer on the new matter asks the assistant about Acme, the system surfaces a "Conflict warning" rather than the past Acme content, escalating to the conflict committee for clearance.

## Architecture

The retrieval pipeline:

1. On document upload (any matter, any file), an embedding job runs. Document is chunked (512-1024 tokens with 100-token overlap), each chunk is embedded via Voyage Legal-2 or text-embedding-3-large. Embeddings stored in pgvector inside Supabase.

2. Each embedding row stores: chunk text, chunk position in document, document_id, matter_id, matter_status, practice_area (derived), created_at, conflict_walls (jsonb of restricted user/matter ids).

3. On chat query, the user's question is also embedded. Top-k similarity search returns candidate chunks across all matters. Filter the candidates through:

   - User's accessible matter set (assignee or shared_with)
   - Conflict walls (matters explicitly walled off from this user)
   - Practice area weighting (boost chunks from same practice area as current matter)

4. Top 10 retained chunks are passed to the assistant as system context with citation metadata.

5. The assistant's response cites each chunk with matter name, document name, and date.

Embedding cost: roughly USD 0.05 per million tokens for text-embedding-3-large or Voyage. A 30-page document is ~15K tokens, so ~USD 0.001 per document. Trivial at firm scale.

## Schema delta

Use Supabase's pgvector extension (`create extension if not exists vector;`).

New table `document_embeddings`:

- `id` uuid primary key
- `document_id` uuid foreign key to documents(id) on delete cascade
- `chunk_index` integer
- `chunk_text` text
- `embedding` vector(3072) — for text-embedding-3-large
- `matter_id` uuid (denormalized for query performance)
- `practice_area` text nullable
- `embedded_at` timestamptz

Index: `using ivfflat (embedding vector_cosine_ops)` for similarity search.

New table `conflict_walls`:

- `id` uuid primary key
- `matter_id` uuid — the protected matter
- `walled_user_id` uuid nullable — user blocked from accessing
- `walled_matter_id` uuid nullable — matter that conflicts with this matter
- `reason` text
- `created_by_user_id` uuid
- `created_at` timestamptz

New columns on projects: `practice_area` text (e.g., "corporate", "real_estate", "tax", "labor", "litigation", "regulatory").

## Screens

### Cross-matter search (`/search`)

A new top-level route `/search` accessible from sidebar. A search bar at top, results below.

User types a natural-language query. Results show:

- A natural-language synthesis from the assistant at top: "Across X matters, here is the firm's approach to Y..."
- Below: a list of source chunks, each in a card with matter name, document name, date, snippet (with the searched phrase highlighted), and "Open matter" / "Open document" actions
- Filter sidebar: practice area, date range, matter status, exclude walled matters

This is the primary surface for active firm-knowledge search.

### Inline RAG context in matter chat

When asking a question in any matter's assistant chat, the response automatically includes citations from cross-matter retrieval where relevant. Each cross-matter citation appears as a small footnote chip:

- `[1] Acme SPA (2024) — page 3` — clickable, opens the source document in a side panel

The footnotes are visually distinct from in-matter citations: cross-matter ones have a small "firm precedent" icon (stylized library / books icon).

### Conflict warning

If a query hits walled content, the assistant surfaces a warning instead of the content:

> ⚠️ Conflict warning: Some firm precedent on this topic comes from matters that are walled from your access. The conflict committee can review and clear access if appropriate. [Request clearance]

Clicking "Request clearance" opens a form to the firm's conflict admin (defined in admin panel).

### Matter settings — practice area

In the matter detail page settings, add a "Practice area" dropdown. Defaults to inferred from matter content (the assistant suggests). User can override. Used for retrieval weighting.

### Conflict walls (admin panel addition)

In the admin panel (Brief 08), under "Templates" or as a new section "Conflicts":

- A list of active conflict walls
- "+ New conflict wall" — pick a matter, pick walled users (or matters), provide reason, save
- Walls take effect immediately
- Audit log entry recorded

### Embedding processing status

When a document is uploaded, after the existing PDF processing step, a new "Indexing for firm search" step. Shows in the document card with a small spinner. Takes 5-30s for typical legal docs. Once complete, a small ✓ "Indexed" badge appears.

If embedding fails (OpenAI/Voyage API down), retry on a schedule. Surface failure as a warning in admin panel for manual reindex.

### Admin: Re-embed everything

In admin panel, an action "Re-embed all documents". Useful when changing embedding models or chunking strategy. Background job, progress visible. Only admin can trigger.

## Component additions

- `<CrossMatterSearch>` — the /search page
- `<RagCitationFootnote>` — the small footnote chip in chat responses
- `<ConflictWarning>` — modal/banner for walled content
- `<PracticeAreaSelect>` — dropdown for matter classification
- `<ConflictWallsManager>` — admin section for managing walls
- `<DocumentIndexingStatus>` — small badge showing embedding state

## Edge cases

**Embedding model deprecation**: when OpenAI / Voyage updates models, store the model id with each embedding. Re-embed in batches when migrating.

**Large documents (>50MB or >100K tokens)**: chunk in batches, may take minutes. Surface progress.

**Documents with poor OCR**: chunks may have garbled text. Tag low-quality chunks (heuristic: many non-alpha characters in a row) and de-prioritize in retrieval.

**Rapidly evolving knowledge**: the firm's stance on Thai law changes. The assistant should prefer recent matters over older ones for legal-position questions. Apply a recency decay: weight chunks by `1 / (1 + months_since_creation)` during ranking.

**Privilege concerns**: never expose chunks from matters that are explicitly marked privileged or under retention hold (a future column on matters: `is_privileged boolean`).

**Client confidentiality demand**: some clients explicitly require that their data not be used as training or as cross-matter retrieval input. Add a per-matter setting "Exclude from firm-wide search" that blocks indexing.

**Anonymization**: when surfacing past matter content, redact party names and replace with generic placeholders ("the buyer", "the target") UNLESS the user has explicit access to the source matter. This protects privilege even within the firm.

**Conflict wall added retroactively**: when a wall is added, immediately remove already-cached results for affected users. Future queries respect the wall.

**Quality threshold**: don't return chunks with similarity score below a threshold (e.g., < 0.7). Avoid surfacing irrelevant noise.

**Retrieval limit per query**: cap at 10 chunks max. Beyond that, re-rank or deduplicate.

## Visual references

Harvey's matter intelligence and cross-matter search is the canonical reference. Linear's command-K search across issues is a UX reference for the search bar pattern. The footnote chip in chat is similar to Notion's database mention chips.

## Accessibility

- Search results are keyboard navigable (Tab through, Enter to open)
- Conflict warnings have proper `role="alert"` for screen readers
- Source citations link out clearly with `aria-label`s describing the destination

## Acceptance criteria

Documents upload and are indexed within 30s. Cross-matter search at `/search` returns relevant results across the firm's archive. Chat assistant on any matter automatically retrieves and cites cross-matter context. Conflict walls block walled content with a clear warning. Practice area classification works. Admin can manage walls. Privilege and confidentiality are respected via per-matter settings.

## Out of scope

- Cross-firm RAG (sharing knowledge with other firms) — never, unless explicit anonymized contribution
- Live web search blended with firm RAG (Brave/Serper integration) — separate brief
- Auto-anonymization of all firm content (we redact only when surfacing across access boundaries; full anonymization is a separate workstream)
- Vector store outside Supabase (we use pgvector for v1; if scale demands, migrate to Pinecone or Weaviate later)
- Multi-modal embeddings (images, signatures from PDFs) — text only in v1
- Client-portal access to RAG (clients should never see this)
- Automatic precedent suggestions while drafting (this is v3+ product work, beyond search)
