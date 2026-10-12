# Design doc: Addback, a corporation tax adjustments review agent

Last updated: 12 October 2026. Status: draft v1. Build in progress.

## 1. Problem and goals

**Problem.** When preparing a UK company's corporation tax computation, tax staff go line by line through the trial balance or nominal ledger to find disallowable and capital items to add back. Mid-tier firms and their SME clients have no tool that does a trustworthy first pass with HMRC citations.

**User.** A tax associate or senior at a mid-tier UK accounting firm preparing a corporation tax computation, plus the manager who reviews it.

**Job to be done.** "Give me a first-pass list of likely add-backs from this ledger, with the HMRC reason for each, so I can review it quickly and key the adjustments into my computation software."

**Goals**
- Find disallowable and capital items in a ledger export with high recall.
- Every proposed adjustment cites a specific HMRC manual paragraph that supports it.
- A human approves, edits or rejects every line; every decision is logged.

**Success looks like**
- Recall on disallowable and capital lines of at least 0.90 on the labelled test ledgers (missing an add-back is the costly error).
- Precision of at least 0.80.
- Citation accuracy of at least 0.90 (the cited paragraph exists and supports the claim).
- Ragas faithfulness of at least 0.85 on the manual Q&A set.
- A 150-line ledger processed in under 2 minutes, at a known cost per ledger.

These are targets to measure against, not claims.

## 2. Requirements

**Functional**
1. Upload a ledger CSV (account code, account name, description, amount, date).
2. Classify each line: allowable, disallowable, capital, or needs judgement.
3. For non-allowable lines, propose an add-back amount, a plain-English reason, a manual citation (e.g. BIM45020) and a confidence score.
4. Reviewer view: accept, edit or reject each line, and ask the agent "why" about any line.
5. Export an adjustments schedule (CSV) and an audit log.

**Non-functional**
- **Legally grounded:** retrieval restricted to gov.uk HMRC manuals (BIM, CTM, CA). No web search.
- **Human-supervised:** nothing is final without a reviewer decision (HMRC guidance on generative AI in tax software, January 2026).
- **Transparent:** the UI shows that AI is used, the source paragraph, and the confidence.
- **Secure:** managed identity, Key Vault, no client data (synthetic ledgers only), ledger text treated as untrusted input.
- **Cost:** within a £20 a month budget alert; scale to zero when idle.
- **Latency:** under 2 minutes for 150 lines.

## 3. Architecture overview

![Architecture](architecture.png)

Request path for one ledger:

1. Reviewer uploads a ledger CSV in the web UI (Container App).
2. The API (FastAPI, Container App) validates and stores the file in Blob Storage.
3. **Rules pre-pass (no AI):** obvious lines are tagged by account-name rules (e.g. "Depreciation", "Amortisation", "Fines") with a fixed citation.
4. **Prompt Shields check** on ledger descriptions, because a description is free text that could carry injected instructions.
5. **Retrieval:** for each remaining line, a hybrid query (keyword plus vector) to Azure AI Search over the HMRC manual index, with semantic ranking on Basic tier.
6. **Classification:** one Azure OpenAI call per batch of lines with structured output (JSON schema): treatment, amount, reason, citation, confidence.
7. **Citation check:** deterministic check that the cited paragraph ID is in the index, then a cheap model call that confirms the retrieved passage supports the reason. Failures drop to "needs judgement".
8. **Review agent (Foundry Agent Service):** when the reviewer asks "why?" or opens a "needs judgement" line, an agent with tools investigates and answers with citations.
9. Reviewer decisions go to an audit log (Table Storage). Export produces the adjustments schedule.

**Agent tools (function tools exposed by the API)**
- `search_manuals(query)`: hybrid search over the HMRC manual index.
- `get_paragraph(ref)`: fetch the full text of a manual paragraph by reference.
- `get_ledger_context(line_id)`: the line plus nearby lines in the same account.
- `propose_adjustment(line_id, treatment, amount, citation, reason)`: writes a proposal for the reviewer; cannot finalise.

**Azure resources (all in Terraform)**
- Resource group, Log Analytics workspace, Application Insights
- Azure AI Foundry resource and project, model deployments (chat and embeddings)
- Azure AI Search (Free for development, Basic for the evaluation week)
- Container Apps environment with two apps (api, ui), Azure Container Registry
- Storage account (Blob for uploads and the corpus, Table for the audit log)
- Key Vault, user-assigned managed identity, role assignments (Search Index Data Reader, Cognitive Services OpenAI User, Storage Blob Data Contributor, Key Vault Secrets User, AcrPull)
- Content Safety (Prompt Shields)
- Budget alert at £20

## 4. Decision log

Format: Decision, Options, Choice, Why, Trade-offs.

**D1. Core pattern**
- Options: plain RAG chat; fully agentic loop per line; deterministic pipeline plus an agent for review.
- Choice: deterministic pipeline for the bulk pass, Foundry agent for review and hard cases.
- Why: 150 lines through an agent loop is slow, costly and harder to test. The bulk pass is a known sequence (retrieve, classify, verify). The agent earns its place where the path is not known in advance: answering "why" and investigating ambiguous lines.
- Trade-offs: two code paths to maintain; the bulk pass cannot ask follow-up questions on its own.

**D2. Rules before AI**
- Options: send every line to the model; rules pre-pass for unambiguous items.
- Choice: rules pre-pass (depreciation, amortisation, fines and penalties, political donations) with fixed citations.
- Why: these are certain, so a model adds cost and risk.
- Trade-offs: rules miss odd naming; anything not matched still goes to the model.

**D3. Model and region**
- Options: gpt-4.1 family in UK South; GPT-5.x via Global Standard; GPT-5.x via Data Zone Standard (EU).
- Choice: a current small GPT-5.x model for classification and the agent, a smaller model for citation checks, text-embedding-3-small for embeddings. Data Zone Standard (EU) in a supported EU region if available, otherwise Global Standard. Model names are Terraform variables.
- Why: the gpt-4.1 family is scheduled to retire in October 2026, and new deployments in UK South have been unreliable. Data Zone EU keeps processing in the EU; the demo uses public manuals and synthetic data.
- Trade-offs: not UK-resident processing. A real firm deployment would need UK regional deployment or provisioned throughput. Availability changes month to month, so this is the riskiest assumption (see section 8).

**D4. Embedding size**
- Options: text-embedding-3-small at 1,536 dimensions; reduced to 512; text-embedding-3-large.
- Choice: text-embedding-3-small with reduced dimensions (512), validated against 1,536 on the eval set.
- Why: keeps the index inside the Free tier's 50 MB during development; the small model is cheap.
- Trade-offs: possible recall loss, which the eval measures. If it drops, go to 1,536 on Basic.

**D5. Vector store and search tier**
- Options: Azure AI Search Free; AI Search Basic; ChromaDB self-hosted.
- Choice: AI Search Free during development, Basic only for the evaluation week (semantic ranker needs Basic or higher; Free has 50 MB and 3 indexes). ChromaDB runs locally as the comparison baseline.
- Why: uses the managed service while keeping cost to days of Basic rather than months.
- Trade-offs: semantic ranker results only during Basic days; Free services can be deleted after inactivity, so the index must rebuild from code.

**D6. Corpus scope**
- Options: all HMRC manuals; BIM, CTM and CA manuals in full; targeted sections only.
- Choice: targeted sections of BIM (deductions, entertainment, capital vs revenue, legal and professional costs, fines), CTM computation sections and the CA manual introduction.
- Why: smaller, cleaner index; fits the Free tier; covers the items in the eval set.
- Trade-offs: out-of-scope items fall to "needs judgement".

**D7. Chunking**
- Options: fixed token windows; one chunk per manual paragraph.
- Choice: one chunk per manual paragraph, with the reference and title as fields.
- Why: citations map cleanly to a paragraph ID, which makes citation checking deterministic.
- Trade-offs: some paragraphs are long; split those with the same reference plus a part number.

**D8. Orchestration framework**
- Options: LangGraph in the container; Semantic Kernel; Foundry Agent Service with function tools; no framework.
- Choice: plain Python for the bulk pipeline, Foundry Agent Service for the review agent.
- Why: managed agent runtime with built-in tracing; keeps the bulk path simple.
- Trade-offs: less portable than LangGraph; tied to Azure.

**D9. Structured output**
- Options: free text then parse; JSON mode; JSON schema structured output.
- Choice: JSON schema structured output, validated with Pydantic. Invalid output is retried once, then marked "needs judgement".
- Why: removes parsing failures and makes outputs testable.
- Trade-offs: schema changes need prompt and test updates.

**D10. Prompt injection defence**
- Options: none; regex filters; Prompt Shields plus prompt structure.
- Choice: Prompt Shields on ledger descriptions, ledger text passed as data in a delimited field, agent tools cannot finalise anything.
- Why: ledger descriptions are free text from client systems.
- Trade-offs: extra call and latency; possible false positives on odd descriptions.

**D11. Front end**
- Options: Streamlit; FastAPI with HTMX templates; React.
- Choice: Streamlit as a second Container App.
- Why: fastest path to a reviewer table with accept, edit, reject.
- Trade-offs: less polished; limited custom layout.

**D12. Observability**
- Options: Langfuse; Application Insights with OpenTelemetry; both.
- Choice: Application Insights with OpenTelemetry, plus Foundry tracing for the agent.
- Why: stays inside Azure alongside the rest of the stack.
- Trade-offs: less LLM-specific UI than Langfuse.

**D13. Audit log store**
- Options: SQLite in the container; Cosmos DB; Azure Table Storage.
- Choice: Table Storage.
- Why: cheap, durable, enough for append-only decisions.
- Trade-offs: limited querying.

## 5. Evaluation plan

**Test data (synthetic, no client data)**
- 3 ledgers of about 50 lines each (150 lines total) for a small trading company, a professional services firm and a property company.
- Each line labelled with treatment, add-back amount and the correct manual reference, drawn from HMRC manual examples and common ledger descriptions.
- About 30 percent of lines non-allowable, including tricky ones (staff party vs client entertaining, repairs vs improvements, legal fees on a lease vs on a dispute).
- 3 lines with injected instructions in the description to test Prompt Shields.

**Metrics**
- Classification: recall and precision on disallowable and capital lines; confusion matrix.
- Citation accuracy: share of proposals whose cited paragraph exists and supports the reason (model check plus manual spot check of 30).
- Retrieval: Ragas context precision and faithfulness on a 25-question manual Q&A set.
- Operations: latency per ledger, tokens and cost per ledger.
- Security: injected lines blocked or neutralised.

**Comparison:** same question set and ledgers with Azure AI Search (hybrid, with and without semantic ranker) vs. ChromaDB plus BM25. Covers quality, setup time, cost, control and debuggability.

**How it runs:** `make eval` runs everything and writes results to /eval/results and the README table. CI runs a 10-line smoke eval on each pull request.

## 6. Risks and failure modes

| Risk | Mitigation |
|---|---|
| Model not deployable in chosen region (see D3) | Test deployment first; model and region are variables; fallback to Global Standard |
| Hallucinated or wrong citation | Citation must exist in the index; support check; else "needs judgement" |
| Missed add-back (false negative) | Recall-first threshold: low confidence goes to review, not to allowable |
| Prompt injection via ledger text | Prompt Shields; data delimited; tools cannot finalise |
| Outdated guidance | Corpus refresh script; corpus version and date shown in the UI |
| Rate limits or timeouts | Batching, retries with backoff, per-ledger status |
| Free Search service deleted when idle | Index rebuilt from code in one command |
| Over-reliance by users | AI disclosure, mandatory review step, reminder that the preparer remains responsible |

## 7. Cost and scalability

| Component | Plan | Cost driver |
|---|---|---|
| Azure OpenAI / Foundry models | Pay per token | Tokens per ledger; measured in eval |
| Foundry Agent Service | No runtime charge for Foundry-native agents | Model tokens and tools only |
| Azure AI Search | Free (dev), Basic (eval week only) | Basic billed while it exists; delete after eval |
| Container Apps | Consumption, scale to zero | Monthly free grant of vCPU-seconds, GiB-seconds and requests |
| Storage, Key Vault, App Insights | Pay as you go | Small at this scale |
| Content Safety | Pay per call | One call per batch |

Controls: £20 budget alert before any deploy; rate limit on the public demo; token cap per ledger; `terraform destroy` for Basic Search after the eval week.

Scaling: per-ledger work is independent, so it scales horizontally with queue-based processing (Service Bus) if needed. Out of scope for the MVP.

## 8. Validate before building

1. **Riskiest assumption: model deployment.** List available models in the target region and deploy the chosen chat and embedding models with Terraform in a throwaway resource group.
2. **Index size.** Scrape the targeted manual sections, chunk by paragraph, embed at 512 dimensions, and check the size fits under 50 MB.
3. **Tiny classification test.** 20 hand-labelled lines through one prompt. If recall is below about 0.7, rethink retrieval before building.
4. **Trace one scenario** end to end: "Xmas party - clients - £2,400" through rules, Prompt Shields, retrieval (BIM45020 area), classification, citation check, reviewer accept, export.

## 9. Out of scope

Filing, the full computation, capital allowances pools, group relief, losses, transfer pricing, real client data, multi-tenant accounts, UK-resident model processing.

## 10. Sources

- Azure AI Search service limits: https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity
- Foundry model region availability: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability
- Foundry Agent Service pricing: https://azure.microsoft.com/en-us/pricing/details/foundry-agent-service/
- Container Apps pricing: https://azure.microsoft.com/en-us/pricing/details/container-apps/
- HMRC guidance on generative AI in tax software (summary): https://vatcalc.com/united-kingdom/uk-hmrc-guidance-generative-ai-in-tax-software
- HMRC BIM45020: https://www.gov.uk/hmrc-internal-manuals/business-income-manual/bim45020
