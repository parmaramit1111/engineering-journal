# Building a RAG Knowledge Assistant from the Ground Up: Architecture, Decisions, and Lessons Learned

**Category:** AI
**Article Type:** Engineering Case Study
**Difficulty:** Advanced
**Reading Time:** 15 minutes
**Published:** August 2026

---

## Overview

Retrieval-Augmented Generation, commonly known as RAG, has become one of the most practical ways to build AI applications that work with private or domain-specific knowledge.

At a high level, the concept looks simple:

```text
Documents
    ↓
Embeddings
    ↓
Vector Database
    ↓
Semantic Search
    ↓
LLM
    ↓
Answer
```

But building a RAG application that is clean, maintainable, extensible, and understandable is a very different problem.

The moment we move beyond a proof of concept, several engineering questions appear:

* How should different document formats be handled?
* Where should parsing logic live?
* How should documents be chunked?
* How do we make embedding providers interchangeable?
* Where should vector storage belong?
* How should background processing work?
* How do we prevent the business layer from becoming tightly coupled to an LLM provider?
* How should retrieved context be assembled?
* How do we keep the system testable?
* How much production infrastructure should actually be built?

This article documents my experience building a complete **RAG Knowledge Assistant from the ground up**, with the primary objective of demonstrating architecture, engineering practices, technical decisions, and lessons learned.

The result is a complete full-stack RAG showcase built with Python, FastAPI, React, PostgreSQL, pgvector, Sentence Transformers, and Ollama.

**Project Repository:**
[Knowledge Assistant RAG Showcase](https://github.com/parmaramit1111/knowledge-assistant)

---

# Why Build Another RAG Application?

There are already many RAG tutorials and example projects.

Most of them demonstrate the basic concept:

```text
Load PDF
   ↓
Split Text
   ↓
Create Embeddings
   ↓
Store Vectors
   ↓
Ask LLM
```

That is useful for learning RAG.

But my objective was different.

I wanted to answer a broader engineering question:

> **How would I architect a RAG application if I wanted the codebase to remain clean as the system grows?**

That changed the way the project was approached.

Instead of starting with an LLM API and building outward, I started with architecture.

The objective was to create a system where:

* business workflows are clearly separated,
* providers can be replaced,
* persistence is isolated,
* background processing is explicit,
* APIs remain thin,
* AI providers do not leak into business logic,
* and the complete RAG pipeline can be understood by another engineer.

This became more of an engineering exercise than simply an AI experiment.

---

# The Goal

The project had a very specific scope.

Build a working Knowledge Assistant capable of:

* uploading documents,
* parsing multiple formats,
* chunking documents,
* generating embeddings,
* storing embeddings,
* performing semantic search,
* constructing grounded prompts,
* generating responses using an LLM,
* returning source references,
* and providing a usable React interface.

The supported document formats include:

* PDF
* DOCX
* TXT
* HTML
* Markdown

The final system follows a complete ingestion and retrieval pipeline.

```text
                     DOCUMENT INGESTION

Upload Document
       │
       ▼
Parse Document
       │
       ▼
Chunk Document
       │
       ▼
Generate Embeddings
       │
       ▼
PostgreSQL + pgvector
       │
       │
       ▼
                     QUERY PIPELINE

User Question
       │
       ▼
Query Embedding
       │
       ▼
Semantic Search
       │
       ▼
Relevant Chunks
       │
       ▼
Prompt Builder
       │
       ▼
LLM Provider
       │
       ▼
Grounded Response
       │
       ▼
Source References
```

The important part is that every stage has a clear responsibility.

---

# Architecture Before Implementation

One of the earliest decisions was to avoid building the application around a specific AI library or provider.

The architecture was designed around application responsibilities first.

The backend follows a layered approach using:

* Clean Architecture principles
* CQRS
* Repository Pattern
* ExecutionContext
* Workflow Services
* Provider Services
* Factory Pattern
* Background Workers
* Async-first design

The high-level request flow looks like this:

```text
HTTP Request
     │
     ▼
FastAPI Controller
     │
     ▼
ExecutionContext
     │
     ▼
Command / Query
     │
     ▼
Workflow Service
     │
     ├───────────────┐
     ▼               ▼
Provider Service   Repository
     │               │
     ▼               ▼
Provider Factory   PostgreSQL
     │
     ▼
Concrete Provider
```

The important principle is simple:

> **Business workflow should not depend directly on infrastructure or third-party provider implementations.**

That principle influenced almost every major decision in the project.

---

# The RAG Pipeline

The complete pipeline consists of several independent stages.

```text
Upload
   ↓
Parse
   ↓
Chunk
   ↓
Embed
   ↓
Store
   ↓
Retrieve
   ↓
Build Prompt
   ↓
Generate
   ↓
Return Sources
```

Each stage produces an output that becomes the input to the next stage.

This makes the pipeline easier to understand, debug, replace, and extend.

---

# Stage 1 — Document Upload

Everything starts with a document.

The API accepts an uploaded file and performs the initial validation and persistence.

The upload stage is responsible for:

* accepting the document,
* validating the file type,
* storing the original file,
* creating the document record,
* and initializing the processing workflow.

At this point, the system does not care how the document will eventually be parsed.

It simply knows:

```text
Document Uploaded
       ↓
Processing Required
```

This separation is intentional.

The HTTP request should not become responsible for performing the entire RAG pipeline.

---

# Stage 2 — Document Parsing

Once a document is uploaded, the system needs to extract usable text.

Different formats require different parsing strategies.

The project therefore uses a parser abstraction and factory.

```text
ParserFactory
      │
      ├── PDF Parser
      ├── DOCX Parser
      ├── TXT Parser
      ├── HTML Parser
      └── Markdown Parser
```

The workflow does not need to know how a PDF is parsed.

It asks for a parser capable of handling the document.

The parser returns normalized document content.

```text
Document
   ↓
Parser
   ↓
ParsedDocument
```

This was one of the first places where provider abstraction proved useful.

Adding another document format should not require modifying the document workflow itself.

---

# Stage 3 — Document Chunking

A large document cannot simply be sent to an embedding model or LLM as one giant block of text.

It needs to be divided into smaller pieces.

The current implementation uses a recursive character splitter with:

```text
Chunk Size:     800
Chunk Overlap:  200
```

The actual numbers are configurable engineering choices rather than universal truths.

The important architectural decision was to make chunking its own provider abstraction.

```text
ParsedDocument
       │
       ▼
ChunkerFactory
       │
       ▼
Recursive Chunker
       │
       ▼
DocumentChunk[]
```

This leaves the system open to future strategies such as:

* semantic chunking,
* Markdown-aware chunking,
* token-based chunking,
* domain-specific chunking.

The surrounding workflow does not need to change when the chunking strategy changes.

---

# Stage 4 — Embeddings

Once documents are chunked, each chunk needs to be converted into a numerical representation.

That representation is the embedding.

The current implementation uses:

```text
Sentence Transformers
        │
        ▼
all-MiniLM-L6-v2
        │
        ▼
384-dimensional vector
```

The pipeline becomes:

```text
DocumentChunk
      │
      ▼
Embedding Service
      │
      ▼
Embedding Provider
      │
      ▼
Vector
      │
      ▼
DocumentChunkEmbedding
```

Embeddings are stored using PostgreSQL and pgvector.

The same abstraction is also used when embedding user queries.

This is important because document embeddings and query embeddings need to exist in the same vector space for meaningful similarity search.

---

# Why PostgreSQL + pgvector?

One of the major architectural decisions was using PostgreSQL with pgvector instead of immediately introducing a dedicated vector database.

There are many excellent vector databases available.

However, this project already needed relational storage for:

* documents,
* parsed documents,
* chunks,
* processing state,
* metadata,
* relationships,
* and embeddings.

Using PostgreSQL with pgvector allowed those concerns to remain together.

The resulting architecture is straightforward:

```text
                   PostgreSQL
                       │
             ┌─────────┴─────────┐
             │                   │
       Relational Data       pgvector
             │                   │
       Documents             Embeddings
       Chunks                Similarity Search
       Metadata
       Workflow State
```

This reduced infrastructure complexity while still providing vector similarity search.

For a showcase project, that was a good trade-off.

---

# Stage 5 — Semantic Search

Once embeddings exist, the system can answer a different question:

> Which pieces of the knowledge base are most relevant to this question?

The process is:

```text
User Question
      │
      ▼
Query Embedding
      │
      ▼
Vector Similarity Search
      │
      ▼
Top-K Results
      │
      ▼
Ranked Document Chunks
```

The current implementation uses cosine similarity with PostgreSQL and pgvector.

The search layer returns relevant chunks along with document metadata.

This gives the application the context required for the next stage.

The important separation here is:

```text
Search ≠ Generation
```

The retrieval system finds the information.

The LLM generates the response.

Those responsibilities should remain separate.

---

# Stage 6 — Prompt Builder

After retrieval, the application has a collection of relevant document chunks.

Those chunks need to be transformed into useful context for the LLM.

This is where the Prompt Builder comes in.

The flow becomes:

```text
Search Results
      │
      ▼
Context Assembly
      │
      ▼
Prompt Builder
      │
      ▼
Prompt Provider
      │
      ▼
LLM Prompt
```

The prompt builder is responsible for:

* assembling retrieved context,
* injecting the user question,
* applying the prompt structure,
* maintaining source-aware context,
* and preparing the final prompt for the LLM.

Again, the prompt logic is not hardcoded into the chat workflow.

A dedicated provider architecture allows different prompt strategies to be introduced later.

---

# Stage 7 — LLM Generation

The final stage is generation.

The current implementation uses Ollama as the LLM provider.

```text
LLMFactory
     │
     ▼
Ollama Provider
     │
     ▼
qwen2.5:1.5b
```

The chat workflow does not directly depend on Ollama.

Instead:

```text
DocumentChatService
        │
        ▼
LLMService
        │
        ▼
LLMFactory
        │
        ▼
Ollama Provider
```

This distinction is important.

If the application had directly called Ollama from the business service, replacing it later with OpenAI, Anthropic, Gemini, or another provider would require modifying business logic.

With the provider abstraction, the workflow remains unchanged.

---

# The Complete Chat Flow

The final chat request can therefore be represented as:

```text
User Question
      │
      ▼
AskQuestionCommand
      │
      ▼
DocumentChatService
      │
      ├───────────────► DocumentSearchService
      │                       │
      │                       ▼
      │                 Relevant Chunks
      │
      ├───────────────► PromptBuilderService
      │                       │
      │                       ▼
      │                    Prompt
      │
      └───────────────► LLMService
                              │
                              ▼
                         LLM Provider
                              │
                              ▼
                     Grounded Response
                              │
                              ▼
                       Source References
```

This became one of the most satisfying parts of the implementation.

The final chat operation looks simple because the complexity has been moved into well-defined components.

---

# Background Processing

Document processing can become expensive.

Parsing, chunking, and embedding should not unnecessarily block the user's upload request.

The project therefore uses background workers.

```text
Scheduler
    │
    ▼
Worker
    │
    ▼
ExecutionContext
    │
    ▼
Command
    │
    ▼
Workflow Service
    │
    ▼
Provider Service
    │
    ▼
Repository
```

The current workers include:

* DocumentWorker
* ChunkWorker
* EmbeddingWorker

This gives each stage a clear execution boundary.

For example:

```text
Upload
  │
  ▼
DocumentWorker
  │
  ▼
ParsedDocument
  │
  ▼
ChunkWorker
  │
  ▼
DocumentChunk
  │
  ▼
EmbeddingWorker
  │
  ▼
DocumentChunkEmbedding
```

This approach also provides a natural place to introduce retry and failure handling in a future production implementation.

---

# ExecutionContext

One architectural component that became particularly useful was the `ExecutionContext`.

The purpose of the context is to define the lifecycle of an execution.

It can provide:

* database session,
* repositories,
* services,
* transaction scope,
* and execution-specific state.

The same concept can be used by both HTTP requests and background workers.

That creates a consistent execution model:

```text
HTTP Request
     │
     ▼
ExecutionContext
     │
     ▼
Command
```

and:

```text
Background Worker
     │
     ▼
ExecutionContext
     │
     ▼
Command
```

This reduces the difference between synchronous and asynchronous workflows from an application architecture perspective.

---

# CQRS

The project also separates commands and queries.

Commands represent operations that change state.

Examples include:

```text
UploadDocumentCommand
ParseDocumentCommand
ChunkDocumentCommand
EmbedDocumentCommand
SearchDocumentsCommand
AskQuestionCommand
```

Queries represent read operations.

Examples include:

```text
GetDocumentQuery
HealthCheckQuery
```

The goal was not to introduce CQRS because it is fashionable.

The goal was to make application intent explicit.

When reading the code, an engineer should be able to quickly determine whether a particular operation is:

```text
Changing state
```

or:

```text
Reading state
```

That clarity becomes increasingly valuable as an application grows.

---

# Repository Pattern

Persistence was isolated behind repositories.

The application workflow should not contain raw database operations everywhere.

Instead:

```text
Workflow Service
      │
      ▼
Repository
      │
      ▼
SQLAlchemy
      │
      ▼
PostgreSQL
```

Repositories are responsible for data access.

Workflow services are responsible for orchestration.

Providers are responsible for integrations.

Controllers are responsible for HTTP.

That separation keeps responsibilities clear.

---

# FastAPI as the Application Boundary

FastAPI was used as the backend API framework.

The API layer remains intentionally thin.

The controller receives the request and delegates to the application layer.

```text
HTTP
 │
 ▼
Controller
 │
 ▼
Command / Query
 │
 ▼
Workflow
```

The controller should not:

* perform vector searches directly,
* construct complex prompts,
* call LLM providers directly,
* parse documents,
* or contain persistence logic.

Those responsibilities belong elsewhere.

This makes the API layer easier to understand and easier to change.

---

# Frontend Architecture

A RAG backend is only half of the user experience.

The project also includes a React and TypeScript frontend using Material UI.

The interface provides:

* document upload,
* drag-and-drop upload,
* chat interaction,
* source references,
* new chat functionality,
* navigation,
* loading states,
* and responsive UI behavior.

The frontend communicates with the FastAPI backend through the API layer.

Conceptually:

```text
React
  │
  ▼
HTTP API
  │
  ▼
FastAPI
  │
  ▼
RAG Pipeline
```

The frontend was deliberately kept independent from the backend implementation details.

That means the backend can evolve without requiring the UI to understand its internal architecture.

---

# Why We Did Not Build Everything

This was probably one of the most important decisions made during the project.

Once the basic RAG pipeline was working, it would have been easy to continue indefinitely.

The project could have expanded into:

```text
Authentication
Authorization
RBAC
Multi-Tenancy
S3
CI/CD
Monitoring
Metrics
Rate Limiting
Billing
Admin Portal
Cloud Infrastructure
Distributed Workers
```

But that was not the objective.

The purpose of this project was to demonstrate:

* architectural thinking,
* coding structure,
* RAG understanding,
* provider abstraction,
* full-stack engineering,
* and the ability to make deliberate technical decisions.

At some point, adding more infrastructure would stop increasing the value of the showcase.

So we stopped.

That decision is important because good engineering is not only about knowing what to build.

It is also about knowing **what not to build**.

---

# Showcase Project vs Private Production System

The resulting repository is intentionally a showcase implementation.

The architecture can serve as a reference for a future private RAG system, but the two projects should not necessarily have identical requirements.

A production system may eventually need:

```text
Private RAG
    │
    ├── Authentication
    ├── Authorization
    ├── S3
    ├── Monitoring
    ├── Metrics
    ├── Security
    ├── Tenant Isolation
    ├── Production Infrastructure
    └── Operational Controls
```

Those concerns are valid when the business requires them.

They simply do not belong in the scope of this particular showcase.

Keeping that boundary helped us finish the project instead of continuously expanding it.

---

# What Worked Well

Several architectural decisions proved particularly valuable.

## Provider Abstraction

The ability to represent parsers, embeddings, prompts, and LLMs as providers kept external technologies isolated.

---

## Factory Pattern

Factories made provider selection explicit.

```text
ParserFactory
EmbeddingFactory
PromptFactory
LLMFactory
```

This keeps provider selection away from the core business workflow.

---

## Workflow Services

Workflow services provided a clear place to orchestrate business processes.

Instead of putting everything inside controllers, the application has dedicated workflows for:

* document processing,
* chunking,
* embedding,
* searching,
* and chat.

---

## Background Workers

Separating document processing from the HTTP request improved the overall application design and provided a natural execution model for expensive operations.

---

## PostgreSQL + pgvector

Using PostgreSQL for both relational data and vector search kept the initial infrastructure simple.

---

## Local AI

Using Sentence Transformers and Ollama allowed the complete pipeline to be developed without requiring every AI operation to depend on a paid external API.

---

# What Could Be Improved

No engineering project is perfect.

The current implementation provides a strong foundation, but several areas could be improved in a future iteration.

## Better Chunking

The current recursive chunking strategy is a good baseline.

However, different domains may benefit from:

* semantic chunking,
* structure-aware chunking,
* Markdown-aware splitting,
* token-aware splitting,
* or domain-specific strategies.

---

## Better Retrieval

Semantic similarity is a strong starting point.

More advanced systems can combine:

```text
Vector Search
     +
Keyword Search
     +
Metadata Filtering
     +
Re-ranking
```

The important point is that the architecture already provides a place for those improvements.

---

## Better Embeddings

The current model is useful for local experimentation.

Different domains may benefit from larger or specialized embedding models.

The provider architecture allows those models to be evaluated without redesigning the application.

---

## Better LLMs

The current local LLM is intentionally lightweight.

A production workload may require a more capable model depending on:

* context size,
* reasoning requirements,
* latency,
* cost,
* domain complexity,
* and response quality.

Again, the LLM abstraction keeps this decision independent from the application workflow.

---

# The Most Important Lesson

The biggest lesson from this project was not about embeddings.

It was not about pgvector.

It was not even about RAG.

It was about **boundaries**.

AI applications can become tightly coupled very quickly.

For example:

```text
Controller
   ↓
LangChain
   ↓
OpenAI
   ↓
Vector DB
```

This can work.

But once the application grows, changing one part can affect everything else.

The architecture we built instead tries to maintain explicit boundaries:

```text
API
 │
 ▼
Application Workflow
 │
 ├── Providers
 │
 └── Repositories
       │
       ▼
   Infrastructure
```

That makes change cheaper.

And in my experience, the ability to change a system safely is one of the most important characteristics of good software architecture.

---

# Another Important Lesson: Don't Hide Complexity

There is a temptation in AI development to hide everything behind a framework.

For example:

```text
RAG Framework
     ↓
Everything Works
```

That is convenient, but it can make it difficult to understand what is actually happening.

Building the pipeline explicitly forced us to understand:

* document parsing,
* chunking,
* embedding generation,
* vector persistence,
* similarity search,
* context assembly,
* prompt generation,
* and LLM invocation.

That understanding is much more valuable than simply knowing which library function to call.

Frameworks are useful.

Understanding the system underneath them is better.

---

# Another Lesson: Architecture Should Follow the Problem

It is easy to over-engineer AI systems.

A project can quickly accumulate:

```text
Microservices
Event Bus
Message Queue
Vector Database
Cache
Search Engine
LLM Gateway
Feature Store
Observability Platform
```

before there is even a working RAG pipeline.

We deliberately avoided that.

The initial system uses:

```text
React
   +
FastAPI
   +
PostgreSQL
   +
pgvector
   +
Sentence Transformers
   +
Ollama
```

That is enough to demonstrate the complete problem.

The architecture remains extensible without requiring unnecessary infrastructure from day one.

---

# The Final System

After completing the implementation, the complete system can be represented like this:

```text
                         KNOWLEDGE ASSISTANT

┌───────────────────────────────────────────────────────────┐
│                    React + TypeScript                     │
│                                                           │
│      Document Upload       Chat       Source References   │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│                       FastAPI API                         │
│                                                           │
│  Controllers                                               │
│       ↓                                                   │
│  Commands / Queries                                       │
│       ↓                                                   │
│  Workflow Services                                        │
│       ↓                                                   │
│  Provider Services                                        │
└───────────────┬───────────────────────────┬───────────────┘
                │                           │
                ▼                           ▼
       Provider Architecture        Repository Architecture
                │                           │
      ┌─────────┼─────────┐                 │
      │         │         │                 ▼
    Parser   Embedding    LLM          PostgreSQL
      │         │         │             + pgvector
      │         │         │
      └─────────┴─────────┘
                │
                ▼
           RAG Pipeline
                │
                ▼
       Grounded AI Response
```

The resulting pipeline is:

```text
Upload
  ↓
Parse
  ↓
Chunk
  ↓
Embed
  ↓
Store
  ↓
Search
  ↓
Build Context
  ↓
Generate
  ↓
Return Sources
```

Simple to understand.

Structured internally.

Extensible when required.

---

# Release

After completing the architecture, implementation, frontend, Docker configuration, scripts, and technical documentation, the project was released as:

**Knowledge Assistant RAG Showcase v1.0.0**

The release represents the completion of the original showcase objective.

The repository is now a reference implementation rather than an unfinished prototype.

It demonstrates not only that RAG works, but how I approach the engineering problem around it.

---

# Final Thoughts

Building a RAG application is easy to demonstrate.

Building one that is understandable, modular, and deliberately scoped is a much more interesting engineering exercise.

The most valuable outcome of this project was not the chat screen.

It was the architecture behind it.

We started with a simple question:

> How should a RAG application be designed if we care about clean architecture and long-term maintainability?

The answer became a system built around:

* explicit workflows,
* clear boundaries,
* provider abstraction,
* factories,
* repositories,
* background processing,
* vector search,
* grounded generation,
* and a clean frontend.

The project also reinforced something I have learned repeatedly throughout my engineering career:

> **Good architecture is not about adding more layers. It is about creating the right boundaries.**

The right abstraction allows a provider to change without changing the workflow.

The right workflow allows the implementation to evolve without changing the API.

The right scope allows the project to actually reach completion.

And the right engineering mindset means knowing when the problem has been solved well enough to stop.

That is what this project was ultimately about.

Not building the biggest RAG platform.

Building a clean one.

---

# Key Takeaways

* RAG is a pipeline, not a single feature.
* Document ingestion deserves the same architectural attention as LLM generation.
* Parsing, chunking, embedding, retrieval, prompting, and generation should remain separate concerns.
* Provider abstraction prevents AI vendors and libraries from leaking into business workflows.
* PostgreSQL + pgvector can provide a simple and effective foundation for many RAG applications.
* Background workers are useful for separating expensive document processing from HTTP requests.
* Local embeddings and local LLMs are excellent for experimentation and architectural development.
* CQRS and workflow services can make application intent easier to understand.
* A good showcase project does not need to become a complete SaaS platform.
* Knowing what **not** to build is an important part of engineering.
* The best architecture is not the one with the most components; it is the one with the clearest boundaries.

---

# Project

**Knowledge Assistant RAG Showcase v1.0.0**

[View the source code and architecture on GitHub](https://github.com/parmaramit1111/knowledge-assistant)

---

# Related Engineering Topics

- Retrieval-Augmented Generation
- Large Language Models
- Vector Search
- Embeddings
- Semantic Search
- PostgreSQL
- pgvector
- FastAPI
- Clean Architecture
- CQRS
- Repository Pattern
- Provider Pattern
- Factory Pattern
- Background Processing
- AI Application Architecture
- Full-Stack AI Development

---

**Article Version:** 1.0

**First Published:** August 2026

**Last Reviewed:** August 2026