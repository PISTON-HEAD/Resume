# Complete AI Engineering Roadmap — Full Stack Edition
### 1–2 Hours/Day | 14–16 Months | Java + Python + Spring Boot + Docker + AI

> **Target:** Become a hireable AI Engineer who can build production-grade AI applications, RAG pipelines, agents, and MCP-powered tools — while keeping your Java/Spring Boot/Docker skills sharp throughout.

---

## How This Roadmap Works

There are two parallel tracks running simultaneously:

| Track | What It Builds |
|-------|---------------|
| **AI Track** | LLMs → Prompt Engineering → Embeddings → RAG → Agents → MCP |
| **Java Revision Track** | Spring Boot → JPA → Security (JWT) → Microservices → Docker → Docker Compose |

The Java revision projects are **not separate** from the AI projects. They are deliberately designed to feed into each other. By Month 9, your Spring Boot microservices become the tools your AI agent calls. This is how real AI engineering jobs work.

---

## Month 1: AI Literacy — Understand the Landscape

**Goal:** Build mental models before writing a single line of code.

### What to Learn
- AI vs ML vs Deep Learning vs Generative AI (nested concepts, not separate fields)
- What is an LLM — how it was trained, what tokens are, what a context window is
- What is a prompt and why prompt engineering is a real engineering skill
- What are embeddings and vectors (conceptually — no math yet)
- What is RAG and why it exists (LLMs don't know your private data)
- Fine-tuning vs RAG — understand the difference early
- Key players: OpenAI, Anthropic, Google (Gemini), Meta (Llama), Mistral
- What is an AI agent and what is MCP (awareness only for now)

### Where to Learn

| Resource | What You Get | Time |
|----------|-------------|------|
| [Andrej Karpathy - Intro to LLMs](https://www.youtube.com/watch?v=zjkBMFhNj_g) | Best 1-hour conceptual LLM overview ever made | 1 hour |
| [3Blue1Brown - Neural Networks playlist](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi) | Visual, intuitive, builds real understanding | ~3 hours |
| [Google - Generative AI Learning Path](https://www.cloudskillsboost.google/paths/118) | Free, structured, no math | ~5 hours |
| [Anthropic - What is MCP?](https://www.anthropic.com/news/model-context-protocol) | High-level MCP awareness | 30 min |

### ✅ By End of Month 1 You Can
- Explain what an LLM is and how it works conceptually
- Explain the difference between fine-tuning and RAG
- Explain why vector databases exist
- Know what MCP is at a high level

---

## Month 2: Python — Just Enough, Nothing More

**Goal:** Read and write Python for AI tasks. You already think like a developer — Python is just syntax.

### What You Need

**Must Know:**
- Variables, strings, lists, dictionaries, loops, functions
- Classes (simpler than Java — no boilerplate)
- `pip` and virtual environments (like Maven but simpler)
- File I/O, working with JSON
- List comprehensions (very common in AI code)
- `async`/`await` basics
- Running `.py` files and Jupyter notebooks

**Skip For Now:**
- Django/Flask, decorators, testing frameworks, numpy/pandas

### Where to Learn
- **[Python for Java Developers - YouTube](https://www.youtube.com/results?search_query=python+for+java+developers)** — Best for you. Search this, find a 2–3 hour crash course aimed at Java developers.
- **[Python.org official tutorial](https://docs.python.org/3/tutorial/)** — Chapters 1–9 only.

### ✅ By End of Month 2 You Can
- Write Python that reads files, calls APIs, handles JSON
- Run Jupyter notebooks
- Read any AI tutorial on GitHub without confusion

---

## Month 3: First LLM App + Spring Boot Revision (JPA)

**Goal:** Build your first AI app AND refresh Spring Boot + JPA simultaneously through one combined project.

### AI Learning This Month
- Calling an LLM API (OpenAI or Ollama locally)
- System prompts vs user prompts
- Multi-turn conversation basics
- Running local models free with Ollama

### Java Revision This Month — JPA + REST API
**What you're revising:** Entity design, JPA repositories, relationships (OneToMany, ManyToOne), REST controllers, service layer, proper layered architecture.

---

### 🔨 Project 1: AI-Powered Notes App — Spring Boot + JPA + Basic Chat
**GitHub repo: `smart-notes-api`**

**What it does:**
- A REST API where users can create, store, and retrieve notes (standard Spring Boot CRUD)
- An AI endpoint that takes a note's content and returns a summary, key points, or tags
- Conversation history stored in the database (so you revise JPA while building AI features)

**Spring Boot / JPA scope:**
- `Note` entity with JPA (`@Entity`, `@Id`, `@GeneratedValue`)
- `ChatHistory` entity to store conversation turns
- `NoteRepository`, `ChatHistoryRepository` — JPA repositories
- REST controllers: `NoteController`, `ChatController`
- Service layer with proper separation

**AI scope:**
- Call OpenAI or Ollama from Spring AI
- Send note content to LLM, return summary as JSON
- Store AI responses in the database

**Tech Stack:** Spring Boot + Spring Data JPA + H2 (for simplicity) + Spring AI + Ollama

```java
@Entity
public class Note {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    @Column(columnDefinition = "TEXT")
    private String content;
    private LocalDateTime createdAt;
    // getters, setters
}

@Entity
public class ChatHistory {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @ManyToOne
    private Note note;
    private String role; // "user" or "assistant"
    @Column(columnDefinition = "TEXT")
    private String message;
    private LocalDateTime timestamp;
}
```

**Python version (do this too):**
Build the same chat functionality in a Python script using OpenAI SDK or Ollama — so you practice both languages.

### Where to Learn (AI Side)

| Resource | Notes |
|----------|-------|
| [Spring AI Docs](https://docs.spring.io/spring-ai/reference/) | Your Java AI implementation reference |
| [OpenAI Cookbook](https://cookbook.openai.com) | Python examples |
| [Ollama](https://ollama.com) | Free local LLMs — `ollama run llama3.2` |

### ✅ By End of Month 3 You Can
- Call an LLM from both Python and Java/Spring Boot
- Build a Spring Boot REST API with JPA (refreshed)
- Store and retrieve conversation history from a database
- Have Project 1 committed to GitHub

---

## Month 4: Prompt Engineering + Context Management + Spring Security (JWT)

**Goal:** Learn prompt engineering properly AND revise Spring Security with JWT.

### AI Learning This Month
- System prompts vs user prompts — how each shapes behavior
- Few-shot prompting (examples inside the prompt)
- Chain-of-thought prompting
- Structured output — making the LLM return JSON
- Temperature, max_tokens, top_p — what these parameters actually do
- **Context management** — what happens when you hit the context window limit, and how to handle it (summarization, sliding window, trimming)

### Java Revision This Month — Spring Security + JWT
**What you're revising:** SecurityFilterChain configuration, JWT generation and validation, stateless authentication, protecting endpoints with roles.

---

### 🔨 Project 2: Secure AI Assistant API — Spring Boot + JWT + Prompt Engineering
**GitHub repo: `secure-ai-assistant`**

**What it does:**
- Extends Project 1 with proper JWT authentication
- Users register and login — get a JWT token
- All AI endpoints are protected (require valid JWT)
- Each user has their own notes and chat history (proper data isolation)
- The AI endpoint uses well-engineered prompts to return structured JSON responses

**Spring Security / JWT scope:**
- `UserEntity` with roles
- JWT filter — validate token on every request
- `SecurityFilterChain` — public routes (register/login) vs protected routes
- Password hashing with BCrypt
- User-scoped data — a user can only access their own notes

**Prompt Engineering scope:**
- System prompt that instructs the LLM to always return valid JSON
- Few-shot examples in the prompt to guide output format
- Context window handling — if conversation history is too long, summarize older messages before sending

**Tech Stack:** Spring Boot + Spring Security + JWT (jjwt library) + JPA + PostgreSQL + Spring AI

```java
// JWT protected endpoint
@PreAuthorize("isAuthenticated()")
@PostMapping("/chat")
public ResponseEntity<ChatResponse> chat(
    @RequestBody ChatRequest request,
    @AuthenticationPrincipal UserDetails user) {
    return ResponseEntity.ok(chatService.chat(user.getUsername(), request));
}
```

### Where to Learn (AI Side)

| Resource | What It Covers |
|----------|---------------|
| [LearnPrompting.org](https://learnprompting.org) | Free, comprehensive |
| [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) | Official best practices |
| [Anthropic Prompt Engineering Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) | Concepts apply to all models |

### ✅ By End of Month 4 You Can
- Write prompts that produce reliable JSON output
- Handle context window limits in production
- Implement JWT authentication in Spring Boot (refreshed)
- Have Project 2 on GitHub

---

## Month 5: Embeddings + Vector Databases + Docker Revision

**Goal:** Understand embeddings and vector search — AND Dockerize everything you've built.

### AI Learning This Month
- Text → vectors: how embedding models work
- Cosine similarity — how vector search finds related content
- pgvector — PostgreSQL extension for vector storage
- Chroma — Python vector database for local experiments
- Choosing chunk sizes and why it matters

### Docker Revision This Month
**What you're revising:** Dockerfile creation, docker-compose.yml, multi-container setups, environment variables, volume mounts, networking between containers.

---

### 🔨 Project 3: Semantic Search Service — Embeddings + Docker Compose
**GitHub repo: `semantic-search-service`**

**What it does:**
- A Spring Boot service that ingests documents and creates embeddings
- A search endpoint: user sends a natural language query, gets back semantically similar documents (NOT keyword search)
- Everything runs in Docker Compose — app + PostgreSQL + pgvector

**Docker scope:**
- Write a proper `Dockerfile` for your Spring Boot app
- Write a `docker-compose.yml` with:
  - Your Spring Boot service
  - PostgreSQL with pgvector extension
  - Environment variables for API keys and DB config
  - Named volumes for data persistence
  - Health checks

**Embeddings scope:**
- Use Spring AI to create embeddings from document text
- Store embeddings in pgvector
- Build a similarity search endpoint

**Tech Stack:** Spring Boot + Spring AI + pgvector + PostgreSQL + Docker Compose

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/semanticdb
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: semanticdb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

**Python version:** Build the same semantic search in Python using Chroma + Ollama embeddings (all free, all local).

### ✅ By End of Month 5 You Can
- Create and store embeddings using Spring AI
- Run semantic search with pgvector
- Write Dockerfiles and docker-compose files confidently (refreshed)
- Have Project 3 running fully in Docker Compose on GitHub

---

## Month 6: Full RAG Application + RAG Evaluation

**Goal:** Build a complete RAG pipeline. This is the most important month.

### What RAG Looks Like in Code

```
Ingestion Pipeline (runs once):
  → Load documents (PDF, text)
  → Split into chunks
  → Create embedding per chunk
  → Store chunks + embeddings in pgvector

Query Pipeline (runs on every user question):
  → User asks question
  → Embed the question
  → Search pgvector for top-k similar chunks
  → Build prompt: "Using this context: [chunks], answer: [question]"
  → Send to LLM → return answer + source citations
```

### What to Learn
- Document loaders (PDF, plain text)
- Chunking strategies — chunk size and overlap matter more than beginners expect
- Top-k retrieval, similarity threshold, hybrid search
- RAG evaluation with RAGAS — measure before you optimize

### Where to Learn

| Resource | Notes |
|----------|-------|
| [Spring AI RAG docs](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html) | Java implementation |
| [LangChain RAG tutorial](https://python.langchain.com/docs/tutorials/rag/) | Best conceptual walkthrough |
| [DeepLearning.AI - "LangChain: Chat with Your Data"](https://www.deeplearning.ai/short-courses/) | Free, directly teaches RAG |
| [RAGAS docs](https://docs.ragas.io) | Automated RAG evaluation |

---

### 🔨 Project 4: Personal Knowledge Base Chatbot — Full RAG
**GitHub repo: `knowledge-base-rag`**

**What it does:**
- Load a set of documents (your own notes, any PDFs you have, documentation)
- Full RAG pipeline — chunk, embed, store, retrieve, answer
- Answers include source citations (which document the answer came from)
- Evaluation with RAGAS — measure context precision and answer faithfulness

**Tech Stack:** Python + LangChain + Chroma + Ollama (all free) for the Python version — then replicate the retrieval logic in Spring AI + pgvector

> **Don't skip evaluation.** Run RAGAS on at least 10 test questions. Tune your chunk size based on the results. This is what separates engineers from people who just copy tutorial code.

### ✅ By End of Month 6 You Can
- Build a complete RAG pipeline end-to-end
- Measure RAG quality with real metrics
- Explain chunking strategies and retrieval tradeoffs
- Have Project 4 on GitHub

---

## Month 7–8: LangChain4j + Spring Boot RAG API + Microservices Revision

**Goal:** Build a production-grade Spring Boot AI API AND revise microservices architecture simultaneously.

### AI Learning This Month
- LangChain4j AI Services — declarative AI integration for Java
- Conversation memory management
- Multi-tenant RAG (different users, different document collections)
- Streaming responses (token by token output for better UX)

### Java Revision This Month — Microservices
**What you're revising:** Service decomposition, inter-service communication with OpenFeign, API Gateway patterns, service discovery concepts, shared vs isolated databases.

---

### 🔨 Project 5: AI Platform — Microservices Architecture
**GitHub repo: `ai-platform-microservices`**

**What it does:**
A multi-service platform split into focused microservices:

```
┌─────────────────────────────────────────────────────┐
│                   API Gateway                        │
│            (Spring Cloud Gateway)                    │
└──────┬──────────────┬───────────────┬───────────────┘
       │              │               │
  ┌────▼────┐   ┌─────▼─────┐  ┌─────▼──────┐
  │  User   │   │ Document  │  │    Chat    │
  │ Service │   │  Service  │  │  Service   │
  │(Auth/   │   │(Upload,   │  │(RAG, LLM,  │
  │  JWT)   │   │ Embed,    │  │ Memory)    │
  │         │   │ pgvector) │  │            │
  └─────────┘   └───────────┘  └────────────┘
```

**Microservices scope:**
- **User Service:** Registration, login, JWT issuance — Spring Security + JPA
- **Document Service:** Document upload, chunking, embedding, pgvector storage
- **Chat Service:** RAG query pipeline, conversation memory, LLM calls via LangChain4j
- **API Gateway:** Spring Cloud Gateway routing all requests, JWT validation at gateway level
- **OpenFeign:** Chat Service calls Document Service to retrieve relevant chunks
- **Docker Compose:** All 4 services + PostgreSQL + pgvector in one `docker-compose.yml`

**LangChain4j scope:**
```java
// AI Service in Chat Service
interface DocumentAssistant {
    @SystemMessage("You are a document assistant. Answer only from provided context. If unsure, say so.")
    String answer(@MemoryId String sessionId, @UserMessage String question);
}

@Bean
DocumentAssistant documentAssistant(ChatLanguageModel model,
                                     ChatMemoryStore memoryStore,
                                     EmbeddingStoreContentRetriever retriever) {
    return AiServices.builder(DocumentAssistant.class)
        .chatLanguageModel(model)
        .chatMemory(memoryStore)
        .contentRetriever(retriever)
        .build();
}
```

### Where to Learn

| Resource | Link |
|----------|------|
| LangChain4j Docs | [docs.langchain4j.dev](https://docs.langchain4j.dev) |
| Spring Cloud Gateway | [docs.spring.io/spring-cloud-gateway](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/) |
| LangChain4j Examples | [github.com/langchain4j/langchain4j-examples](https://github.com/langchain4j/langchain4j-examples) |

### ✅ By End of Month 7–8 You Can
- Build a multi-service Spring Boot AI platform
- Route and authenticate requests through an API Gateway
- Use OpenFeign for inter-service AI data flow
- Have your biggest project on GitHub — this is a strong portfolio piece

---

## Month 9–10: AI Agents + Tool Calling + MCP

**Goal:** Build LLMs that take actions. Learn MCP properly and build a real MCP server.

### Part 1: Agents and Tool Calling

An agent is an LLM that:
1. Decides which tool to use from a set of available tools
2. Calls that tool (an API, a function, a DB query)
3. Uses the result to decide what to do next
4. Loops until it has a final answer

Your Spring Boot microservices from Month 7–8 become the tools your AI agent calls.

```java
// LangChain4j — your Spring Boot service methods become AI tools
public class DocumentTools {
    @Tool("Search for documents related to a topic")
    public List<String> searchDocuments(@P("The search query") String query) {
        return documentServiceClient.semanticSearch(query); // calls your Document Service via Feign
    }

    @Tool("Get a summary of a specific document by its ID")
    public String getDocumentSummary(@P("The document ID") Long documentId) {
        return documentServiceClient.getSummary(documentId);
    }
}
// The LLM decides when and how to call these — you don't hardcode the logic
```

### Part 2: MCP — Model Context Protocol

MCP is a universal protocol for giving AI agents structured, safe access to external tools and data. Built by Anthropic, now an industry standard adopted by OpenAI, Google, and others.

**What to Learn:**
- MCP client-server architecture
- How to build an MCP server that exposes tools
- How to connect Claude Desktop or any MCP client to your server
- Resources, Tools, and Prompts — the three MCP primitives
- How MCP compares to raw function calling

### Where to Learn

| Resource | Notes |
|----------|-------|
| [MCP Official Docs](https://modelcontextprotocol.io) | Start here — spec, concepts, quickstart |
| [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) | Build MCP servers in Python |
| [MCP Java SDK](https://github.com/modelcontextprotocol/java-sdk) | Build MCP servers in Java |
| [DeepLearning.AI - "AI Agents in LangGraph"](https://www.deeplearning.ai/short-courses/) | Free, teaches agent reasoning loop |
| [Anthropic - Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | Essential reading — written by Anthropic |

---

### 🔨 Project 6: AI Agent with Tool Calling
**GitHub repo: `ai-agent-tools`**

**What it does:**
- An AI agent that has access to multiple tools: web search, your document search (from Project 5), a weather API, a calculator
- The LLM reasons about which tool to call, calls it, uses the result, and continues
- Implement the ReAct pattern (Reason + Act loop)

**Tech Stack:** Spring Boot + LangChain4j tools + free public APIs (OpenWeather, etc.)

---

### 🔨 Project 7: Custom MCP Server
**GitHub repo: `mcp-server`**

**What it does:**
- A Python MCP server exposing 3 tools:
  - `search_documents` — calls your Document Service API
  - `summarize_text` — sends text to an LLM for summarization
  - `get_weather` — calls a weather API
- Connect it to Claude Desktop and test it works interactively
- Optional: Rebuild in Java using the MCP Java SDK

**Why this project:** MCP servers are increasingly in demand in enterprise AI. This is forward-looking portfolio work that few candidates have.

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("my-ai-tools")

@server.tool()
async def search_documents(query: str) -> str:
    """Search for documents related to a query"""
    # call your Spring Boot Document Service
    response = requests.get(f"http://localhost:8081/documents/search?q={query}")
    return response.json()

async def main():
    async with stdio_server() as streams:
        await server.run(*streams, server.create_initialization_options())
```

### ✅ By End of Month 9–10 You Can
- Build AI agents that call real APIs and tools
- Understand the agent reasoning loop (ReAct pattern)
- Build and deploy a working MCP server
- Connect external clients to your MCP server

---

## Month 11–12: Production, Observability, and Capstone

**Goal:** Build something genuinely production-ready. This is what you show in interviews.

### What to Learn
- **Observability:** Trace LLM calls — LangFuse (open source, free self-hosted)
- **Cost management:** Log token usage, add semantic caching
- **Guardrails:** Detect prompt injection, input validation
- **Streaming:** Return LLM output token by token (proper UX)
- **RAG at scale:** Automated quality checks with RAGAS

### Where to Learn

| Resource | Notes |
|----------|-------|
| [LangFuse](https://langfuse.com) | Open-source LLM observability — self-host for free |
| [RAGAS](https://docs.ragas.io) | Automated RAG evaluation |

---

### 🔨 Project 8: Production AI Platform — Capstone
**GitHub repo: `ai-platform-production`**

This is the culmination of everything. Take your Month 7–8 microservices platform and make it production-grade:

**Architecture:**
```
┌──────────────────────────────────────────────────────────────┐
│                        API Gateway                            │
│                  (JWT auth + rate limiting)                    │
└──────┬──────────────┬───────────────┬──────────────┬─────────┘
       │              │               │              │
  ┌────▼────┐   ┌─────▼─────┐  ┌─────▼──────┐ ┌────▼──────┐
  │  User   │   │ Document  │  │    Chat    │ │   MCP     │
  │ Service │   │  Service  │  │  Service   │ │  Server   │
  └─────────┘   └───────────┘  └────────────┘ └───────────┘
                                      │
                               ┌──────▼──────┐
                               │  LangFuse   │
                               │ Observability│
                               └─────────────┘
```

**What it includes:**
1. All microservices from Project 5 — properly Dockerized
2. AI agents with tool calling from Project 6
3. MCP server from Project 7 — exposed as a service
4. LangFuse tracing — every single LLM call logged and traceable
5. RAGAS evaluation suite — automated quality testing
6. Streaming responses in the Chat Service
7. Full `docker-compose.yml` — one command to run everything
8. Clean README with architecture diagram, setup instructions, and demo screenshots

**Tech Stack:** Spring Boot + LangChain4j + Spring AI + pgvector + PostgreSQL + LangFuse + Python MCP Server + Docker Compose

> Write a proper README. Add a screenshot of LangFuse traces. Add a GIF of the chat working. This is what hiring managers open when they visit your GitHub.

### ✅ By End of Month 11–12 You Can
- Deploy a multi-service AI platform with one Docker Compose command
- Trace and debug LLM behavior in production
- Measure and improve RAG quality automatically
- Show an impressive, complete capstone in interviews

---

## Complete Summary Table

| Month | AI Topic | Java/Spring/Docker Topic | Project | GitHub Repo |
|-------|----------|------------------------|---------|-------------|
| 1 | AI Literacy | — | — | — |
| 2 | Python Basics | — | — | — |
| 3 | First LLM App | Spring Boot + JPA revision | Smart Notes API | `smart-notes-api` |
| 4 | Prompt Eng + Context | Spring Security + JWT revision | Secure AI Assistant | `secure-ai-assistant` |
| 5 | Embeddings + Vector DB | Docker + Docker Compose revision | Semantic Search Service | `semantic-search-service` |
| 6 | Full RAG Pipeline | — | Knowledge Base Chatbot | `knowledge-base-rag` |
| 7–8 | LangChain4j | Microservices + OpenFeign + Gateway | AI Platform Microservices | `ai-platform-microservices` |
| 9–10 | Agents + MCP | — | Agent Tools + MCP Server | `ai-agent-tools`, `mcp-server` |
| 11–12 | Production AI | Docker Compose full stack | Production AI Platform | `ai-platform-production` |

---

## Your GitHub Portfolio After 14–16 Months

| # | Repo | What It Demonstrates |
|---|------|---------------------|
| 1 | `smart-notes-api` | Spring Boot + JPA + first LLM integration |
| 2 | `secure-ai-assistant` | JWT security + prompt engineering + context management |
| 3 | `semantic-search-service` | Embeddings + pgvector + Docker Compose |
| 4 | `knowledge-base-rag` | Full RAG pipeline + evaluation |
| 5 | `ai-platform-microservices` | LangChain4j + microservices + API Gateway + OpenFeign |
| 6 | `ai-agent-tools` | AI agents + tool calling + ReAct pattern |
| 7 | `mcp-server` | Custom MCP server (rare skill in 2025–2026) |
| 8 | `ai-platform-production` | Full production platform + observability + Docker |

A recruiter looking at this portfolio sees: Java expertise, AI engineering depth, microservices architecture, Docker, and forward-looking skills like MCP. That combination is genuinely uncommon.

---

## The Three Platforms You'll Use Most

1. **[deeplearning.ai](https://deeplearning.ai)** — Best structured AI education. All short courses are free. Use throughout for concepts.
2. **[docs.langchain4j.dev](https://docs.langchain4j.dev)** — Your primary Java AI framework reference.
3. **[modelcontextprotocol.io](https://modelcontextprotocol.io)** — Official MCP documentation. Required reading by Month 9.

---

## One Hard Rule

> **You do not move to the next month until the current month's project is committed to GitHub.**

No exceptions. A half-built project in your head is worth nothing. A working project on GitHub — even if it's messy — is evidence. Evidence is what gets you hired.
