**AI CTF Automator**

Project Spec & Module Breakdown

# **1. What we're building**

A multi-agent system that autonomously solves CTF challenges. The goal is a working system that can participate in real CTF competitions, not just a research demo. We want something that actually does well, not something that looks good on paper.

High level: user drops in a challenge (description, files, docker env), the system figures out the category, builds a plan, retrieves relevant writeups and payloads from our knowledge base, executes tools, and submits the flag. Human stays in the loop optionally.

Not trying to beat SOTA benchmarks on day 1. Trying to beat the baseline and have a clean, extendable codebase.

# **2. Architecture**

## **2.1 Overview**

Four layers: intake, agents, RAG engine, tool execution. MCP (Model Context Protocol) standardizes the tool layer so every tool call goes through the same interface - this reduces hallucination and makes testing easier.

| **Layer** | **What it does** | **Key tech** |
| --- | --- | --- |
| Intake | Takes challenge input, classifies category, seeds context for planner | Auto-prompter agent, Docker env probe |
| Agents | Planner breaks problem into subtasks. Two executors handle them in parallel or sequence |Some Open Source Model, ReAct loop |
| RAG engine | Retrieves relevant writeups/payloads from KB. Grades quality, retries if bad, falls back to web search | Hybrid BM25+dense, CRAG, cross-encoder rerank |
| Tool execution | Runs all actual shell/python/binary tools inside Docker sandbox via MCP | MCP server, pwntools, ghidra, gdb |

## **2.2 Agent design**

Three agents total. Same model tier for all of them - research showed that mixing strong/weak models in planner/executor drops performance 5-6%.

### **Auto-prompter**

First thing that runs. Probes the challenge environment, identifies the category (web / pwn / crypto / forensics / rev / misc), and writes a seed context block for the planner. This saves the planner from wasting its first few turns just figuring out what it's looking at.

### **Planner**

Receives the seed context and builds a task tree. Breaks the challenge into concrete subtasks (e.g. "extract binary, find overflow point, build payload, send to server"). Monitors executor progress and reassigns or adjusts when something fails. Does not directly run tools.

### **Executors (x2)**

Two executors, not three. One handles web/crypto/forensics. One handles pwn/rev. Both have the same capabilities - the split is just to allow parallel work on challenges with multiple sub-problems. Each executor runs its own ReAct loop: think, call tool, observe, repeat.

**Note:** *Every tool observation gets verified against actual env state before the next step. If the model claims a tool returned something it didn't, the loop catches it. This prevents the 'soliloquizing' problem where the agent hallucinates tool outputs.*

## **2.3 RAG engine**

This is the main thing that separates us from a bare LLM agent. The retrieval pipeline has four stages:

### **Stage 1 - Query formation (HyDE)**

Instead of sending the raw task description to the vector DB, we first ask the model to generate a hypothetical writeup for a similar challenge. Then we embed that and search on it. This works much better because the gap between 'question embedding' and 'answer embedding' is large in CTF space - writeups look nothing like challenge descriptions.

### **Stage 2 - Hybrid retrieval**

Two retrieval methods run in parallel, results fused with Reciprocal Rank Fusion (RRF):

* BM25 (keyword): catches exact tool names, CVE numbers, function names, specific flags
* Dense (semantic): catches conceptually similar writeups even if phrased differently

Then a cross-encoder reranks the top 50 results. Research shows this two-stage setup gets ~27% better nDCG@10 vs dense-only. BM25 alone actually beats dense on most CTF-specific terms.

### **Stage 3 - CRAG (Corrective RAG)**

Grades each retrieved document strip individually. If a strip scores below threshold it either rewrites the query and retries, or falls back to web search. This is more surgical than Self-RAG's full-doc retry and wastes less context on bad retrievals.

### **Stage 4 - Hint injection**

Takes the graded, reranked results and injects relevant snippets directly into the executor's prompt as hints. Not the full writeup - just the relevant technique or payload section.

## **2.4 Knowledge base**

Four sources, all indexed into the same retrieval pipeline:

| **Source** | **Contents** | **Format** | **Update frequency** |
| --- | --- | --- | --- |
| CTF writeups | Scraped from CTFtime, GitHub writeup repos, personal writeups | Chunked by section, embedded | Manual + periodic scrape |
| PayloadsAllTheThings | Web, pwn, crypto, rev payloads and bypass techniques | Chunked by category | Pull from upstream on update |
| CVE/exploit refs | CVE descriptions, PoC links, affected versions | Short chunks, BM25-friendly | Weekly pull |
| Web search fallback | Live search for novel CVEs or recent challenges | Not indexed - live at query time | Always live |

**Note:** *We're using HyPE (Hypothetical Prompt Embeddings) at ingest time - pre-generate 3-5 questions each chunk can answer and store those as additional vectors. This improves recall by ~40% without any query-time cost.*

## **2.5 Routing (adaptive)**

Not every subtask needs retrieval. The planner tags each subtask with a complexity score. Low-complexity subtasks (e.g. 'decode this base64', 'write a Python XOR loop') skip the RAG pipeline entirely and go straight to the executor. This cuts cost and latency significantly on easy challenges.

## **2.6 Context management**

Long challenges will blow the context window if we're not careful. We use a rolling summary approach: after every N turns, compress the history into a structured scratchpad. The scratchpad has three fields:

* Current plan (from planner)
* Completed steps and their outcomes
* Active tool state (what's running, what's been tried)

Full turn history is dropped after compression. Only the scratchpad + last 2-3 tool outputs stay in context.

# **3. Module breakdown**

Six people, six modules. Interfaces are defined below so work can happen in parallel. Each module has a clear input/output contract.

| **Person** | **Module** | **Deliverable** |
| --- | --- | --- |
| 1 | Docker sandbox + MCP tool server | MCP server exposing shell, pwntools, ghidra, gdb, tshark as callable tools |
| 2 | Auto-prompter + Planner agent | Agent that classifies challenges and builds task trees |
| 3 | Executor A (web/crypto/forensics) | ReAct loop with env verification gate, category-specific prompts |
| 4 | Executor B (pwn/rev) | Same as above, different toolset + prompts |
| 5 | RAG engine | HyDE query builder + hybrid retrieval + CRAG correction + reranker |
| 6 | Knowledge base ingestion | Scraper + chunker + HyPE indexer + ChromaDB setup |

# **4. Module specs**

## **Module 1 - Docker sandbox + MCP tool server**

Owner: Person 1

### **What it is**

An MCP server that runs inside (or alongside) a Docker container. Every tool the agents can use is exposed as an MCP endpoint. This standardizes tool interfaces - agents call tools the same way regardless of what the tool is, and the server handles the actual subprocess calls.

### **Tools to expose**

| **Tool name** | **Wraps** | **Notes** |
| --- | --- | --- |
| shell | bash subprocess | Stateful session - commands persist within a challenge |
| python\_exec | Python REPL | Pre-installed: pwntools, pycryptodome, sage, sympy, mpmath |
| connect | pwncat / nc | Interactive server connection tool - essential for pwn |
| debug | gdb + peda/pwndbg | Interactive debugger session |
| decompile | ghidra headless | Returns decompiled function or file |
| binary\_tools | binwalk, strings, file, objdump | Static analysis shortcuts |
| network\_capture | tshark | Parse pcap files |
| submit\_flag | CTF platform API or local checker | Returns correct/incorrect |

### **Interface contract (input/output)**

All tools take a JSON input with at minimum: { tool: string, args: object, session\_id: string }

All tools return: { stdout: string, stderr: string, exit\_code: int, session\_id: string }

Interactive tools (connect, debug) additionally support a send field for follow-up inputs within the same session.

### **Env verification gate**

Every tool response goes through a simple verifier before being passed to the agent. The verifier checks: does the response look like real output (non-empty, not obviously hallucinated, exit code makes sense for the command)? If not, it flags it. The executor then re-runs the command rather than proceeding on bad data.

### **Pre-installed tools in Docker**

Don't let agents waste turns installing tools. Pre-install everything in the Dockerfile:

* Python: pwntools, pycryptodome, sage, sympy, mpmath, requests, beautifulsoup4
* System: gdb, peda, pwndbg, binwalk, strings, objdump, file, tshark, wine, wine32
* CTF-specific: RsaCtfTool, ghidra (headless), john, hashcat

## **Module 2 - Auto-prompter + Planner**

Owner: Person 2

### **Auto-prompter**

Runs first. Gets the raw challenge input (description text, any attached files, docker image name). Does the following:

* Reads challenge description and any hint files
* Probes the docker env with a few lightweight commands (ls, file \*, netstat)
* Classifies the challenge category with confidence score
* Writes a seed context block: category, key observations, likely attack surface

Output: structured JSON context block handed to the planner.

### **Planner**

Takes seed context + challenge input. Builds an initial task tree. Task tree is a list of subtasks with: id, description, category (retrieval needed: yes/no), depends\_on, status.

Planner runs a monitoring loop: every time an executor reports back, it updates task status and decides what to assign next. If an executor is stuck (3+ failed attempts on same subtask), planner can reassign or restructure the subtask.

### **Interface contract**

Input to auto-prompter: { description: string, files: [paths], docker\_image: string }

Output of auto-prompter / input to planner: { category: string, confidence: float, observations: [string], attack\_surface: string }

Planner output (task tree item): { id: string, description: string, needs\_retrieval: bool, executor: 'A'|'B'|null, depends\_on: [ids], status: 'pending'|'running'|'done'|'failed' }

## **Modules 3 & 4 - Executors**

Owners: Person 3 (web/crypto/forensics), Person 4 (pwn/rev)

### **Shared design**

Both executors run the same ReAct loop. The difference is their system prompt, which is category-tuned, and which tools they primarily use.

ReAct loop: Thought -> Action (tool call via MCP) -> Observation (verified env output) -> repeat until flag found or give\_up.

### **Executor A - web / crypto / forensics**

* Primary tools: shell, python\_exec, network\_capture, binary\_tools
* System prompt tuned for: XSS, SQLi, SSRF, LFI, JWT attacks, classical crypto, hash cracking, file carving, steganography
* Gets RAG hints pre-injected for crypto and forensics (web usually benefits less from retrieval)

### **Executor B - pwn / rev**

* Primary tools: shell, python\_exec, debug, decompile, binary\_tools, connect
* System prompt tuned for: buffer overflows, format strings, heap exploits, ROP chains, reverse engineering binaries
* Gets RAG hints pre-injected always - pwn and rev benefit most from similar writeups

### **Env verification gate (both executors)**

After every tool call, before processing the observation:

* Check exit\_code is expected for the command run
* Check stdout is non-empty if output was expected
* If mismatch, re-run the tool once. If still wrong, report back to planner as failed step.

This prevents the agent from hallucinating tool output and making decisions based on fake observations.

### **Interface contract**

Input: { task: TaskTreeItem, hints: [string], session\_id: string }

Output: { task\_id: string, status: 'done'|'failed'|'needs\_help', flag: string|null, summary: string, steps\_taken: [string] }

## **Module 5 - RAG engine**

Owner: Person 5

### **Pipeline**

Four stages as described in section 2.3. The engine is a standalone service that executors call when they need knowledge. It is not embedded in the executor loop directly.

| **Stage** | **Method** | **Library** |
| --- | --- | --- |
| Query formation | HyDE - generate hypothetical writeup then embed it. Also generate 3 query rephrasings for multi-query expansion | LangChain HyDE, or manual |
| Retrieval | BM25 + dense in parallel, fused with RRF. Fetch top 100 candidates total | ChromaDB (dense), BM25s or rank\_bm25 (sparse) |
| Reranking | Cross-encoder reranks top 50 from RRF output | sentence-transformers cross-encoder/ms-marco-MiniLM-L-6-v2 |
| Correction (CRAG) | Grade each retrieved chunk individually. If score < 0.5, rewrite query and retry or fall back to web search (Serper/Google API) | Custom grader or use LLM-as-judge |

### **Adaptive routing**

Before starting retrieval, check the task needs\_retrieval flag from the task tree. If false, return empty hints immediately. If true, also check a simple heuristic: if the task description is under 15 words and looks like a coding task, skip retrieval. This saves ~0.5-1s per easy subtask.

### **Interface contract**

Input: { query: string, category: string, top\_k: int = 5 }

Output: { hints: [string], sources: [string], retrieval\_score: float, web\_fallback\_used: bool }

## **Module 6 - Knowledge base ingestion**

Owner: Person 6

### **Sources to ingest**

| **Source** | **How to get it** | **Priority** |
| --- | --- | --- |
| CTFtime writeups | Scrape CTFtime.org writeup links. Many link to GitHub - fetch those too | High |
| GitHub CTF writeup repos | Search GitHub for '\*-ctf-writeups' repos. Clone and parse markdown | High |
| PayloadsAllTheThings | Clone the repo, parse by category folder | High |
| CVE data | NVD JSON feeds (free). Parse description + affected versions | Medium |
| Personal writeups | Any writeups team members have from past CTFs | Medium |

### **Chunking strategy**

Don't chunk blindly by character count. Use semantic chunking:

* Writeups: chunk by section header (## Approach, ## Solution, ## Payload, etc.)
* PayloadsAllTheThings: chunk by individual payload entry
* CVE: each CVE is one chunk

Target chunk size: 300-600 tokens. Overlap: 50 tokens between adjacent chunks from the same writeup.

### **HyPE indexing (at ingest time)**

For each chunk, call the LLM to generate 3-5 questions that chunk could answer. Store those question embeddings in addition to the chunk embedding. This improves retrieval recall by ~40% at no query-time cost.

Example for a chunk about SQL injection:

* 'How do I bypass login with SQL injection?'
* 'What payload breaks out of a single-quote string context?'
* 'How to exfiltrate data via UNION SELECT in a CTF?'

### **Storage**

ChromaDB for vector storage (local, easy to run). BM25 index built from the same chunks using rank\_bm25. Both indexed by the same chunk IDs so results can be fused.

### **Interface contract**

The ingestion pipeline is offline (runs once, or on update). The RAG engine queries ChromaDB and the BM25 index directly. No API needed between modules 5 and 6.

# **5. Inter-module interfaces**

Quick reference for what each module sends and receives. Keep these stable - if you need to change an interface, tell everyone first.

| **From** | **To** | **Payload** |
| --- | --- | --- |
| Challenge input | Auto-prompter | { description, files[], docker\_image } |
| Auto-prompter | Planner | { category, confidence, observations[], attack\_surface } |
| Planner | Executor A or B | { task (TaskTreeItem), session\_id } |
| Executor A/B | RAG engine | { query, category, top\_k } |
| RAG engine | Executor A/B | { hints[], sources[], retrieval\_score, web\_fallback\_used } |
| Executor A/B | MCP tool server | { tool, args, session\_id } |
| MCP tool server | Executor A/B | { stdout, stderr, exit\_code, session\_id } |
| Executor A/B | Planner | { task\_id, status, flag|null, summary, steps\_taken[] } |
| Planner | Output | { flag, solved: bool, total\_cost, steps\_taken[] } |

# **6. Tech stack**

| **Component** | **Choice** | **Why** |
| --- | --- | --- |
| LLM | Some Open Source Model | free cost high parameter good benchmark model |
| Agent framework | Custom ReAct loop in Python | Lightweight. Frameworks like LangGraph add complexity we don't need yet |
| MCP server | Python MCP SDK (mcp package) | Official SDK, easy to extend |
| Vector DB | ChromaDB | Local, no infra, easy to reset between experiments |
| BM25 | rank\_bm25 or BM25s | Simple, fast, no server needed |
| Reranker | cross-encoder/ms-marco-MiniLM-L-6-v2 | Good quality, runs on CPU |
| Docker | Docker + docker-py | Sandboxed tool execution, matches NYU CTF Bench setup |
| Web search fallback | Serper API or SerpAPI | Cheap, simple JSON API |
| Ingestion scraper | requests + beautifulsoup4 + gitpython | Standard Python scraping |

# **7. What not to build (yet)**

Things that sound good but aren't worth the time for v1:

* GraphRAG / LightRAG - graph DB adds infra overhead. Hybrid retrieval with reranker gets most of the benefit
* Fine-tuning - needs GPU hours and weeks of data prep. RAG gives us updatable knowledge without training
* Three executors - two is enough. The third doesn't add meaningful parallelism for most CTF challenges
* A web UI - CLI is fine, focus on solving challenges correctly
* Automated writeup generation post-solve - nice to have, not core
* HITL (human in the loop) integration - optional for competition, build it after v1 works
