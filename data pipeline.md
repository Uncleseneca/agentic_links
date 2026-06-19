This is a comprehensive system design interview framework for building a **Confluence-to-RAG Ingestion Pipeline**. You can use this structure to lead an interview, evaluate a candidate, or prepare for one.

---

## 1. Requirement Definition (The Setup)

Start by anchoring the conversation in constraints. If a candidate dives into coding without defining these, prompt them to clarify.

* **Scale:** 1M+ existing pages, ~5k daily updates.
* **Latency:** End-to-end freshness (change in Confluence to availability in RAG) < 15 minutes.
* **Access Control:** Strict adherence to Confluence group-based permissions. No leakage of restricted docs.
* **Data Types:** Standard text, complex HTML/Markdown tables, attachments (optional, but scope it).
* **Constraints:** Rate limiting by Confluence APIs, cost of token usage for embeddings.

---

## 2. High-Level Architecture (The Data Flow)

The system must handle asynchronous updates reliably. A decoupling approach is mandatory.

1. **1. Ingestion Layer (Webhooks/Poll):** Handles Confluence event stream.
Use a hybrid approach: **Webhooks** for low-latency updates and a **Scheduled Poller** for catching missed events/system recovery.


2. **2. Decoupling Queue:** Kafka or AWS SQS.
Ingested events go into a persistent queue. This isolates the source API from internal processing spikes.


3. **3. Transformation Service:** Structure-aware parsing.
Extract, normalize, and chunk. **Must** extract ACL metadata (`group_ids`) here to bind it to chunks.


4. **4. Embedding Service:** Vectorization.
Convert text chunks into dense vectors. Apply batching to optimize cost and API throughput.


5. **5. Vector Store:** Storage & Indexing.
Upsert vectors + scalar metadata (for ACL filtering).


---

## 3. Technical Deep Dives (The Core Questions)

Use these questions to probe the candidate's understanding of distributed systems and data engineering.

### Q1: The Consistency Problem (Backfill vs. Stream)

**The Scenario:** A backfill of 1M documents is running. Simultaneously, a user edits Page A.
**Candidate should explain:**

* **Idempotency:** Using `SHA256(page_id + chunk_index)` to ensure deterministic vector IDs.
* **Version tracking:** Using a version timestamp in the Vector DB metadata to ensure old backfill records don't overwrite newer events.

### Q2: Managing Rate Limits & Backpressure

**The Scenario:** A mass update occurs (e.g., a corporate restructuring). 50k pages need updates at once.
**Candidate should explain:**

* **Leaky Bucket/Token Bucket:** Throttling workers based on API quotas.
* **Exponential Backoff with Jitter:** How to handle `HTTP 429` (Too Many Requests) without crashing the downstream embedding service.

### Q3: Security & ACL Synchronization

**The Scenario:** A document moves from "Public" to "Restricted."
**Candidate should explain:**

* **Metadata Filtering:** Stamping chunks with `allowed_groups` during ingestion.
* **Two-Tier Verification:** Performing an initial vector search (coarse filter), followed by a real-time permission check against the Identity Service (fine filter) before passing data to the LLM.

### Q4: Semantic Table Parsing

**The Scenario:** A Confluence table with 10 columns containing pricing data.
**Candidate should explain:**

* **Linearization:** Converting tables into structured natural language (e.g., "The value of X in row Y is Z") rather than just dumping raw HTML.
* **Context Injection:** Prepending the page title and section header to every chunk to provide global context to the LLM.

---

## 4. Probing Follow-Up Questions (The "Test")

Use these to push the candidate past standard textbook answers.

1. **The "Embedding Drift" Question:** "The organization decides to switch embedding models to a more powerful, higher-dimension model. We have 1M+ vectors in production. How do we migrate without taking the system offline or incurring massive costs?"
* *Expectation:* Discussing a "Shadow Index" strategy (creating a new index, backfilling in the background, and atomically switching the traffic alias).


2. **The "Ghost Data" Question:** "We delete a Confluence space with 10k pages. Our pipeline handles individual page deletes fine, but how do we ensure we efficiently purge all 10k documents from the Vector DB without issuing 10k separate delete commands?"
* *Expectation:* Discussing batch deletions via scalar metadata indexing (`DELETE WHERE space_id = 'X'`).


3. **The "RAG Observability" Question:** "The system is live, but users complain the RAG answers are 'dumb' or outdated. How do we build an observability loop to debug the pipeline?"
* *Expectation:* Discussing tracing (OpenTelemetry), logging the exact chunk retrieved vs. the final LLM answer, and implementing a "human-in-the-loop" feedback mechanism (thumbs up/down) linked to specific document versions.


4. **The "Cold Start" Question:** "How do we handle the cold start problem for the first 1,000 pages during the initial backfill to get the system usable as fast as possible?"
* *Expectation:* Prioritization logic (e.g., process "Most Viewed" or "Recently Updated" pages first to get immediate value while the rest of the queue processes).



---

### Interview Rubric for the Evaluator

* **Junior:** Focuses on the "how" (e.g., writing the code to fetch a page).
* **Mid-Level:** Focuses on the "architecture" (e.g., using queues, databases, and standard patterns).
* **Senior/Staff:** Focuses on the "failure modes" (e.g., what happens when the API is down? How do we handle partial failures? How does this evolve in 2 years?).

Which of these deep-dive areas (Consistency, Resiliency, Security, or Observability) would you like to build out further into a detailed "solution architecture" document?