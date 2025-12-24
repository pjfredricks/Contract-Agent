# Contract Intelligence Agent - Project Specification

## Executive Summary
Build an agentic RAG system that analyzes legal contracts, answers questions about them, and performs intelligent reasoning over multiple documents. This project demonstrates production-ready AI patterns: vector search, multi-turn reasoning, tool calling, and document intelligence.

## Business Context
**Problem:** Legal teams spend hours manually reviewing contracts for due diligence, compliance checks, and risk assessment. A typical M&A deal involves reviewing 50-200 contracts.

**Solution:** An AI agent that can:
- Answer questions about contract terms ("What's the termination clause in the Acme vendor agreement?")
- Compare across documents ("Which contracts have auto-renewal clauses?")
- Identify risks ("Flag any unlimited liability provisions")
- Generate summaries and reports

**Market Fit:** Legal tech, M&A firms, procurement teams, contract lifecycle management (CLM) platforms.

---

## Core Features (MVP)

### 1. Document Ingestion
- Upload PDF contracts (5-50 pages typical)
- Extract text with structure preservation (sections, clauses)
- Chunk intelligently (semantic chunking, not just fixed 512 tokens)
- Generate embeddings and store in vector database

### 2. Semantic Search (RAG Foundation)
- Query: "What are the payment terms?"
- Retrieve: Top 5 relevant chunks from vector DB
- Context window: Combine chunks + original query → LLM

### 3. Agentic Reasoning (The Key Differentiator)
The agent can perform multi-step reasoning:
- **Clarification:** "I found 3 payment clauses. Do you want the milestone schedule or the NET-30 terms?"
- **Cross-reference:** "The NDA references Section 4.2 of the Master Agreement. Let me check that..."
- **Tool use:** Call functions to extract structured data (dates, dollar amounts, parties)

### 4. Tool Calling Capabilities
The agent has access to tools:
- `search_contracts(query)` - Semantic search across all documents
- `extract_clause(contract_id, clause_type)` - Pull specific clause types (termination, liability, etc.)
- `compare_terms(contract_ids, term_type)` - Side-by-side comparison
- `calculate_dates(start_date, duration)` - Date math for terms

### 5. Conversational Memory
- Track conversation context across multiple turns
- "What about the Acme contract?" (agent remembers we're discussing vendor agreements)
- Session management for multi-document analysis

---

## Technical Architecture

### Tech Stack (Spring Boot + AI)
```
┌─────────────────────────────────────────────────┐
│  Frontend: React (simple chat UI)              │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  Spring Boot REST API                           │
│  - Spring AI (LLM abstraction)                  │
│  - Controller: /api/chat, /api/upload           │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  Agent Orchestration Layer                      │
│  - ConversationService (manages turns)          │
│  - AgentExecutor (tool calling loop)            │
│  - ToolRegistry (registers available tools)     │
└─────────────────────────────────────────────────┘
                      ↓
┌──────────────────┬──────────────────┬───────────┐
│  Vector Store    │  LLM Provider    │  Tools    │
│  (Pinecone or    │  (OpenAI GPT-4o  │           │
│   Weaviate or    │   or Claude)     │  Custom   │
│   pgvector)      │                  │  Java     │
└──────────────────┴──────────────────┴───────────┘
```

### Data Flow (Single Query)
1. **User:** "What's the liability cap in the Acme contract?"
2. **API:** Receives query → creates conversation context
3. **Agent:** Decides to use `search_contracts("liability cap Acme")`
4. **Vector DB:** Returns top 5 chunks (cosine similarity > 0.8)
5. **LLM:** Receives chunks + query → generates answer
6. **Agent:** Checks if answer is complete
   - If yes → return to user
   - If no → call another tool (e.g., `extract_clause` for full text)
7. **Response:** "The liability cap is $500,000 per incident (Section 8.3)"

---

## Implementation Phases

### Phase 1: RAG Foundation (Week 1-2)
**Goal:** Get basic Q&A working over a single contract

**Tasks:**
- Set up Spring Boot project with Spring AI
- Integrate OpenAI API (or Claude via Anthropic SDK)
- Implement PDF ingestion pipeline (Apache PDFBox)
- Chunk documents (simple: 500 tokens, 50 overlap)
- Generate embeddings (OpenAI `text-embedding-3-small`)
- Store in vector database (start with Pinecone free tier)
- Build `/api/query` endpoint: query → embed → search → LLM → response

**Deliverable:** Can ask "What is the term length?" and get an answer from a single uploaded contract.

### Phase 2: Agentic Layer (Week 2-3)
**Goal:** Add multi-turn reasoning and tool calling

**Tasks:**
- Implement `AgentExecutor` class (tool calling loop)
- Create 3 tools:
  - `search_contracts(query)` - wraps vector search
  - `extract_clause(contract_id, clause_type)` - regex + LLM extraction
  - `get_full_section(contract_id, section_number)` - retrieves entire section
- Use function calling (OpenAI functions or Claude tool use)
- Add conversation memory (store last 5 turns in session)

**Deliverable:** Agent can handle "What's the termination clause?" → realizes it needs Section 12 → calls `get_full_section` → returns complete answer.

### Phase 3: Multi-Document Intelligence (Week 3-4)
**Goal:** Reason across multiple contracts

**Tasks:**
- Upload 5-10 sample contracts (use public templates or generate with Claude)
- Implement `compare_terms(contract_ids, term_type)` tool
- Add metadata filtering (contract type, date, parties)
- Build summary endpoint: `/api/summarize` (generates executive summary of all contracts)

**Deliverable:** Can answer "Which contracts have auto-renewal?" and return a list with references.

### Phase 4: Production Polish (Week 4-5)
**Goal:** Make it demo-ready and portfolio-worthy

**Tasks:**
- Add authentication (Spring Security + JWT)
- Rate limiting (bucket4j)
- Error handling (retry logic, fallback responses)
- Monitoring (log token usage, query latency)
- Frontend UI (React chat interface with file upload)
- Write tests (JUnit for tools, integration tests for agent)
- Deploy (Railway, Render, or AWS)
- Document: README with architecture diagram, API examples

**Deliverable:** Publicly accessible demo with GitHub repo.

---

## Key Technical Challenges (And How to Solve Them)

### Challenge 1: Chunking Strategy
**Problem:** Fixed 500-token chunks break mid-sentence or split clauses.

**Solution:**
- Use semantic chunking (LangChain's `RecursiveCharacterTextSplitter` pattern)
- Preserve section headers in chunks
- Add overlap (50-100 tokens) to maintain context

### Challenge 2: Hallucination Control
**Problem:** LLM makes up contract terms not in the document.

**Solution:**
- Always return source citations (chunk IDs, page numbers)
- Use prompts that enforce "Answer ONLY from provided context"
- Add confidence scores (if similarity < 0.75, say "I'm not confident")

### Challenge 3: Tool Calling Loop
**Problem:** Agent gets stuck in infinite loops or calls wrong tools.

**Solution:**
- Limit tool calls per turn (max 5)
- Add explicit tool descriptions ("Use this when user asks about dates")
- Log all tool calls for debugging

### Challenge 4: Cost Management
**Problem:** Embedding 100 pages = expensive.

**Solution:**
- Cache embeddings (don't re-embed same document)
- Use cheaper models for embeddings (`text-embedding-3-small` not `large`)
- Batch embedding requests (embed 100 chunks at once, not 1 by 1)

---

## Sample Contracts for Testing
You'll need realistic data. Here are sources:

1. **Generate with Claude:** "Create a 5-page vendor service agreement with standard clauses"
2. **Public templates:** EDGAR (SEC filings), LegalZoom templates
3. **Anonymize real contracts:** Remove company names, use placeholder data

**Suggested test set (5 contracts):**
- Vendor Service Agreement (SaaS)
- Non-Disclosure Agreement (NDA)
- Employment Agreement
- Master Services Agreement (MSA)
- Equipment Lease Agreement

---

## Success Metrics

### Technical Metrics
- **Retrieval accuracy:** Top 5 chunks contain answer >90% of time
- **Response latency:** <3 seconds for simple queries, <10s for multi-step
- **Token efficiency:** <2000 tokens per complex query (optimize prompt)

### Demo Metrics
- Can answer 20 sample questions correctly
- Handles follow-up questions (conversational memory works)
- Flags "I don't know" when answer isn't in documents (no hallucination)

---

## Resume Lines You Can Claim

After completing this project:

✅ "Built agentic RAG system with multi-turn reasoning over legal contracts"  
✅ "Implemented semantic search using vector embeddings and Pinecone"  
✅ "Integrated OpenAI function calling for tool orchestration"  
✅ "Optimized chunking strategy to improve retrieval accuracy by 35%"  
✅ "Deployed production-ready AI agent with auth, rate limiting, and monitoring"

---

## Extensions (If You Want to Go Further)

### Advanced Features
- **Structured extraction:** Pull all parties, dates, dollar amounts into a table
- **Risk scoring:** Flag high-risk clauses (unlimited liability, no termination rights)
- **Change tracking:** Compare two versions of same contract, highlight differences
- **Export reports:** Generate PDF summary of all contracts analyzed

### Technical Depth
- **Fine-tuning:** Fine-tune a small model (GPT-3.5) on contract Q&A pairs
- **Hybrid search:** Combine vector search + keyword search (BM25)
- **Multi-modal:** Extract tables and images from contracts (OCR)

---

## Getting Started Checklist

- [ ] Clone Spring Boot starter template
- [ ] Sign up for OpenAI API key (or Anthropic)
- [ ] Sign up for Pinecone (or run pgvector locally)
- [ ] Download 5 sample contracts (PDFs)
- [ ] Build Phase 1: Basic RAG (Week 1-2)
- [ ] Iterate to Phase 4 (Week 3-5)
- [ ] Deploy and document

---

## Resources

**Spring AI Docs:** https://docs.spring.io/spring-ai/reference/  
**OpenAI Function Calling:** https://platform.openai.com/docs/guides/function-calling  
**Pinecone Quickstart:** https://docs.pinecone.io/docs/quickstart  
**RAG Patterns:** https://www.anthropic.com/research/retrieval-augmented-generation

---

## Final Thoughts

This project hits the sweet spot: complex enough to be impressive, focused enough to finish in 4-6 weeks. It teaches you the patterns that companies are hiring for right now—RAG, agents, tool calling, and production AI.

When you're done, you'll have a portfolio piece that demonstrates you can build real AI systems, not just call an API once.

Now go build it.
