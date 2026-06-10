# SCM Assistant - Trinamix Junior AI Engineer Task

This repository contains the exported Chatflow and documentation for the Supply Chain Chatbot built using Flowise Cloud, submitted for the Trinamix Junior AI Engineer hiring task.

## Live Chatbot
**Public URL:** https://cloud.flowiseai.com/chatbot/ec189a73-209d-4902-a88e-fa4f37dde0f7

## Architecture & Technical Stack
This project implements a Dual-Agent Hybrid Architecture to safely and accurately handle both unstructured policy documents and structured supply chain data.

* **Orchestrator Agent (Main Chatflow):** A Tool Agent that handles user intent, routes queries to the appropriate tools, and synthesizes final answers. 
* **SQL Database Agent (Agentflow):** A dedicated microservice accessed via a Custom Tool. It safely converts natural language to SQL, queries a Supabase PostgreSQL database, and returns structured data. This isolation prevents sandbox errors and execution timeouts.
* **RAG Document Store:** Processes and retrieves governance policy rules from the provided PDF using Flowise's native document store.

**Models & Infrastructure Used:**
* **Orchestrator LLM:** Google Gemini (gemini-3.1-flash-lite-preview)
* **SQL Agent LLM:** Groq (llama-3.3-70b-versatile) - Chosen for high-speed, strict JSON tool-calling capabilities to prevent 504 timeouts.
* **Embeddings:** HuggingFace Inference (BAAI/bge-base-en-v1.5)
* **Vector Database:** Pinecone
* **Relational Database:** Supabase (PostgreSQL) - Hosting the `supplier_performance_data.csv` data(for future scalability).

## Chunking Configuration Experiments
As required, multiple chunking configurations were tested for the PDF Document Store using the Flowise PDF Loader:

1. **Configuration A: Large Chunks (Selected as Optimal)**
    * **Chunk Size:** 1200
    * **Chunk Overlap:** 250
    * **Result:** This configuration provided the best performance. The larger chunk size successfully encapsulated complete policy sections and tables (like the Supplier Tier Classification Framework) without fracturing the context. The 250-character overlap ensured that threshold rules were not separated from their corresponding tier definitions.

2. **Configuration B: Granular Chunks**
    * **Chunk Size:** 500
    * **Chunk Overlap:** 50
    * **Result:** While retrieval was highly specific, this configuration occasionally fragmented complex rules or multi-tier tables across different chunks. This caused the LLM to miss critical surrounding context when synthesizing answers that required comparing multiple data points.

## Sample Q&A Results
*(Note: The answers below are the verbatim outputs from the deployed chatbot)*

**Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**
> The following Tier-3 suppliers currently have an active disruption flag:

-   Longhua Polymer Works
-   Bohai Electronics
-   Dravex Components India
-   Deccan Polymer Mfg
-   Sahyadri Alloy Tech

> Per the BQBYTE Disruption Response Procedures, the applicable response level depends on the supplier's risk level:

-   Level 1 (Monitor): Applies to Low Risk suppliers. Requires weekly status updates and a 15% safety stock adjustment.
-   Level 2 (Manage): Applies to Medium Risk suppliers. Requires bi-weekly escalation calls, a 30% safety stock adjustment, and placing an alternate supplier on 48-hour readiness notice.
-   Level 3 (Activate): Applies to High Risk suppliers (or any supplier with two simultaneous disruption flags). Requires immediate escalation to the CPO, a 50% safety stock adjustment, activation of an alternate supplier for at least 40% of volume within 10 business days, and a full Root Cause Analysis (RCA) within 15 business days.

> Note: Certain disruption categories (export control restrictions, active labour strikes, regulatory enforcement actions, and port closures exceeding 72 hours) mandate a Level 3 response regardless of the supplier's risk level.

**Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?**
> There are 1,885 suppliers that qualify for the annual Volume Rebate Program.

**Q3: Which region has the highest total PO value, and does it breach the concentration limit?**
> The APAC region has the highest total PO value at $131,620,356.14.

> According to the BQBYTE Concentration Risk Rule (Section 5.3), no single region may account for more than 45% of total annual procurement spend. To determine if this limit is breached, the total annual procurement spend would need to be compared against the APAC total. Based on the current data, the APAC region represents the highest concentration of spend, and if this figure exceeds 45% of the total global procurement budget, it would constitute a breach requiring the submission of a Diversification Plan within 60 days.

**Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**
> The following suppliers are currently on Supplier Watch List (SWL) status:

- SUP-001
- SUP-008
- SUP-014
- SUP-021

> Restrictions: Per the BQBYTE policy, SWL status imposes the following restrictions:

New PO Issuance: Limited to a maximum of 20% of the supplier's prior quarter volume.
Activation: Suppliers on SWL status cannot be activated as an alternate supplier.
Additionally, any supplier with a Compliance Score below 60 at any audit is automatically placed on SWL status, and suppliers who are "Audit Overdue" (based on tier-specific time thresholds) are placed on provisional SWL status.

**Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**
> The product category with the highest average defect rate is Packaging Materials, with an average defect rate of 1.91%.

> According to the BQBYTE Supplier Tier Classification Framework (Section 3.2), the maximum permissible defect rate for Tier-2 suppliers is 2.50%. Therefore, the 1.91% defect rate for Packaging Materials does not exceed the Tier-2 limit.

## Future Improvements
If scaling this project for a production environment, I would implement the following optimizations:
1. **Dynamic Schema Injection:** Instead of hardcoding the SQL schema in the Agentflow prompt, implement a tool that dynamically fetches the schema from `information_schema` in Supabase. This ensures the agent remains robust even if database columns are updated in the future.
2. **Strict UI Schema Definitions:** Utilize native Input Schemas for Custom Tools rather than relying entirely on Javascript error-handling. This forces the LLM to output perfectly formatted JSON strings, eliminating hallucinated inputs and reducing unnecessary API round-trips.
3. **Human-in-the-Loop (HITL) for SQL:** Add an approval node where the generated SQL query is displayed to an administrator before executing read-heavy operations on live production databases.