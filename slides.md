---
theme: default
title: Distributed Spatial Database & Autonomous MCP Compilation
info: |
  Master's Thesis Pitch — Capstone Research Proposal
  Distributed PM Quadtree · MASS Agents · MCP Streaming
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# Distributed Spatial Database<br/>and Autonomous MCP Compilation

Master's Thesis Pitch

<div class="pt-12 text-sm opacity-75">
  A scientific inquiry into distributed spatial indexing and LLM-driven map compilation
</div>

---
layout: default
---

# Research Agenda

<div class="grid grid-cols-2 gap-8 mt-8 text-left text-sm">

<div>

### Core Question

Can **distributed PM Quadtrees** with **MASS agent migration** eliminate the memory and I/O bottlenecks of monolithic spatial databases — and can an **MCP server** expose that engine to an autonomous LLM for real-time map compilation?

</div>

<div>

### Today's Narrative

1. The Scientific Gap
2. The Hypothesis
3. Phase 1 — Distributed Engine
4. MASS Lifecycle Animation
5. Phase 2 — Autonomous MCP Pipeline
6. Evaluation Methodology
7. Next Steps

</div>

</div>

---
layout: two-cols
---

# The Scientific Gap

::left::

### Prior Lab Research

- Spatial data loaded from **monolithic files** (GeoJSON, shapefiles)
- **Point-Region (PR) Quadtrees** index discrete points well
- Continuous geometry — **roads, corridors, polygons** — breaks the model

::right::

### Observed Limitations

| Limitation | Impact |
|------------|--------|
| Memory ceiling | Full dataset must fit in RAM |
| I/O bottleneck | Sequential file reads block insertion |
| PR tree mismatch | Lines span cells; poor subdivision |
| No distribution | Single-node scaling wall |

<div class="mt-4 text-sm opacity-80">

**Research gap:** No distributed spatial index designed for *continuous* geometry with agent-based routing across nodes.

</div>

---
layout: center
class: text-left
---

# The Hypothesis

<div class="max-w-3xl mx-auto space-y-6 text-lg">

**H₁:** A **Polygonal Map (PM) Quadtree** — subdividing only where geometry intersects — combined with **MASS Agents** as migratory network routers will reduce memory overhead and I/O latency for continuous spatial insertions.

**H₂:** An **MCP Server** exposing spatial tool calls to a local LLM agent, fed by a **Video ETL pipeline**, will enable autonomous, streaming compilation of distributed map data.

</div>

<div class="mt-12 flex justify-center gap-12 text-sm opacity-75">

<div>PM Quadtree → precise subdivision</div>
<div>MASS Agents → distributed routing</div>
<div>MCP Streaming → autonomous compilation</div>

</div>

---
layout: default
---

# Phase 1: The Distributed Engine

<div class="grid grid-cols-1 md:grid-cols-2 gap-6 mt-4">

<div class="text-sm text-left space-y-3">

### Architecture

- **PM Quadtree** on the MASS framework
- MASS Agents act as **distributed network routers**
- Agents **migrate across nodes** to execute spatial insertions
- Targets continuous geometry: roads, corridors, boundaries

### Why PM over PR?

PR trees assume point data. PM trees subdivide cells **only where a polygon or line intersects**, preserving spatial locality for continuous features.

</div>

<div>

<QuadtreeDemo />

<div class="text-xs opacity-60 mt-1">On iPhone: tap green button → fullscreen draw mode. Desktop: click &amp; drag.</div>

</div>

</div>

---
layout: default
---

# MASS Lifecycle: Distributed Subdivision

<div class="grid grid-cols-2 gap-6 mt-2">

<div class="text-sm text-left space-y-3">

### Synchronized Cluster Boot

All processes in a MASS computation call **`MASS.init()`** and **`MASS.finish()`** together — starting and stopping as one coordinated unit.

### Per-Node Parallelism

On `init()`, each process (node) spawns **one thread per local CPU core**. All threads access shared **Places** (PM cells) and **Agents** (migratory routers).

### Agent-Routed Division

An **Agent migrates** to the node owning the target Place, executes PM subdivision where the road intersects, then the cluster calls **`MASS.finish()`** to detach threads and clean up.

</div>

<div>

<DemoIframe demo="mass-division-demo.html" height="360px" />

<div class="text-xs opacity-60 mt-1">Animated: init → thread spawn → agent migration → subdivide → finish</div>

</div>

</div>

---
layout: default
---

# Phase 2: The Autonomous MCP Pipeline

<div class="grid grid-cols-1 md:grid-cols-2 gap-6 mt-4">

<div class="text-sm text-left space-y-3">

### Pipeline Flow

1. **Video ETL** extracts spatial context from visual input
2. Local **LLM agent** interprets context and plans insertions
3. **MCP Server** exposes distributed DB as spatial tools
4. Agent streams tool calls: `execute_insert`, `query_region`, …
5. Map compiles **dynamically** across the cluster

### MCP as the Bridge

The Model Context Protocol standardizes how the LLM discovers and invokes spatial operations — turning the distributed engine into a **tool-augmented reasoning surface**.

</div>

<div>

<DemoIframe demo="mcp-demo.html" height="340px" />

<div class="text-xs opacity-60 mt-1">Interactive: stream simulated MCP tool calls onto the grid</div>

</div>

</div>

---
layout: default
---

# Evaluation Methodology

<div class="grid grid-cols-3 gap-6 mt-8 text-sm text-left">

<div class="border border-gray-700 rounded-lg p-4">

### Memory Overhead

- Baseline: monolithic load + PR tree
- Treatment: distributed PM Quadtree
- Metric: peak RAM per node vs. dataset size

</div>

<div class="border border-gray-700 rounded-lg p-4">

### I/O Bottleneck

- Measure insertion throughput under concurrent writes
- Compare sequential file I/O vs. agent-routed distributed inserts
- Metric: inserts/sec at 95th-percentile latency

</div>

<div class="border border-gray-700 rounded-lg p-4">

### Agent Migration Latency

- Time for MASS Agent to migrate and execute insertion
- Sweep: cluster size, geometry complexity, network topology
- Metric: end-to-end insertion latency (p50, p99)

</div>

</div>

<div class="mt-8 text-sm opacity-80">

**Phase 2 metrics:** MCP tool-call round-trip latency, compilation accuracy vs. ground-truth geometry, and end-to-end Video ETL → map latency.

</div>

---
layout: center
class: text-center
---

# Q&A · Next Steps

<div class="mt-8 space-y-4 text-left max-w-2xl mx-auto text-sm">

- [ ] Implement PM Quadtree core on MASS with agent migration protocol
- [ ] Benchmark against monolithic PR-tree baseline from prior lab work
- [ ] Build MCP Server with spatial tool schema (`insert`, `query`, `subdivide`)
- [ ] Integrate Video ETL pipeline with local LLM agent
- [ ] End-to-end demo: video input → MCP stream → distributed map

</div>

<div class="mt-12 text-2xl opacity-60">

Questions?

</div>

<div class="abs-br m-6 text-sm opacity-50">
  Distributed Spatial Database · Autonomous MCP Compilation
</div>
