# System Architecture Overview

This document summarizes the target capabilities for a Notion-like personal knowledge base with AI-assisted tables and RAG-enhanced chat. It captures the high-level component design, data models, and processing flows outlined in the project brief.

## 1. Component Breakdown

### Frontend (Next.js + React + TypeScript)
- Notion-style page editor with block-level editing.
- Smart table component for AI-assisted data entry.
- File upload experience (direct to backend or signed OSS upload).
- Chat UI for RAG question answering with contextual references.

### Backend API (Node.js + TypeScript)
- REST/GraphQL powered by NestJS or Express.
- Core modules: Auth, Workspace, Page, Block, File, Knowledge (embeddings and search), Chat.
- Background workers for file parsing and embedding generation.

### Storage
- Relational DB (MySQL/PostgreSQL) as the primary datastore.
- Object storage on Alibaba Cloud OSS.
- Vector store options: pgvector within Postgres or external services like Milvus/Qdrant/Pinecone.

### LLM / AI Services
- OpenAI Chat Completions / Responses API for generation.
- Embeddings API for vectorization.

## 2. OSS File Upload Patterns

Two supported flows:

1. **Direct upload (preferred)**
   - Backend issues a time-bound upload token/policy via `GET /api/files/upload-token`.
   - Frontend uploads directly to OSS, then calls `POST /api/files/confirm` with metadata to persist and enqueue parsing.
2. **Proxy upload**
   - Frontend sends `multipart/form-data` to `POST /api/files/upload`.
   - Backend uploads to OSS and removes temporary files.

## 3. Core Data Models

Outlined in SQL-like form for migrations:

- **Users/Workspaces**: users, workspaces, workspace_members.
- **Notion-style pages/blocks**: pages table with metadata and archival flag; blocks table with parent/child hierarchy and JSON content.
- **Files & parsing**: files table tracks upload/parsing status; file_chunks stores cleaned text segments; embeddings persists vectors with metadata.
- **Smart tables**: knowledge_tables defines schemas; knowledge_rows stores row data as JSON.

## 4. File Parsing & Embedding Pipeline

1. Frontend uploads file to OSS and confirms upload to backend.
2. Backend enqueues parsing job with file/workspace identifiers.
3. Worker downloads file from OSS and selects parser based on MIME type (PDF, DOCX, TXT, etc.).
4. Clean text, chunk on semantic boundaries (~500–1000 tokens), and insert `file_chunks` records.
5. Call OpenAI embeddings for each chunk and store vectors plus metadata in `embeddings`.
6. Update file status to `parsed` or `failed` on errors.

## 5. RAG Chat Workflow

- Endpoint: `POST /api/chat/ask` with workspace scope and optional file filters.
- Steps:
  1. Embed the user question and perform vector search (topK chunks).
  2. Build prompt with a system message restricting answers to provided context.
  3. Invoke OpenAI chat API and return the answer along with referenced chunks.

## 6. Smart Table AI Features

- Column types: text, number, date, tags, multi-select, boolean, file/page references, AI-generated, formulas.
- AI operations: summarization, auto-tagging, information extraction, and batch fill based on prompt templates.
- Example backend operation `POST /api/tables/:id/ai-fill-column` renders prompt per row, calls LLM, and writes results back to `knowledge_rows.data_json`.

## 7. Extraction from Files to Tables

- Users can map parsed documents (e.g., contracts) into structured rows.
- Endpoint `POST /api/contracts/extract-from-file` reads all chunks for a file, prompts LLM for JSON output matching target table schema, and inserts the row.

## 8. Suggested Implementation Phases

1. Backend foundations: Auth/Workspace/Page/Block CRUD.
2. Frontend foundations: auth flows and basic block editor.
3. File upload + OSS integration.
4. Parsing + embedding worker.
5. RAG chat endpoint and UI.
6. Smart tables and AI fill.
7. Advanced features: structured extraction, cross-linking, richer chat context handling.
