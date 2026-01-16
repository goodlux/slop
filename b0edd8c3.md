---
title: LSP Integration Strategy: Repolex + Claude Code
author: rob.kunkle
familiar: claude-sonnet-4-5-20250929
created: 2026-01-16T09:16:05.048864
tags: [lsp, claude-code, mcp, architecture, integration]
slop_id: b0edd8c3
---

# LSP Integration Strategy: Repolex + Claude Code

**Date:** 2025-12-23
**Context:** Claude Code v2.0.74 added native LSP support (Dec 2024)
**Strategic Goal:** Position repolex as semantic backend for Claude Code's LSP layer

---

## What Changed: Claude Code LSP Upgrade

### New LSP Capabilities (v2.0.74+)

Claude Code now has **native Language Server Protocol (LSP) tools**:

- **Go-to-definition**: Jump to symbol definitions
- **Find references**: Locate all symbol usages
- **Hover documentation**: Inline docs on hover
- **Document symbols**: Overview of file structure

### Current Limitation

**José Valim's insight (Twitter):**
> "Most LSP APIs are awkward for agentic usage because they require passing file:line:column. You can't simply ask 'tell me where Foo#bar is defined'."

**Translation:** LSP is great for humans clicking in editors, but Claude agents need semantic queries like "find all payment processing functions" or "show me the authentication flow".

---

## The Opportunity: Repolex as Semantic LSP Backend

### Problem LSP Solves (for humans)
- Navigate code by clicking
- See definitions on hover
- Find references in one file

### Problem Repolex Solves (for AI agents)
- **Semantic search**: "Find all functions that handle payments"
- **Cross-file analysis**: "Show authentication flow across modules"
- **Architecture queries**: "What are the main functional areas?"
- **Relationship queries**: "What depends on this class?"

### Combined Power: LSP + Repolex

```
Traditional LSP (file:line:col)
         ↓
Claude Code Agent
         ↓
Repolex Semantic Backend (SPARQL queries)
         ↓
Knowledge Graph (TreeSitter + NLP)
```

---

## Integration Architecture

### Layer 1: High-Level Overview (Repolex-Generated)

**Purpose:** Give Claude agent architectural understanding before diving into code.

```turtle
# Repository structure graph
:gliner2 a repo:Repository ;
    repo:hasFunctionalArea :gliner2_model_core ;
    repo:hasFunctionalArea :gliner2_training ;
    repo:hasFunctionalArea :gliner2_inference ;
    repo:primaryLanguage "Python" ;
    repo:architecturePattern "Transformer-based NER" ;
    schema:url <https://github.com/fastino-ai/GLiNER2> .

:gliner2_model_core a repo:FunctionalArea ;
    rdfs:label "Model Architecture" ;
    repo:containsModule :gliner2/model ;
    repo:keyClass :GLiNERModel ;
    repo:responsibility "Define transformer architecture for entity extraction" .
```

**Query Example:**
```sparql
# Claude asks: "What are the main functional areas?"
SELECT ?area ?label ?responsibility WHERE {
    :gliner2 repo:hasFunctionalArea ?area .
    ?area rdfs:label ?label ;
          repo:responsibility ?responsibility .
}
```

**Result:** Claude understands the repo has 3 main areas (model, training, inference) before looking at any code.

### Layer 2: Functional Area Detail (Repolex-Generated)

**Purpose:** Understand module structure and key APIs.

```turtle
:gliner2_model_core a repo:FunctionalArea ;
    repo:containsModule :gliner2/model ;
    repo:keyClass :GLiNERModel .

:GLiNERModel a lang:class_definition ;
    rdfs:label "GLiNERModel" ;
    woc:hasMethod :GLiNERModel.forward ;
    woc:hasMethod :GLiNERModel.encode ;
    repo:purpose "Main model class for entity extraction" .
```

### Layer 3: Full AST (Current Code Graph)

**Purpose:** Deep code analysis with complete TreeSitter parse tree.

```turtle
lang:function_definition rdfs:label "forward" ;
    woc:hasParameter :param_input_ids ;
    woc:sourceText "def forward(self, input_ids, attention_mask)..." ;
    woc:startLine 127 ;
    woc:definedInFile <.../model.py> .
```

---

## Integration Points with Claude Code

### 1. **MCP Server for Repolex** (Highest Priority)

Create `mcp-server-repolex` to expose semantic queries to Claude Code agents.

```typescript
// Tool: semantic_search
{
  name: "repolex_semantic_search",
  description: "Search code semantically using natural language",
  inputSchema: {
    query: "string",  // e.g., "find authentication functions"
    repo: "string",   // e.g., "fastino-ai/GLiNER2"
    scope: "functions|classes|modules"  // optional filter
  }
}

// Tool: get_architecture_overview
{
  name: "repolex_get_architecture",
  description: "Get high-level architecture overview of repository",
  inputSchema: {
    repo: "string"
  }
}
```

### 2. **Enhanced Hover Documentation** (via LSP + Repolex)

**Current LSP Hover:**
```python
class GLiNERModel:
    """Main model class"""
```

**Repolex-Enhanced Hover:**
```python
class GLiNERModel:
    """Main model class for entity extraction"""

    # Semantic Context (from Repolex):
    Functional Area: Model Architecture
    Purpose: Define transformer architecture for entity extraction
    Key Methods: forward, encode, batch_predict
    Used By: GLiNERTrainer, InferencePipeline
    Dependencies: transformers.PreTrainedModel
```

### 3. **Semantic Go-to-Definition** (Complement LSP)

**Traditional LSP go-to-definition:**
- User clicks `model.forward()` → jumps to definition

**Repolex semantic navigation:**
- Claude asks "Show me all functions involved in the prediction pipeline"
- Repolex returns: `encode()` → `forward()` → `batch_predict()` → `post_process()`
- Claude can navigate the **semantic flow**, not just individual definitions

### 4. **Cross-Repo Context** (Repolex Unique Capability)

**Scenario:** User is working in `my-app` which uses `GLiNER2` as dependency.

```python
# In my-app/inference.py
from gliner import GLiNERModel

model = GLiNERModel.from_pretrained("...")
```

**Traditional LSP:** Can't jump to `GLiNERModel` definition (external package)

**Repolex-powered:**
1. Detects import from `gliner` package
2. Queries Repolex knowledge graph for `fastino-ai/GLiNER2`
3. Returns semantic context + GitHub link
4. Claude can explain how `GLiNERModel` works even though it's in external repo

---

## Implementation Roadmap

### Phase 1: MCP Server (2-3 weeks)

**Goal:** Claude Code agents can query repolex via MCP tools.

**Deliverables:**
- `mcp-server-repolex` package
- Tools: `semantic_search`, `get_architecture`, `find_relationships`
- Installation: `npx @repolex/mcp-server-repolex`
- Demo: Claude Code agent uses repolex to explain GLiNER2 architecture

### Phase 2: Layer 1 + Layer 2 Graphs (3-4 weeks)

**Goal:** Generate high-level overview and functional area graphs.

**Deliverables:**
- Layer 1 generator: Creates repository overview graph
- Layer 2 generator: Creates functional area graphs
- Auto-detect functional areas from directory structure + code analysis
- Add GitHub URL generation to all entities

### Phase 3: Web Demo (2 weeks)

**Goal:** Demonstrate repolex capabilities to compete with Google CodeWiki.

**Features:**
- Parse public GitHub repo (paste URL)
- Generate Layer 1 + Layer 2 + Layer 3 graphs
- Interactive visualization (D3.js force graph)
- Natural language Q&A powered by Claude API + SPARQL
- GitHub deep links on all entities

**Stack:**
- Frontend: Next.js + shadcn/ui
- Visualization: D3.js or Cytoscape.js
- Backend: FastAPI serving SPARQL endpoint
- LLM: Claude API for NL → SPARQL translation

---

## Competitive Analysis: Repolex vs. Google CodeWiki

| Feature | Google CodeWiki | Repolex |
|---------|-----------------|---------|
| **Code Graph** | ✓ (proprietary) | ✓ (TreeSitter + RDF) |
| **Multi-language** | ✓ (Google-scale) | ✓ (100+ via TreeSitter) |
| **LLM Explanations** | ✓ (Gemini) | ✓ (Claude via MCP) |
| **GitHub Links** | ✓ | ✓ (TODO) |
| **Open Source** | ✗ | ✓ (eventually) |
| **Claude Code Integration** | ✗ | ✓ (via MCP) |
| **Semantic Queries** | ✗ (likely) | ✓ (SPARQL) |
| **Cross-Repo Context** | ✗ (likely) | ✓ (multi-graph) |

**Repolex Advantages:**
1. **Claude Code native integration** (MCP)
2. **Semantic queries** (SPARQL, not just text search)
3. **Open ontologies** (TreeSitter RDF)
4. **Cross-repo analysis** (query multiple graphs)
5. **Anthropic partnership potential**

---

## Why This Matters Now

### LSP Fills the Human Gap

LSP gives Claude Code **traditional IDE features** for **human developers**.

### Repolex Fills the AI Agent Gap

Repolex gives Claude Code **semantic understanding** for **AI agents**.

### Together: Best of Both Worlds

```
Human Developer Mode:
    User clicks → LSP go-to-definition → See code

AI Agent Mode:
    Claude asks → Repolex semantic search → Understand architecture
```

### José Valim's Point

> "LSP APIs are awkward for agentic usage because they require file:line:column"

**Repolex solves this:** Agents can ask "what are the payment functions?" not "go to payment.py:127".

---

## Summary

**Claude Code LSP upgrade changes the game because:**

1. **It validates the need for code intelligence** - Anthropic invested in LSP because developers need it
2. **It creates an integration point** - MCP + LSP = powerful combo
3. **It highlights the gap** - LSP is great for humans, awkward for agents
4. **Repolex fills that gap** - Semantic queries for AI agents

**Strategic positioning:**

> "Repolex is the semantic backend for Claude Code's LSP layer. LSP handles human navigation (go-to-definition). Repolex handles AI agent understanding (semantic search, architecture queries, cross-repo context)."

**Next milestone:** Working MCP server + Layer 1/2 graphs for GLiNER2 demo.
