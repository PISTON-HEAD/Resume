# MCP + RAG Roadmap — AI Document Search Chatbot
### Complete Beginner to Working Product in 5 Weeks
### 2–3 Hours/Day | Theory + Hands-On + Interview Prep

---

## Honest Answer: Can You Do This in 1 Month?

**Build a working AI document chatbot in 1 month?** Yes — if you focus ruthlessly.

**Learn MCP and RAG "deeply" in 1 month?** You'll learn them well enough to build real things and answer interview questions confidently. True depth (production optimization, advanced retrieval strategies, custom MCP ecosystems) takes 2–3 months.

**Here's the deal:** This roadmap is designed for **5 weeks** (giving you a small buffer). You'll build **3 mini projects** that each teach a critical piece, then combine everything into your **final AI Document Search Chatbot**. Every mini project feeds directly into the final product — nothing is wasted.

### What You'll Have After 5 Weeks
1. A working AI Document Search Chatbot (upload PDFs, chat with them)
2. A custom MCP server that exposes your document search as a tool
3. Enough theory to answer RAG and MCP interview questions confidently
4. 4 GitHub repos demonstrating real AI engineering skills

### Prerequisites
- Basic programming ability (you have Java — that's more than enough)
- Python basics (variables, functions, pip — if you don't have this, add 3–4 days at the start)
- A computer that can run Ollama (any modern laptop) OR an OpenAI API key ($5–10 budget)

---

## Architecture of What You're Building

By the end of this roadmap, this is your system:

```
┌──────────────────────────────────────────────────────┐
│                  Your Document Chatbot                │
│                                                      │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────┐ │
│  │ Upload  │───▶│  Ingestion   │───▶│  Vector DB  │ │
│  │  PDFs   │    │  Pipeline    │    │  (Chroma)   │ │
│  │  Docs   │    │  Chunk+Embed │    │             │ │
│  └─────────┘    └──────────────┘    └──────┬──────┘ │
│                                            │        │
│  ┌─────────┐    ┌──────────────┐    ┌──────▼──────┐ │
│  │  User   │───▶│   RAG Query  │───▶│  Retrieval  │ │
│  │  Chat   │    │   Pipeline   │◀───│  Engine     │ │
│  │  UI     │◀───│  LLM+Context │    │             │ │
│  └─────────┘    └──────────────┘    └─────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │         MCP Server (exposes tools)           │    │
│  │  • search_documents  • list_documents        │    │
│  │  • get_document_info • summarize_document    │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## Week 0 (Days 1–3): Foundation — What Are You Actually Building?

**Goal:** Understand every concept at a high level BEFORE writing code. This saves you dozens of hours of confusion later.

### Day 1: Understanding LLMs and Their Limitation

**The Core Problem Your Project Solves:**
LLMs (like ChatGPT, Claude, Llama) are trained on public internet data. They do NOT know the contents of your private documents — your company's policies, your research papers, your notes. If you ask ChatGPT "What does section 4.2 of my contract say?", it has no idea.

**RAG exists to solve this.** Instead of retraining the entire model (expensive, slow), you *retrieve* the relevant parts of your documents and *paste them into the prompt* alongside the user's question. The LLM reads your document content right there in the prompt and answers based on it.

**What to Learn Today:**
- What is an LLM — a text prediction engine trained on massive data
- What are tokens — how LLMs see text (not words, not characters — subword pieces)
- What is a context window — the LLM's "working memory" (typically 4K–128K tokens)
- Why the context window is both the solution AND the constraint for RAG
- What is a prompt — the actual text you send to the LLM (system + user + context)

**Resources:**
| Resource | Time | What You Get |
|----------|------|-------------|
| [Andrej Karpathy - Intro to LLMs (YouTube)](https://www.youtube.com/watch?v=zjkBMFhNj_g) | 1 hour | The single best conceptual overview of LLMs ever made |
| [What is RAG? - IBM Technology (YouTube)](https://www.youtube.com/watch?v=T-D1OfcDW1M) | 10 min | Quick, clear RAG explanation |

**Interview Prep — Be Able to Answer:**
- "What is an LLM and how does it generate text?" → It predicts the next token based on all previous tokens, using patterns learned from training data.
- "Why can't you just ask ChatGPT about your private documents?" → It wasn't trained on them. Its knowledge is frozen at training time. RAG solves this by injecting relevant document content into the prompt at query time.

---

### Day 2: Understanding RAG — The Full Picture

**RAG = Retrieval-Augmented Generation**

It has two phases:

**Phase 1: Ingestion (done once per document)**
```
Document → Split into chunks → Convert each chunk to a vector (embedding) → Store in vector database
```

**Phase 2: Query (done every time a user asks a question)**
```
User question → Convert to vector → Find similar chunks in vector DB → Stuff chunks into prompt → Send to LLM → Return answer
```

**Key Concepts to Understand Today:**

| Concept | What It Is | Why It Matters |
|---------|-----------|---------------|
| **Chunking** | Splitting a document into smaller pieces (200–1000 tokens each) | Whole documents are too large for the context window AND too vague for search |
| **Embeddings** | Converting text into a numerical vector (array of ~768–1536 numbers) | Vectors let you do *semantic* search — finding meaning, not just keywords |
| **Vector Database** | A database optimized for storing and searching vectors by similarity | Regular databases search by exact match; vector DBs find "meaning-similar" content |
| **Cosine Similarity** | A math formula that measures how similar two vectors are (0 = unrelated, 1 = identical) | This is how the retrieval step finds the most relevant chunks |
| **Top-k Retrieval** | Returning the k most similar chunks (usually k=3 to k=5) | You don't dump the entire database into the prompt — just the most relevant pieces |
| **Context Stuffing** | Placing retrieved chunks into the LLM prompt | The actual mechanism that lets the LLM "know" about your documents |

**Resources:**
| Resource | Time | What You Get |
|----------|------|-------------|
| [RAG from Scratch - LangChain (YouTube playlist)](https://www.youtube.com/playlist?list=PLfaIDFEXuae2LXbO1_PKyVJiQ23ZztA0x) | Watch first 3 videos (~45 min) | Best visual explanation of RAG internals |
| [What are Vector Embeddings? - Weaviate](https://weaviate.io/blog/vector-embeddings-explained) | 15 min read | Clear explanation of embeddings |
| [Chunking Strategies - Pinecone](https://www.pinecone.io/learn/chunking-strategies/) | 15 min read | Why chunk size matters enormously |

**Interview Prep — Be Able to Answer:**
- "Explain RAG in simple terms" → Instead of retraining the model on new data, we search for relevant document pieces at query time and include them in the prompt. The LLM reads them and answers based on that context.
- "What are embeddings?" → Numerical representations of text where semantically similar texts have similar numbers. "Dog" and "puppy" would have very similar embeddings, even though the words look different.
- "Why not just put the whole document in the prompt?" → Context windows have limits. Even with large windows, stuffing irrelevant content reduces answer quality and increases cost. Retrieval finds just the relevant parts.
- "What's the difference between keyword search and semantic search?" → Keyword search matches exact words. Semantic search matches meaning. "How do I terminate my subscription?" would match a chunk about "cancellation policy" even though no words overlap.

---

### Day 3: Understanding MCP — The Missing Piece

**MCP = Model Context Protocol**

**The Problem MCP Solves:**
Imagine you've built your document chatbot. Now you want Claude Desktop, or ChatGPT, or any other AI assistant to use it. Without MCP, you'd need to write custom integration code for each client. MCP is a universal standard — build one MCP server, and ANY MCP-compatible client can use your tools.

**Think of it like this:**
- USB is a universal protocol for connecting devices to computers
- MCP is a universal protocol for connecting tools/data to AI models

**MCP Architecture:**
```
┌─────────────┐     MCP Protocol      ┌─────────────┐
│  MCP Client │◄──────────────────────▶│  MCP Server │
│  (Claude,   │   (JSON-RPC over      │  (Your code │
│   Cursor,   │    stdio or HTTP)      │   that does │
│   any IDE)  │                        │   things)   │
└─────────────┘                        └─────────────┘
```

**The Three MCP Primitives:**

| Primitive | What It Is | Example |
|-----------|-----------|---------|
| **Tools** | Functions the AI can call | `search_documents(query)` → returns matching chunks |
| **Resources** | Data the AI can read | `documents://list` → returns all uploaded document names |
| **Prompts** | Pre-written prompt templates | `summarize-document` → a template for summarizing any doc |

**For your project, MCP means:** You build an MCP server that exposes your document search as tools. Then Claude Desktop (or any MCP client) can search and chat with your documents directly.

**Resources:**
| Resource | Time | What You Get |
|----------|------|-------------|
| [MCP Official Docs — Overview](https://modelcontextprotocol.io/introduction) | 30 min | The canonical explanation |
| [MCP for Beginners (YouTube) — search "MCP model context protocol tutorial"](https://www.youtube.com/results?search_query=MCP+model+context+protocol+tutorial+beginners) | 30–45 min | Visual walkthrough |
| [Anthropic — What is MCP?](https://www.anthropic.com/news/model-context-protocol) | 15 min | Why Anthropic built it |
| [MCP Quickstart — For Server Developers](https://modelcontextprotocol.io/quickstart/server) | 20 min | See the actual code |

**Interview Prep — Be Able to Answer:**
- "What is MCP?" → Model Context Protocol — an open standard by Anthropic that lets AI models connect to external tools and data sources through a universal interface. Think of it as a USB standard for AI.
- "How is MCP different from function calling?" → Function calling is model-specific (OpenAI's format differs from Anthropic's). MCP is model-agnostic — one server works with any compatible client. MCP also adds resource management and prompt templates, not just tool calling.
- "What are the three MCP primitives?" → Tools (actions the AI can take), Resources (data the AI can read), and Prompts (reusable prompt templates).
- "Why would you use MCP instead of just building an API?" → An API requires the client to know your specific endpoints. MCP is self-describing — the AI client discovers available tools automatically and knows how to call them. It's standardized across all MCP clients.

---

## Week 1 (Days 4–10): Hands-On Fundamentals — Your First RAG Pipeline

**Goal:** Build a working RAG pipeline from scratch. By the end of this week, you'll have a Python script that answers questions about a PDF.

### Days 4–5: Setup + First LLM Interaction

**Setup Checklist:**
- [ ] Install Python 3.11+ (`python --version` to check)
- [ ] Install Ollama from [ollama.com](https://ollama.com) — this gives you free local LLMs
- [ ] Run `ollama pull llama3.2` — downloads a 3B parameter model (~2GB)
- [ ] Run `ollama pull nomic-embed-text` — downloads an embedding model
- [ ] Create a virtual environment: `python -m venv venv` then `venv\Scripts\activate` (Windows)
- [ ] Install dependencies: `pip install langchain langchain-ollama langchain-community chromadb pypdf`

**Alternatively, if using OpenAI API:**
- [ ] Get an API key from [platform.openai.com](https://platform.openai.com)
- [ ] `pip install langchain langchain-openai chromadb pypdf`

---

### 🔨 Mini Project 1: Simple Q&A Over a Text File
**GitHub repo: `mini-rag-basics`**

**What it does:** Takes a text file, splits it into chunks, stores embeddings in Chroma, and answers questions about it. The absolute simplest possible RAG pipeline.

**Why this project:** Before handling PDFs, web UIs, and MCP servers — nail the core pipeline.

**Build it step by step:**

**Step 1 — Talk to an LLM (Day 4)**

```python
# 01_hello_llm.py
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.2")
response = llm.invoke("What is retrieval-augmented generation?")
print(response.content)
```

Run this. If it works, you can talk to an LLM from Python. That's milestone 1.

**Step 2 — Create Embeddings (Day 4)**

```python
# 02_embeddings.py
from langchain_ollama import OllamaEmbeddings

embeddings = OllamaEmbeddings(model="nomic-embed-text")

# Embed a single sentence
vector = embeddings.embed_query("What is machine learning?")
print(f"Vector dimensions: {len(vector)}")
print(f"First 5 values: {vector[:5]}")

# See how similarity works
import numpy as np

vec1 = embeddings.embed_query("Dogs are great pets")
vec2 = embeddings.embed_query("Puppies make wonderful companions")
vec3 = embeddings.embed_query("The stock market crashed yesterday")

def cosine_sim(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print(f"Dogs vs Puppies similarity: {cosine_sim(vec1, vec2):.4f}")    # High (~0.8+)
print(f"Dogs vs Stocks similarity: {cosine_sim(vec1, vec3):.4f}")     # Low (~0.3)
```

This makes embeddings tangible. You'll SEE that similar meaning = similar numbers.

**Step 3 — Chunk, Embed, Store, Retrieve, Answer (Day 5)**

```python
# 03_simple_rag.py
from langchain_ollama import ChatOllama, OllamaEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate

# 1. Load a text file
with open("sample.txt", "r", encoding="utf-8") as f:
    text = f.read()

# 2. Split into chunks
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_text(text)
print(f"Split into {len(chunks)} chunks")

# 3. Store in Chroma (vector DB)
embeddings = OllamaEmbeddings(model="nomic-embed-text")
vectorstore = Chroma.from_texts(chunks, embeddings, persist_directory="./chroma_db")

# 4. Retrieve relevant chunks
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 5. Build RAG chain
llm = ChatOllama(model="llama3.2")

prompt = ChatPromptTemplate.from_template("""
Answer the question based ONLY on the following context. 
If you cannot answer from the context, say "I don't have enough information to answer that."

Context:
{context}

Question: {question}

Answer:""")

# 6. Query loop
while True:
    question = input("\nAsk a question (or 'quit'): ")
    if question.lower() == 'quit':
        break
    
    # Retrieve
    docs = retriever.invoke(question)
    context = "\n\n".join(docs)
    
    # Generate
    chain = prompt | llm
    response = chain.invoke({"context": context, "question": question})
    
    print(f"\nAnswer: {response.content}")
    print(f"\n--- Sources (chunks used) ---")
    for i, doc in enumerate(docs):
        print(f"Chunk {i+1}: {doc[:100]}...")
```

Create a `sample.txt` with any content (copy-paste a Wikipedia article, your notes, anything). Run it. Ask questions. **You now have a working RAG pipeline.**

**What to Experiment With (Day 5):**
- Change `chunk_size` from 500 to 200, then to 1000. Ask the same questions. See how answers change.
- Change `k` from 3 to 1, then to 5. See the quality tradeoff.
- Add `chunk_overlap=100` and see if cross-chunk answers improve.

**Interview Prep — Be Able to Answer:**
- "Walk me through how a RAG pipeline works, step by step" → Load doc → chunk it → embed each chunk → store in vector DB. On query: embed the question → retrieve top-k similar chunks → inject into prompt → LLM generates answer from that context.
- "What is a chunk overlap and why does it matter?" → When splitting documents, chunks share some text at boundaries. Without overlap, a sentence that spans two chunks gets split and neither chunk has the full context. Overlap ensures continuity.
- "What is Chroma?" → An open-source vector database. It stores embeddings and supports similarity search. Light, local, perfect for development and small-to-medium projects.

---

### Days 6–8: PDF Processing + Improved RAG

**What to Learn:**
- Loading PDFs with PyPDF
- Metadata preservation (which page did this chunk come from?)
- Document objects in LangChain (text + metadata)
- Better retrieval: similarity score thresholds

---

### 🔨 Mini Project 2: PDF Question Answering
**GitHub repo: `pdf-qa-rag`**

**What it does:** Upload one or more PDFs. Ask questions. Get answers with page number citations.

```python
# pdf_rag.py
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_ollama import ChatOllama, OllamaEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.prompts import ChatPromptTemplate
import os

# --- INGESTION ---
def ingest_pdfs(pdf_folder: str, persist_dir: str = "./chroma_db"):
    """Load all PDFs from a folder, chunk them, embed, store."""
    documents = []
    
    for filename in os.listdir(pdf_folder):
        if filename.endswith(".pdf"):
            filepath = os.path.join(pdf_folder, filename)
            loader = PyPDFLoader(filepath)
            docs = loader.load()  # Each page becomes a Document with metadata
            documents.extend(docs)
            print(f"Loaded {filename}: {len(docs)} pages")
    
    # Split into chunks, preserving metadata
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50,
        separators=["\n\n", "\n", ". ", " ", ""]
    )
    chunks = splitter.split_documents(documents)
    print(f"Total chunks: {len(chunks)}")
    
    # Store in Chroma
    embeddings = OllamaEmbeddings(model="nomic-embed-text")
    vectorstore = Chroma.from_documents(
        chunks, embeddings, persist_directory=persist_dir
    )
    return vectorstore


# --- QUERY ---
def query_documents(vectorstore, question: str):
    """Retrieve relevant chunks and generate an answer."""
    llm = ChatOllama(model="llama3.2")
    
    # Retrieve with scores
    results = vectorstore.similarity_search_with_score(question, k=4)
    
    # Build context with source info
    context_parts = []
    sources = []
    for doc, score in results:
        source = doc.metadata.get("source", "unknown")
        page = doc.metadata.get("page", "?")
        context_parts.append(f"[Source: {os.path.basename(source)}, Page {page + 1}]\n{doc.page_content}")
        sources.append(f"{os.path.basename(source)}, Page {page + 1} (score: {score:.3f})")
    
    context = "\n\n---\n\n".join(context_parts)
    
    prompt = ChatPromptTemplate.from_template("""
You are a helpful document assistant. Answer the question based ONLY on the provided context.
Always cite which document and page your answer comes from.
If the context doesn't contain enough information, say so clearly.

Context:
{context}

Question: {question}

Answer (with citations):""")
    
    chain = prompt | llm
    response = chain.invoke({"context": context, "question": question})
    
    return response.content, sources


# --- MAIN ---
if __name__ == "__main__":
    pdf_folder = "./documents"  # Put your PDFs here
    
    # Check if already ingested
    if os.path.exists("./chroma_db"):
        print("Loading existing vector store...")
        embeddings = OllamaEmbeddings(model="nomic-embed-text")
        vectorstore = Chroma(persist_directory="./chroma_db", embedding_function=embeddings)
    else:
        print("Ingesting PDFs...")
        vectorstore = ingest_pdfs(pdf_folder)
    
    # Chat loop
    print("\n📚 PDF Q&A Ready! Ask questions about your documents.\n")
    while True:
        question = input("You: ")
        if question.lower() in ('quit', 'exit'):
            break
        
        answer, sources = query_documents(vectorstore, question)
        print(f"\nAssistant: {answer}")
        print(f"\nSources used:")
        for s in sources:
            print(f"  • {s}")
        print()
```

**What to test:**
1. Put 2–3 different PDFs in the `documents/` folder
2. Ask questions that span multiple documents
3. Ask questions that DON'T have answers in the documents — verify it says "I don't know" instead of hallucinating

**Days 7–8: Improve It**
- Add a `--reingest` flag to rebuild the vector store when new documents are added
- Experiment with chunk sizes (try 300, 500, 800) on the same questions — record which gives better answers
- Add conversation history (keep last 3 exchanges in the prompt)

```python
# Add conversation memory
conversation_history = []

def query_with_memory(vectorstore, question: str, history: list):
    # Include recent conversation in prompt for follow-up questions
    history_text = ""
    if history:
        history_text = "Previous conversation:\n"
        for h in history[-3:]:  # Last 3 exchanges
            history_text += f"Human: {h['question']}\nAssistant: {h['answer']}\n\n"
    
    # ... rest of retrieval logic
    # Add history_text to the prompt template
```

**Interview Prep — Be Able to Answer:**
- "How do you handle PDFs in a RAG pipeline?" → Use a PDF parser (PyPDF, pdfplumber) to extract text page by page. Each page becomes a document with metadata (filename, page number). Then chunk, embed, and store like any text. Metadata is preserved so we can cite sources.
- "What happens if a user asks a follow-up question?" → Without conversation memory, each question is independent. With memory, you include recent conversation history in the prompt so the LLM understands context like "What about the second one?" referring to something previously discussed.
- "How do you evaluate if your RAG is working well?" → Check: 1) Are the right chunks being retrieved? (retrieval quality) 2) Is the LLM answering faithfully from those chunks? (faithfulness) 3) Is the answer actually correct? (correctness). You can use frameworks like RAGAS for automated evaluation.

---

### Days 9–10: Theory Deep Dive + Consolidation

**Day 9 — RAG Theory for Interviews**

Study these topics (read articles, watch videos):

**Chunking Strategies:**
| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| Fixed-size | Split every N characters/tokens | Simple, works for most cases |
| Recursive | Split by paragraphs, then sentences, then words | Best general-purpose strategy |
| Semantic | Group sentences by meaning similarity | Best quality, more complex |
| Document-based | Split by headings, sections | Structured documents (HTML, Markdown) |

**Retrieval Strategies:**
| Strategy | What It Does |
|----------|-------------|
| Naive (top-k) | Return k most similar chunks |
| With threshold | Return chunks above a similarity score |
| Hybrid search | Combine vector search + keyword search (BM25) |
| Re-ranking | Retrieve many, then re-rank with a cross-encoder |
| Multi-query | Rephrase the question multiple ways, retrieve for each |
| Parent document | Store small chunks for retrieval, but return the larger parent chunk |

**Common RAG Failures and Fixes:**

| Problem | Symptom | Fix |
|---------|---------|-----|
| Chunks too large | Answers are vague, retrieves irrelevant content | Reduce chunk size |
| Chunks too small | Misses context needed for good answers | Increase chunk size or overlap |
| Wrong chunks retrieved | Answer doesn't match the question | Try hybrid search, re-ranking, or multi-query |
| LLM hallucinates | Makes up information not in chunks | Strengthen system prompt: "ONLY use provided context" |
| Cross-chunk information | Answer requires info split across chunks | Increase chunk overlap, or use parent document retriever |

**Resources for Deep Theory:**
| Resource | Time |
|----------|------|
| [DeepLearning.AI — "LangChain: Chat with Your Data"](https://www.deeplearning.ai/short-courses/langchain-chat-with-your-data/) | Free, ~2 hours, the best structured RAG course |
| [DeepLearning.AI — "Building and Evaluating Advanced RAG"](https://www.deeplearning.ai/short-courses/building-evaluating-advanced-rag/) | Free, ~1.5 hours, covers advanced retrieval |

**Day 10 — MCP Theory for Interviews**

**MCP Architecture Deep Dive:**

```
                    MCP Protocol (JSON-RPC 2.0)
                    
┌─────────────┐                          ┌─────────────┐
│  MCP Host   │    spawns & manages      │  MCP Client  │
│  (Claude    │───────────────────────▶  │  (1 per      │
│   Desktop,  │                          │   server)    │
│   IDE)      │                          └──────┬───────┘
└─────────────┘                                 │
                                                │ JSON-RPC
                                                │ (stdio or SSE)
                                         ┌──────▼───────┐
                                         │  MCP Server  │
                                         │  (Your code) │
                                         │              │
                                         │  Tools       │
                                         │  Resources   │
                                         │  Prompts     │
                                         └──────────────┘
```

**Key MCP Concepts:**

| Concept | Explanation |
|---------|------------|
| **Host** | The application the user interacts with (Claude Desktop, VS Code, etc.) |
| **Client** | Created by the host, maintains 1:1 connection with a server |
| **Server** | Your code — exposes tools, resources, and prompts |
| **Transport** | How client and server communicate: `stdio` (local) or `SSE/HTTP` (remote) |
| **Tool Discovery** | Client asks server "what tools do you have?" — server responds with list + schemas |
| **Tool Invocation** | Client sends tool name + arguments → server executes → returns result |

**MCP vs Function Calling vs API:**

| Feature | Raw API | Function Calling | MCP |
|---------|---------|-----------------|-----|
| Discovery | Manual (read docs) | Model-specific schema | Automatic (server declares capabilities) |
| Standardization | None | Provider-specific | Universal standard |
| Multi-tool | Custom orchestration | Model decides | Model decides |
| Data access | Custom endpoints | Not included | Resources primitive |
| Client compatibility | Custom per client | Per LLM provider | Any MCP client |

**Interview Prep — Be Able to Answer:**
- "What problem does MCP solve?" → Before MCP, every AI integration was custom. If you wanted Claude AND ChatGPT to use your tools, you'd write two different integrations. MCP is a standard protocol — build once, works everywhere.
- "How does an MCP server communicate with a client?" → Through JSON-RPC 2.0 messages over either stdio (local, server runs as subprocess) or HTTP with Server-Sent Events (remote). The client discovers available tools, then invokes them by name with typed arguments.
- "Can you use MCP with any LLM?" → MCP is LLM-agnostic. The MCP client sits between the LLM and the server. The LLM sees tools as function-calling schemas. The client translates between the LLM's native function-calling format and MCP's protocol.

---

## Week 2 (Days 11–17): Build the Document Chatbot Core

**Goal:** Build the full document chatbot with a web UI. This is the main project.

### Days 11–12: Project Setup + Document Ingestion

### 🔨 Main Project: AI Document Search Chatbot
**GitHub repo: `ai-document-chatbot`**

**Tech Stack:**
- **Backend:** Python + FastAPI
- **RAG:** LangChain + Chroma + Ollama (or OpenAI)
- **PDF Processing:** PyPDF, python-docx (for Word docs), unstructured (optional)
- **Frontend:** Streamlit (simplest option) or HTML + JavaScript
- **MCP:** Python MCP SDK (added in Week 3)

**Project Structure:**
```
ai-document-chatbot/
├── backend/
│   ├── main.py              # FastAPI app
│   ├── ingestion.py         # Document loading + chunking + embedding
│   ├── retrieval.py         # RAG query pipeline
│   ├── models.py            # Pydantic models
│   └── config.py            # Settings
├── documents/               # Uploaded documents stored here
├── chroma_db/               # Vector database storage
├── frontend/
│   └── app.py               # Streamlit chat UI
├── mcp_server/
│   └── server.py            # MCP server (Week 3)
├── tests/
│   ├── test_ingestion.py
│   └── test_retrieval.py
├── requirements.txt
├── docker-compose.yml       # (Week 4)
└── README.md
```

**Day 11 — Build the Ingestion Pipeline:**

```python
# backend/ingestion.py
import os
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_ollama import OllamaEmbeddings
from langchain_community.vectorstores import Chroma

CHROMA_DIR = "./chroma_db"
DOCUMENTS_DIR = "./documents"

def get_vectorstore():
    """Get or create the Chroma vector store."""
    embeddings = OllamaEmbeddings(model="nomic-embed-text")
    return Chroma(persist_directory=CHROMA_DIR, embedding_function=embeddings)

def load_document(filepath: str):
    """Load a single document based on file extension."""
    ext = os.path.splitext(filepath)[1].lower()
    
    if ext == ".pdf":
        loader = PyPDFLoader(filepath)
    elif ext == ".txt":
        loader = TextLoader(filepath, encoding="utf-8")
    else:
        raise ValueError(f"Unsupported file type: {ext}")
    
    return loader.load()

def ingest_document(filepath: str) -> dict:
    """Full ingestion pipeline for a single document."""
    # Load
    documents = load_document(filepath)
    
    # Chunk
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50,
        separators=["\n\n", "\n", ". ", " ", ""]
    )
    chunks = splitter.split_documents(documents)
    
    # Add custom metadata
    filename = os.path.basename(filepath)
    for chunk in chunks:
        chunk.metadata["filename"] = filename
    
    # Embed and store
    vectorstore = get_vectorstore()
    vectorstore.add_documents(chunks)
    
    return {
        "filename": filename,
        "pages": len(documents),
        "chunks": len(chunks)
    }
```

**Day 12 — Build the FastAPI Backend:**

```python
# backend/main.py
from fastapi import FastAPI, UploadFile, File, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import shutil
import os
from ingestion import ingest_document, DOCUMENTS_DIR
from retrieval import query_documents

app = FastAPI(title="AI Document Chatbot")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Tighten in production
    allow_methods=["*"],
    allow_headers=["*"],
)

class ChatRequest(BaseModel):
    question: str

class ChatResponse(BaseModel):
    answer: str
    sources: list[dict]

@app.post("/upload")
async def upload_document(file: UploadFile = File(...)):
    """Upload and ingest a document."""
    allowed_extensions = {".pdf", ".txt"}
    ext = os.path.splitext(file.filename)[1].lower()
    if ext not in allowed_extensions:
        raise HTTPException(400, f"Unsupported file type. Allowed: {allowed_extensions}")
    
    # Save file
    os.makedirs(DOCUMENTS_DIR, exist_ok=True)
    filepath = os.path.join(DOCUMENTS_DIR, file.filename)
    with open(filepath, "wb") as f:
        shutil.copyfileobj(file.file, f)
    
    # Ingest
    result = ingest_document(filepath)
    return {"message": "Document ingested successfully", **result}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Ask a question about uploaded documents."""
    answer, sources = query_documents(request.question)
    return ChatResponse(answer=answer, sources=sources)

@app.get("/documents")
async def list_documents():
    """List all uploaded documents."""
    if not os.path.exists(DOCUMENTS_DIR):
        return {"documents": []}
    files = os.listdir(DOCUMENTS_DIR)
    return {"documents": files}
```

---

### Days 13–14: RAG Query Engine + Chat UI

**Day 13 — Build the Retrieval/Query Engine:**

```python
# backend/retrieval.py
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate
from ingestion import get_vectorstore
import os

def query_documents(question: str, k: int = 4) -> tuple[str, list[dict]]:
    """RAG query: retrieve relevant chunks and generate an answer."""
    vectorstore = get_vectorstore()
    
    # Retrieve
    results = vectorstore.similarity_search_with_score(question, k=k)
    
    if not results:
        return "No documents have been uploaded yet. Please upload some documents first.", []
    
    # Build context
    context_parts = []
    sources = []
    for doc, score in results:
        filename = doc.metadata.get("filename", "unknown")
        page = doc.metadata.get("page", "N/A")
        context_parts.append(
            f"[Source: {filename}, Page: {page + 1 if isinstance(page, int) else page}]\n{doc.page_content}"
        )
        sources.append({
            "filename": filename,
            "page": page + 1 if isinstance(page, int) else page,
            "score": round(float(score), 4),
            "preview": doc.page_content[:150] + "..."
        })
    
    context = "\n\n---\n\n".join(context_parts)
    
    # Generate
    llm = ChatOllama(model="llama3.2")
    
    prompt = ChatPromptTemplate.from_template("""You are a helpful document assistant. Your job is to answer questions based ONLY on the provided document context.

Rules:
1. Only use information from the provided context
2. Always cite which document and page your information comes from
3. If the context doesn't contain enough information to answer, say "I couldn't find information about that in the uploaded documents."
4. Be concise but thorough

Context from documents:
{context}

User's question: {question}

Answer:""")
    
    chain = prompt | llm
    response = chain.invoke({"context": context, "question": question})
    
    return response.content, sources
```

**Day 14 — Build the Chat UI with Streamlit:**

```python
# frontend/app.py
import streamlit as st
import requests

API_URL = "http://localhost:8000"

st.set_page_config(page_title="📚 Document Chatbot", layout="wide")
st.title("📚 AI Document Search Chatbot")

# Sidebar — Document Upload
with st.sidebar:
    st.header("📄 Upload Documents")
    uploaded_file = st.file_uploader(
        "Upload a PDF or Text file",
        type=["pdf", "txt"],
        accept_multiple_files=False
    )
    
    if uploaded_file and st.button("Upload & Process"):
        with st.spinner("Processing document..."):
            files = {"file": (uploaded_file.name, uploaded_file, uploaded_file.type)}
            response = requests.post(f"{API_URL}/upload", files=files)
            if response.status_code == 200:
                result = response.json()
                st.success(f"✅ Uploaded: {result['filename']} ({result['chunks']} chunks)")
            else:
                st.error(f"Error: {response.text}")
    
    st.divider()
    st.header("📁 Uploaded Documents")
    doc_response = requests.get(f"{API_URL}/documents")
    if doc_response.status_code == 200:
        docs = doc_response.json()["documents"]
        if docs:
            for doc in docs:
                st.write(f"• {doc}")
        else:
            st.write("No documents uploaded yet.")

# Main area — Chat
if "messages" not in st.session_state:
    st.session_state.messages = []

# Display chat history
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.write(message["content"])
        if "sources" in message:
            with st.expander("📎 Sources"):
                for s in message["sources"]:
                    st.write(f"• **{s['filename']}** (Page {s['page']}) — relevance: {s['score']}")
                    st.caption(s['preview'])

# Chat input
if question := st.chat_input("Ask a question about your documents..."):
    # Show user message
    st.session_state.messages.append({"role": "user", "content": question})
    with st.chat_message("user"):
        st.write(question)
    
    # Get answer
    with st.chat_message("assistant"):
        with st.spinner("Searching documents..."):
            response = requests.post(f"{API_URL}/chat", json={"question": question})
            if response.status_code == 200:
                data = response.json()
                st.write(data["answer"])
                
                with st.expander("📎 Sources"):
                    for s in data["sources"]:
                        st.write(f"• **{s['filename']}** (Page {s['page']}) — relevance: {s['score']}")
                        st.caption(s['preview'])
                
                st.session_state.messages.append({
                    "role": "assistant",
                    "content": data["answer"],
                    "sources": data["sources"]
                })
            else:
                st.error("Error getting response")
```

**How to Run:**
```bash
# Terminal 1 — Backend
cd backend
uvicorn main:app --reload --port 8000

# Terminal 2 — Frontend
cd frontend
streamlit run app.py
```

---

### Days 15–17: Improvements + More File Types + Conversation Memory

**Day 15 — Add Word Document Support + Better Chunking:**

```bash
pip install python-docx
```

Add to `ingestion.py`:
```python
from langchain_community.document_loaders import Docx2txtLoader

# In load_document():
elif ext in (".docx", ".doc"):
    loader = Docx2txtLoader(filepath)
```

Update the upload endpoint to allow `.docx` files.

**Day 16 — Add Conversation Memory:**

The chatbot should handle follow-up questions like "Tell me more about that" or "What was the second point?"

```python
# backend/retrieval.py — enhanced with conversation history

def query_documents_with_history(
    question: str, 
    conversation_history: list[dict],
    k: int = 4
) -> tuple[str, list[dict]]:
    """RAG query with conversation context for follow-ups."""
    vectorstore = get_vectorstore()
    
    # If this looks like a follow-up, rewrite the question using history
    llm = ChatOllama(model="llama3.2")
    
    if conversation_history:
        # Rewrite the question to be standalone
        rewrite_prompt = ChatPromptTemplate.from_template("""
Given the conversation history and a follow-up question, rewrite the question to be a standalone question.
If the question is already standalone, return it as-is.

Conversation history:
{history}

Follow-up question: {question}

Standalone question:""")
        
        history_text = "\n".join(
            f"{msg['role'].title()}: {msg['content']}" 
            for msg in conversation_history[-4:]  # Last 2 exchanges
        )
        
        chain = rewrite_prompt | llm
        rewritten = chain.invoke({"history": history_text, "question": question})
        search_query = rewritten.content
    else:
        search_query = question
    
    # Retrieve using the (possibly rewritten) query
    results = vectorstore.similarity_search_with_score(search_query, k=k)
    # ... rest of the pipeline same as before
```

**Day 17 — Testing + Polish:**
- Test with at least 5 different documents
- Test follow-up questions
- Test "I don't know" responses for out-of-scope questions
- Add error handling for empty vector store
- Write a proper README.md

---

## Week 3 (Days 18–24): MCP Server + Advanced RAG

**Goal:** Build an MCP server that exposes your document chatbot as tools. Learn advanced RAG techniques.

### Days 18–20: Build Your MCP Server

**Setup:**
```bash
pip install mcp
```

### 🔨 Mini Project 3: MCP Server for Document Search
**GitHub repo: `document-mcp-server`** (or add to `ai-document-chatbot` as a module)

**What it does:** An MCP server that Claude Desktop (or any MCP client) can connect to, giving the AI access to your document search tools.

```python
# mcp_server/server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, Resource
import json
import sys
import os

# Add parent directory to path so we can import backend modules
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'backend'))
from ingestion import get_vectorstore, ingest_document, DOCUMENTS_DIR
from retrieval import query_documents

server = Server("document-search")

# --- TOOLS ---

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="search_documents",
            description="Search through uploaded documents using semantic search. Returns relevant passages with source citations.",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The search query or question to find relevant document passages"
                    },
                    "num_results": {
                        "type": "integer",
                        "description": "Number of results to return (default: 4)",
                        "default": 4
                    }
                },
                "required": ["query"]
            }
        ),
        Tool(
            name="ask_documents",
            description="Ask a question and get an AI-generated answer based on the uploaded documents. Uses RAG to find relevant context and generate a cited answer.",
            inputSchema={
                "type": "object",
                "properties": {
                    "question": {
                        "type": "string",
                        "description": "The question to answer based on document contents"
                    }
                },
                "required": ["question"]
            }
        ),
        Tool(
            name="list_documents",
            description="List all documents that have been uploaded to the system.",
            inputSchema={
                "type": "object",
                "properties": {}
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "search_documents":
        query = arguments["query"]
        k = arguments.get("num_results", 4)
        
        vectorstore = get_vectorstore()
        results = vectorstore.similarity_search_with_score(query, k=k)
        
        output = []
        for doc, score in results:
            output.append({
                "content": doc.page_content,
                "filename": doc.metadata.get("filename", "unknown"),
                "page": doc.metadata.get("page", "N/A"),
                "relevance_score": round(float(score), 4)
            })
        
        return [TextContent(type="text", text=json.dumps(output, indent=2))]
    
    elif name == "ask_documents":
        question = arguments["question"]
        answer, sources = query_documents(question)
        
        result = {
            "answer": answer,
            "sources": sources
        }
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    
    elif name == "list_documents":
        if not os.path.exists(DOCUMENTS_DIR):
            return [TextContent(type="text", text="No documents uploaded yet.")]
        
        files = os.listdir(DOCUMENTS_DIR)
        return [TextContent(type="text", text=json.dumps({"documents": files}, indent=2))]
    
    else:
        return [TextContent(type="text", text=f"Unknown tool: {name}")]


# --- RESOURCES ---

@server.list_resources()
async def list_resources():
    resources = []
    if os.path.exists(DOCUMENTS_DIR):
        for filename in os.listdir(DOCUMENTS_DIR):
            resources.append(
                Resource(
                    uri=f"document:///{filename}",
                    name=filename,
                    description=f"Uploaded document: {filename}",
                    mimeType="application/pdf" if filename.endswith(".pdf") else "text/plain"
                )
            )
    return resources


# --- MAIN ---

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**Connect to Claude Desktop (Day 19):**

Edit Claude Desktop's config file:
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Mac: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "document-search": {
      "command": "python",
      "args": ["C:\\path\\to\\your\\mcp_server\\server.py"]
    }
  }
}
```

Restart Claude Desktop. You should see your tools available. Now you can say to Claude: "Search my documents for information about X" and it will call your MCP server.

**Day 20 — Test Thoroughly + Add Error Handling:**
- Test all three tools from Claude Desktop
- Test with multiple documents
- Add proper error handling (what if no docs uploaded? what if Ollama isn't running?)
- Add logging so you can see what's happening

---

### Days 21–24: Advanced RAG Techniques

**Day 21 — Hybrid Search (Vector + Keyword):**

Pure vector search sometimes misses exact matches. "What is the value of parameter X123?" might not find a chunk containing "X123" if the embedding doesn't capture that specific code. Hybrid search combines vector similarity with keyword matching.

```bash
pip install rank-bm25
```

```python
# backend/retrieval_advanced.py
from rank_bm25 import BM25Okapi
from langchain_community.vectorstores import Chroma
from ingestion import get_vectorstore

def hybrid_search(question: str, k: int = 4):
    """Combine vector search and keyword search."""
    vectorstore = get_vectorstore()
    
    # Vector search
    vector_results = vectorstore.similarity_search_with_score(question, k=k*2)
    
    # Keyword search (BM25)
    all_docs = vectorstore.get()  # Get all documents
    if not all_docs["documents"]:
        return []
    
    tokenized_corpus = [doc.lower().split() for doc in all_docs["documents"]]
    bm25 = BM25Okapi(tokenized_corpus)
    bm25_scores = bm25.get_scores(question.lower().split())
    
    # Combine scores (reciprocal rank fusion)
    combined = {}
    for rank, (doc, score) in enumerate(vector_results):
        doc_id = doc.page_content[:100]  # Use content prefix as ID
        combined[doc_id] = {"doc": doc, "rrf_score": 1 / (rank + 60)}
    
    top_keyword_indices = bm25_scores.argsort()[-k*2:][::-1]
    for rank, idx in enumerate(top_keyword_indices):
        doc_id = all_docs["documents"][idx][:100]
        if doc_id in combined:
            combined[doc_id]["rrf_score"] += 1 / (rank + 60)
        else:
            # Create a Document-like object
            combined[doc_id] = {
                "doc": type('Doc', (), {
                    "page_content": all_docs["documents"][idx],
                    "metadata": all_docs["metadatas"][idx] if all_docs["metadatas"] else {}
                })(),
                "rrf_score": 1 / (rank + 60)
            }
    
    # Sort by combined score, return top-k
    sorted_results = sorted(combined.values(), key=lambda x: x["rrf_score"], reverse=True)
    return [(item["doc"], item["rrf_score"]) for item in sorted_results[:k]]
```

**Day 22 — Multi-Query Retrieval:**

Users often ask vague questions. Multi-query generates multiple phrasings of the same question and retrieves for each, casting a wider net.

```python
def multi_query_retrieve(question: str, k: int = 4):
    """Generate multiple query variations and retrieve for each."""
    llm = ChatOllama(model="llama3.2")
    
    # Generate alternative questions
    prompt = ChatPromptTemplate.from_template("""
You are an AI assistant helping to improve document search. 
Given a user question, generate 3 alternative phrasings that might find different relevant passages.
Return ONLY the 3 questions, one per line. No numbering, no extra text.

Original question: {question}

Alternative questions:""")
    
    chain = prompt | llm
    response = chain.invoke({"question": question})
    alt_questions = [q.strip() for q in response.content.strip().split("\n") if q.strip()]
    
    all_questions = [question] + alt_questions[:3]
    
    # Retrieve for each question
    vectorstore = get_vectorstore()
    all_results = {}
    
    for q in all_questions:
        results = vectorstore.similarity_search_with_score(q, k=k)
        for doc, score in results:
            key = doc.page_content[:100]
            if key not in all_results or score < all_results[key][1]:
                all_results[key] = (doc, score)
    
    # Return top-k unique results
    sorted_results = sorted(all_results.values(), key=lambda x: x[1])
    return sorted_results[:k]
```

**Day 23 — RAG Evaluation (RAGAS):**

```bash
pip install ragas
```

```python
# tests/evaluate_rag.py
"""
Create a test set of questions with known answers from your documents.
Run RAGAS to measure your RAG pipeline's quality.
"""
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from datasets import Dataset

# Define test cases — questions where you KNOW the answer from your documents
test_questions = [
    "What is the company's vacation policy?",
    "How do I submit an expense report?",
    "What are the working hours?",
]

test_ground_truths = [
    "Employees get 20 days of paid vacation per year.",
    "Submit expense reports through the HR portal within 30 days.",
    "Working hours are 9 AM to 5 PM, Monday to Friday.",
]

# Run your RAG pipeline on each question
answers = []
contexts = []

for question in test_questions:
    answer, sources = query_documents(question)
    answers.append(answer)
    contexts.append([s["preview"] for s in sources])

# Create dataset
data = {
    "question": test_questions,
    "answer": answers,
    "contexts": contexts,
    "ground_truth": test_ground_truths,
}

dataset = Dataset.from_dict(data)

# Evaluate
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
)

print(results)
# This gives you numerical scores telling you:
# - faithfulness: Does the answer stick to the retrieved context? (no hallucination)
# - answer_relevancy: Is the answer actually relevant to the question?
# - context_precision: Are the retrieved chunks actually relevant?
```

**Day 24 — Apply Improvements to Your Chatbot:**
- Integrate hybrid search or multi-query into the main chatbot
- Run RAGAS evaluation, record baseline scores
- Tune chunk_size based on evaluation results
- Update the MCP server to use the improved retrieval

**Interview Prep — Advanced RAG Questions:**
- "How would you improve a RAG pipeline that gives poor answers?" → Diagnose first: check if retrieval is finding the right chunks (context precision). If not, try hybrid search, multi-query, or re-ranking. If retrieval is good but answers are bad, improve the prompt or use a better LLM. Always measure with RAGAS before and after changes.
- "What is hybrid search?" → Combining semantic vector search with traditional keyword search (BM25). Vector search finds meaning-similar content, keyword search finds exact term matches. Reciprocal Rank Fusion merges the two result lists.
- "How do you handle documents that are updated frequently?" → Track document versions. On update, delete old chunks from the vector store and re-ingest. Use metadata to identify which chunks belong to which document version.

---

## Week 4 (Days 25–31): Polish, Dockerize, and Advanced Features

**Goal:** Make the project production-presentable. Add Docker, improve the UI, and handle edge cases.

### Days 25–26: Docker Compose

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create directories
RUN mkdir -p documents chroma_db

EXPOSE 8000

CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  backend:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./documents:/app/documents
      - ./chroma_db:/app/chroma_db
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    # Ollama runs on your host machine, accessible via host.docker.internal

  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    ports:
      - "8501:8501"
    environment:
      - API_URL=http://backend:8000
    depends_on:
      - backend
```

### Days 27–28: Advanced Features

**Add these to differentiate your project:**

1. **Document deletion** — Remove a document and its chunks from the vector store
2. **Search history** — Log all queries and answers
3. **Streaming responses** — Show the LLM answer token-by-token (much better UX)
4. **Multiple file upload** — Upload several files at once
5. **Document preview** — Show which part of which document was used

**Streaming example:**
```python
# backend/retrieval.py — streaming version
from langchain.callbacks import StreamingStdOutCallbackHandler

async def query_documents_stream(question: str):
    """Stream the RAG response token by token."""
    vectorstore = get_vectorstore()
    results = vectorstore.similarity_search_with_score(question, k=4)
    
    context = "\n\n".join([doc.page_content for doc, _ in results])
    
    llm = ChatOllama(model="llama3.2", streaming=True)
    
    prompt = ChatPromptTemplate.from_template("""...""")  # Same as before
    
    chain = prompt | llm
    
    async for chunk in chain.astream({"context": context, "question": question}):
        yield chunk.content
```

### Days 29–30: Final Polish

- [ ] Write a comprehensive README with:
  - Project description and screenshot/GIF
  - Architecture diagram
  - Setup instructions (both local and Docker)
  - How to connect the MCP server to Claude Desktop
  - Tech stack explanation
- [ ] Add proper error messages throughout
- [ ] Test with at least 10 different documents
- [ ] Run RAGAS evaluation and include results in README
- [ ] Push everything to GitHub with clean commit history

### Day 31: Review + Interview Prep Sprint

**Full Interview Question Bank:**

**RAG Questions:**

| Question | Key Points for Your Answer |
|----------|-----------------------------|
| What is RAG? | Retrieval-Augmented Generation. Retrieve relevant context from external sources and inject it into the LLM prompt. Solves the "LLMs don't know your private data" problem without retraining. |
| RAG vs Fine-tuning? | RAG: cheap, real-time data, no training. Fine-tuning: expensive, changes model behavior, data is baked in. Use RAG for knowledge, fine-tuning for behavior/style. Most enterprise use cases need RAG. |
| What are embeddings? | Dense vector representations of text. Similar meanings = similar vectors. Created by embedding models (OpenAI ada-002, nomic-embed-text, etc.). Stored in vector databases. |
| How does chunking affect RAG? | Too large: retrieves irrelevant noise. Too small: loses context. Overlap prevents information loss at boundaries. Typical sizes: 200–1000 tokens with 10–20% overlap. |
| What is a vector database? | Database optimized for similarity search on high-dimensional vectors. Examples: Chroma (lightweight), Pinecone (managed), pgvector (PostgreSQL extension), Weaviate (open-source). |
| What are common RAG failures? | 1) Wrong chunks retrieved → hybrid search/re-ranking. 2) LLM hallucinates → stronger system prompt. 3) Chunk boundary issues → overlapping chunks. 4) Ambiguous queries → multi-query retrieval. |
| How do you evaluate RAG? | Use RAGAS: faithfulness (no hallucination), answer relevancy, context precision (right chunks found), context recall (all relevant chunks found). Set up automated test suites. |
| What is hybrid search? | Combines vector similarity search with keyword search (BM25). Catches both semantic matches and exact term matches. Merged via reciprocal rank fusion. |

**MCP Questions:**

| Question | Key Points for Your Answer |
|----------|---------------------------|
| What is MCP? | Model Context Protocol — open standard for connecting AI models to external tools and data. Created by Anthropic, adopted by OpenAI, Google, etc. Standardizes AI-tool integration. |
| MCP architecture? | Host (app like Claude Desktop) → Client (manages connection) → Server (your code). Communication via JSON-RPC over stdio or HTTP/SSE transport. |
| Three MCP primitives? | Tools (actions AI can call), Resources (data AI can read), Prompts (reusable prompt templates). |
| MCP vs function calling? | Function calling is provider-specific. MCP is a universal standard. Build one MCP server → works with any MCP client. Also includes resources and prompt templates. |
| How do you build an MCP server? | Use the MCP SDK (Python or TypeScript). Define tools with schemas, implement handlers, run over stdio or HTTP. Connect via client configuration (e.g., Claude Desktop config JSON). |

**General AI Engineering:**

| Question | Key Points |
|----------|-----------|
| How would you build a document Q&A system? | Describe YOUR project. Ingestion pipeline (load → chunk → embed → store), query pipeline (embed question → retrieve → prompt + context → LLM answer). Mention evaluation, hybrid search, streaming. |
| How do you handle large documents? | Chunking with overlap. For very large docs, hierarchical chunking (paragraphs + sections). Metadata tracking for source citations. |
| How do you prevent hallucination? | Strong system prompts ("only answer from context"), faithfulness evaluation, confidence scoring, showing source citations so users can verify. |
| What's the cost of running RAG? | Embedding cost (one-time per document), LLM cost per query (tokens in prompt + response). Local models (Ollama) = free but slower. Use caching for repeated queries. |

---

## Week 5 (Days 32–35): Stretch Goals + Portfolio

**If you still have time and energy, add these:**

### Option A: Add a Spring Boot Version
Since you know Java, rebuild the backend in Spring Boot using Spring AI:
```java
// Spring AI RAG — the same pipeline in Java
@RestController
public class ChatController {
    
    private final VectorStore vectorStore;
    private final ChatModel chatModel;
    
    @PostMapping("/chat")
    public ChatResponse chat(@RequestBody ChatRequest request) {
        // Retrieve relevant documents
        List<Document> similar = vectorStore.similaritySearch(
            SearchRequest.query(request.question()).withTopK(4)
        );
        
        // Build prompt with context
        String context = similar.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));
        
        // Generate answer
        // ...
    }
}
```

### Option B: Build an MCP Server in Java
```java
// Using MCP Java SDK
@Service
public class DocumentMcpServer {
    
    @Tool(description = "Search documents by semantic similarity")
    public String searchDocuments(@Param("query") String query) {
        // Call your vector store
        List<Document> results = vectorStore.similaritySearch(query);
        return results.stream()
            .map(doc -> doc.getContent())
            .collect(Collectors.joining("\n---\n"));
    }
}
```

### Option C: Deploy Publicly
- Deploy the FastAPI backend to a free tier (Railway, Render, Fly.io)
- Deploy Streamlit frontend to Streamlit Cloud (free)
- Use a cloud vector DB (Pinecone free tier or Chroma cloud)

---

## Complete Summary Table

| Week | Focus | What You Build | Key Skills |
|------|-------|---------------|------------|
| 0 (Days 1–3) | Theory Foundation | — | LLM concepts, RAG theory, MCP concepts |
| 1 (Days 4–10) | RAG Fundamentals | Mini Project 1: Text File Q&A, Mini Project 2: PDF Q&A | Embeddings, chunking, Chroma, LangChain |
| 2 (Days 11–17) | Document Chatbot Core | Main Project: AI Document Chatbot (backend + UI) | FastAPI, Streamlit, document ingestion, RAG pipeline |
| 3 (Days 18–24) | MCP + Advanced RAG | Mini Project 3: MCP Server + Advanced retrieval | MCP SDK, hybrid search, multi-query, RAGAS evaluation |
| 4 (Days 25–31) | Polish + Production | Docker, streaming, testing, README | Docker Compose, error handling, interview prep |
| 5 (Days 32–35) | Stretch Goals | Java version / deployment / extras | Spring AI, deployment, portfolio |

---

## Your GitHub Portfolio After 5 Weeks

| # | Repo | What It Demonstrates |
|---|------|---------------------|
| 1 | `mini-rag-basics` | Embeddings, vector search, first RAG pipeline |
| 2 | `pdf-qa-rag` | PDF processing, metadata, source citations |
| 3 | `ai-document-chatbot` | Full-stack RAG chatbot + web UI + MCP server |
| 4 | `document-mcp-server` | Custom MCP server (or bundled with #3) |

**One repo stands out:** `ai-document-chatbot` — this is a complete, deployable AI application with document upload, RAG-powered chat, source citations, MCP integration, Docker support, and evaluation metrics. That's more real-world AI engineering than most candidates can show.

---

## Resources Master List

### Courses (All Free)
| Course | Platform | Duration | When to Take |
|--------|----------|----------|-------------|
| [LangChain: Chat with Your Data](https://www.deeplearning.ai/short-courses/langchain-chat-with-your-data/) | DeepLearning.AI | ~2 hours | Week 1 |
| [Building and Evaluating Advanced RAG](https://www.deeplearning.ai/short-courses/building-evaluating-advanced-rag/) | DeepLearning.AI | ~1.5 hours | Week 3 |
| [LangChain for LLM Application Development](https://www.deeplearning.ai/short-courses/langchain-for-llm-application-development/) | DeepLearning.AI | ~1.5 hours | Week 1 |

### Documentation
| Resource | URL | What For |
|----------|-----|----------|
| LangChain Python Docs | [python.langchain.com](https://python.langchain.com/docs/introduction/) | RAG implementation reference |
| MCP Official Docs | [modelcontextprotocol.io](https://modelcontextprotocol.io) | MCP server development |
| MCP Python SDK | [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) | MCP code reference |
| Chroma Docs | [docs.trychroma.com](https://docs.trychroma.com) | Vector database usage |
| FastAPI Docs | [fastapi.tiangolo.com](https://fastapi.tiangolo.com) | Backend API |
| Streamlit Docs | [docs.streamlit.io](https://docs.streamlit.io) | Chat UI |
| RAGAS Docs | [docs.ragas.io](https://docs.ragas.io) | RAG evaluation |

### Videos
| Video | When to Watch |
|-------|--------------|
| [Andrej Karpathy - Intro to LLMs](https://www.youtube.com/watch?v=zjkBMFhNj_g) | Day 1 |
| [RAG from Scratch - LangChain playlist](https://www.youtube.com/playlist?list=PLfaIDFEXuae2LXbO1_PKyVJiQ23ZztA0x) | Week 1 |
| [IBM Technology - What is RAG?](https://www.youtube.com/watch?v=T-D1OfcDW1M) | Day 2 |

---

## The Hard Rules (Same as Your Other Roadmap)

> **1. You do not move to the next week until the current week's project works and is committed to GitHub.**

> **2. If you're stuck for more than 30 minutes, ask for help (ChatGPT, Claude, Stack Overflow, Discord). Don't waste time being stuck — that's not learning, that's suffering.**

> **3. Every day, write 2–3 sentences about what you learned in a `learning-log.md` file in your project repo. This helps retention AND shows employers you're thoughtful.**

---

## How This Connects to Your Bigger Roadmap

This 5-week sprint covers **Months 5, 6, and 9–10** of your 14–16 month roadmap in an accelerated, focused way:

| This Roadmap | Maps To | In Your Big Roadmap |
|-------------|---------|-------------------|
| Week 0–1 | Month 5 | Embeddings + Vector Databases |
| Week 2 | Month 6 | Full RAG Application |
| Week 3 | Month 9–10 | MCP Server Development |
| Week 4 | Month 11–12 | Production Polish + Evaluation |

After completing this, you can slot back into your bigger roadmap at Month 7–8 (microservices) with a much stronger foundation in RAG and MCP.
