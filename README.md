
# Architecture Context Analyzer

> **Work in progress:** A Python static-analysis and architecture-recovery tool for understanding, untangling, and improving existing codebases — with a particular focus on producing concise architectural context for humans and AI coding agents.

## Why this exists

Large codebases rarely reflect a perfectly intentional architecture.

As systems evolve, responsibilities move, abstractions overlap, dependencies accumulate, and implementation decisions become embedded in the structure of the repository. The resulting architecture may still work, but understanding **where functionality belongs**, **what can safely change**, and **how components should be separated** becomes increasingly difficult.

This problem is amplified by AI-assisted development.

Coding agents operate with limited context. Giving an agent more source files does not necessarily give it a better understanding of the system. In a poorly separated codebase, limited context can encourage an agent to:

* create functionality that already exists elsewhere;
* add abstractions instead of reusing existing ones;
* introduce reverse or circular dependencies;
* place functionality in a convenient but architecturally inappropriate module;
* preserve accidental architecture because it appears to be intentional;
* increase coupling while solving a locally reasonable problem.

This project explores a different approach:

> **Analyze the architecture first, identify meaningful boundaries and structural problems, and provide only the architectural context needed to make a particular change well.**

The goal is to help developers — human or AI — **make more informed changes with less context**.

---

# Project goals

The project is intended to evolve from a simple dependency analyzer into an **architecture reasoning tool**.

It should help answer questions such as:

* What depends on what?
* Where are the circular dependencies?
* Which modules have unusually high coupling?
* Where are dependencies flowing in both directions?
* Which parts of the repository form cohesive functional groups?
* Where do existing package boundaries disagree with actual dependency structure?
* Which dependencies appear to cross otherwise clean architectural boundaries?
* What modules would be affected by a proposed change?
* Where are the best candidate seams for separating responsibilities?
* What would happen if a module or responsibility were moved?
* What architectural context does a developer or AI agent actually need for a specific task?

The emphasis is not merely on detecting "bad code."

The larger goal is to help recover the **conceptual architecture hiding inside an imperfect implementation**.

---

# Core idea

The tool distinguishes between several related concepts.

## Observed architecture

What the source code currently does.

This can be derived from evidence such as:

* imports;
* modules and packages;
* function and class relationships;
* inheritance;
* calls;
* filesystem structure;
* test relationships;
* configuration;
* Git change history.

For example:

```text
dags.assets
    ↓
bps_monthly_dashboard.transform
    ↓
bps_monthly_dashboard.database
    ↑
dags.storage
```

## Architectural findings

Structural properties that can be derived from the observed graph.

Examples include:

* dependency cycles;
* high fan-in or fan-out;
* bidirectional package coupling;
* unstable dependency direction;
* unexpectedly central modules;
* cross-boundary dependencies;
* modules that participate in many unrelated dependency paths.

Some findings are objective facts.

Others are deliberately opinionated heuristics.

The tool should distinguish between them.

## Candidate architecture

The architecture suggested by the evidence rather than necessarily represented by the current directory structure.

For example:

```text
Orchestration
      ↓
Domain / Transformation
      ↓
Infrastructure
```

A single dependency running against this otherwise consistent direction may indicate a useful place to investigate.

Candidate architecture is a hypothesis, not ground truth.

The tool should help users reason about architectural alternatives rather than pretend that one decomposition is universally correct.

---

# Design philosophy

## Analyze source without executing it

The initial analyzer should operate directly on a source tree.

A project should not need to:

* successfully start;
* have credentials;
* connect to databases;
* have all optional dependencies installed;
* initialize application frameworks;
* execute imports with side effects.

Where practical:

```text
repository
    ↓
static analysis
    ↓
architecture model
```

should be enough.

---

## Start useful with zero configuration

The basic workflow should eventually be:

```bash
archtool .
```

and produce useful output.

Configuration should make the analysis **better or stricter**, not make the tool usable for the first time.

---

## Explain findings instead of assigning arbitrary scores

A module with many dependencies is not automatically badly designed.

A highly connected module may be:

* a legitimate orchestration component;
* a stable shared abstraction;
* an accidental god module;
* a missing architectural boundary.

Instead of simply reporting:

```text
Coupling score: 72
```

prefer:

```text
High outgoing coupling

dags.assets.publish imports 18 internal modules.

Repository median: 3

This may be expected for an orchestration module, but it also
makes the module sensitive to changes across several otherwise
independent parts of the application.
```

The tool should expose evidence and help the user decide what it means.

---

## Separate facts, heuristics, and rules

### Facts

Objective properties of the dependency model.

```text
A → B → C → A
```

is a circular dependency.

### Heuristics

Potential architectural concerns.

```text
Module A imports substantially more independent components
than similar modules in the repository.
```

This deserves investigation but is not necessarily wrong.

### Rules

Architecture explicitly chosen by the project.

```text
domain must not import orchestration
```

A rule violation can be treated as an error because the project has declared the constraint.

---

## Prefer progressive disclosure

A 400-node dependency graph is rarely the best explanation of a system.

The UI should allow architecture to be explored at different resolutions:

```text
Repository
    ↓
Package
    ↓
Subpackage
    ↓
Module
    ↓
Symbol
```

Users should be able to move between a high-level system view and the local neighborhood of a particular module without losing context.

---

## Optimize for architectural context, not maximum context

An AI coding agent should not necessarily receive every related source file.

Instead, the tool should eventually be capable of producing something like:

```text
Target:
    bps_monthly_dashboard.oracle_sync

Architectural role:
    Oracle persistence / synchronization

Direct dependencies:
    database
    schema
    adapters.oracle

Direct dependents:
    sync

Relevant architectural boundary:
    orchestration → domain → infrastructure

Constraint:
    bps_monthly_dashboard currently has no dependency on dags.

Potential concern:
    introducing a dags import here would reverse the existing
    dependency direction.

Relevant files:
    oracle_sync.py
    sync.py
    adapters/oracle.py
    schema.py
```

This should require dramatically less context than providing the entire surrounding repository.

---

# Initial scope

The first releases will focus on **Python module and package architecture**.

The immediate model is based primarily on static import relationships.

Given:

```python
from bps_monthly_dashboard.settings import get_path_settings
from bps_monthly_dashboard.transform_batch import write_csv_output

from dags.file_metadata import source_period_dimensions
from dags.snapshot_storage import bps_data_root
```

the analyzer should recognize local dependencies such as:

```text
current_module
    ├── bps_monthly_dashboard.settings
    ├── bps_monthly_dashboard.transform_batch
    ├── dags.file_metadata
    └── dags.snapshot_storage
```

without treating third-party imports such as:

```python
import pandas
import dagster
```

as internal repository dependencies.

---

# Initial architecture

The project should keep analysis separate from presentation.

```text
Python source
     │
     ▼
┌───────────────────────┐
│ Source discovery      │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Static parser         │
│ import resolution     │
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Architecture graph    │
└───────────┬───────────┘
            │
      ┌─────┼──────────────┐
      │     │              │
      ▼     ▼              ▼
  analysis  DOT        interactive UI
      │
      ▼
 architecture
 findings
      │
      ▼
 agent context
```

A possible package structure:

```text
src/
    archtool/
        discovery/
            modules.py
            packages.py

        parsing/
            imports.py
            resolver.py

        graph/
            model.py
            algorithms.py

        analysis/
            cycles.py
            coupling.py
            boundaries.py
            clustering.py

        context/
            neighborhood.py
            briefing.py

        output/
            console.py
            dot.py
            json.py

        ui/
            app.py

        cli.py
```

The exact structure is expected to evolve.

---

# Dependency model

The first internal model will likely represent dependencies independently from any particular renderer.

For example:

```python
@dataclass(frozen=True)
class ModuleDependency:
    source: str
    target: str
    source_file: Path
    line_number: int | None
```

The graph layer can then support operations such as:

```python
graph.dependencies_of(module)

graph.dependents_of(module)

graph.shortest_path(source, target)

graph.cycles()

graph.neighborhood(module, depth=2)

graph.collapse(depth=2)
```

Graphviz, Streamlit, CLI output, JSON exports, and future agent interfaces should all consume the same underlying model.

---

# Planned analysis

## Dependency cycles

Circular dependencies will be detected using strongly connected components rather than reporting every possible circular path independently.

Example:

```text
storage
   ↓
assets
   ↓
metadata
   ↓
storage
```

should be treated as one architectural knot that can then be explored in detail.

---

## Fan-in and fan-out

For each module:

```text
fan-in
    how many modules depend on this module

fan-out
    how many internal modules this module depends on
```

These values are evidence, not automatic quality judgments.

---

## Package coupling

Dependencies should be aggregatable above the module level.

For example:

```text
dags → bps_monthly_dashboard
31 dependencies

bps_monthly_dashboard → dags
1 dependency
```

That one reverse dependency may be considerably more architecturally interesting than the individual imports involved.

---

## Dependency direction

The analyzer should make it easy to see whether dependency flow is primarily:

```text
orchestration
    ↓
domain
    ↓
infrastructure
```

or:

```text
orchestration
    ↕
domain
    ↕
infrastructure
```

Bidirectional dependencies are not automatically wrong, but they are useful places to investigate.

---

## Architectural neighborhoods

For a selected module, the system should be able to derive a focused subgraph:

```text
target
├── direct dependencies
├── direct dependents
├── important transitive dependencies
├── cycles
└── relevant package boundaries
```

This will eventually form the basis of task-specific AI context.

---

# Architecture recovery

A major long-term goal is to move beyond describing the existing package structure.

The analyzer should look for evidence of **natural architectural seams**.

Possible evidence may include:

* high internal dependency density;
* low external dependency density;
* shared callers;
* shared dependencies;
* semantic similarity;
* naming conventions;
* directory relationships;
* Git change coupling;
* test relationships;
* consistent dependency direction.

For example, analysis might suggest:

```text
Candidate component: County Resolution

county_lookup.py
geography.py
place_lookup.py
county_names.py

Evidence:
    strong internal relationships
    few external entry points
    shared terminology
    frequent co-change
```

The goal is not to automatically declare this the correct architecture.

The goal is to identify a plausible boundary worth investigating.

---

# Architectural seams

A **seam** is a candidate place where responsibilities can be separated behind a clearer interface.

Future analysis may identify:

```text
┌──────────────────────┐
│ County Resolution    │
│                      │
│ county_lookup        │
│ geography            │
│ county_names         │
└──────────┬───────────┘
           │
     narrow interface
           │
           ▼
┌──────────────────────┐
│ Transformation       │
│                      │
│ transform_batch      │
│ postprocess          │
└──────────────────────┘
```

The tool should eventually help answer:

* What dependencies currently cross this boundary?
* Which should remain?
* Which appear to expose implementation details?
* What would need to move?
* What would break?
* Would this remove cycles?
* Would coupling improve?
* What might an appropriate public interface look like?

---

# Refactoring analysis

A longer-term goal is to support **what-if architecture analysis**.

For example:

```text
Move:
    oracle_sync.py

From:
    bps_monthly_dashboard

To:
    adapters.oracle
```

The analyzer could estimate:

```text
Cross-boundary dependencies
    before: 14
    after:   8

Cycles
    before: 1
    after:  0

Callers requiring updates
    sync.py
    cli.py

New dependency inversions
    none
```

This allows a developer to reason about a structural change before editing the source tree.

---

# Streamlit UI

An interactive Streamlit application is planned as a primary exploration interface.

The UI should **not** begin with an enormous dependency graph.

Instead, it should summarize architectural information and allow progressive exploration.

Potential landing view:

```text
Architecture
────────────────────────────────────

168 modules
412 internal dependencies
4 top-level packages
2 dependency cycles

Potential concerns
────────────────────────────────────

2 dependency cycles
3 highly coupled modules
1 reverse package dependency
4 modules with rapidly increasing coupling

[Overview] [Packages] [Cycles] [Coupling] [Graph]
```

The graph view should support:

* package collapsing;
* expansion by hierarchy;
* filtering;
* dependency direction;
* incoming/outgoing dependencies;
* focused neighborhoods;
* highlighting cycles;
* highlighting boundary crossings;
* selecting a module for analysis;
* exporting the current view.

---

# Graphviz export

Graphviz DOT will remain a supported output format.

Example:

```dot
subgraph cluster_dags {
    label="dags";

    dags__file_metadata [label="file_metadata"];
    dags__snapshot_storage [label="snapshot_storage"];
}

subgraph cluster_bps {
    label="bps_monthly_dashboard";

    bps__settings [label="settings"];
    bps__transform_batch [label="transform_batch"];
}

dags__enrichment_base -> {
    bps__settings
    bps__transform_batch
    dags__file_metadata
    dags__snapshot_storage
}
```

Graphviz provides a portable representation useful for:

* documentation;
* static exports;
* architecture reviews;
* repository artifacts;
* CI output.

---

# AI and agent context

AI integration is a long-term goal, but the architecture model should remain deterministic and useful without an LLM.

The initial goal is to generate compact architectural briefings.

For example:

```bash
archtool context src/package/export.py
```

might eventually produce:

```text
Target
    package.export

Role
    Builds external data exports.

Direct dependencies
    database
    schema
    settings

Direct dependents
    cli
    publish

Architectural observations
    export is currently outside all detected cycles.
    database is a highly shared dependency.
    publish is part of the orchestration layer.

Potential constraint
    lower-level persistence modules currently do not import publish.

Relevant neighborhood
    export.py
    database.py
    schema.py
    publish.py
```

Future interfaces may include:

* Markdown context;
* JSON;
* command-line queries;
* MCP;
* integrations with coding agents.

A token budget may eventually control how much architectural detail is returned:

```bash
archtool context export.py --budget 1500
```

The central question is:

> **What is the minimum architectural context required to make this change coherently?**

---

# Architecture rules

Explicit architecture enforcement may eventually be supported.

For example:

```toml
[[tool.archtool.forbid]]
from = "domain"
to = "orchestration"

[[tool.archtool.layers]]
modules = [
    "orchestration",
    "application",
    "domain",
    "infrastructure",
]
```

These rules are intentionally distinct from inferred architectural findings.

An inferred boundary is evidence.

A configured rule is a project decision.

---

# Architecture drift

Long-term analysis may also compare architecture over time.

For example:

```text
Architecture changes

+ publish → oracle_sync
+ publish → snapshot_storage

New cycle:
    publish
      → oracle_sync
      → publish

Fan-out:
    publish
    11 → 13
```

This may allow CI to distinguish between:

```text
existing architectural debt
```

and:

```text
new architectural debt introduced by this change
```

so adoption does not require immediately repairing every historical problem.

---

# Roadmap

## v0.1 — Observe

Goal:

> Build a trustworthy local dependency model.

Planned:

* recursive Python source discovery;
* local module identification;
* AST-based import extraction;
* relative import resolution;
* internal vs external dependency classification;
* module dependency graph;
* package clustering;
* Graphviz DOT export;
* basic CLI;
* automated tests.

Primary question:

> **What depends on what?**

---

## v0.2 — Analyze

Planned:

* strongly connected components;
* circular dependency reporting;
* fan-in;
* fan-out;
* package-level aggregation;
* shortest dependency paths;
* reverse dependency queries;
* focused architectural neighborhoods.

Primary question:

> **Where are the structural pressure points?**

---

## v0.3 — Explore

Planned:

* Streamlit application;
* package/module navigation;
* expand/collapse hierarchy;
* filtered graph visualization;
* cycle highlighting;
* coupling views;
* module inspection;
* DOT export from selected views.

Primary question:

> **How can I understand this architecture without staring at the entire graph?**

---

## v0.4 — Reason

Planned research:

* architectural boundary detection;
* structural clustering;
* dependency-direction anomalies;
* candidate seams;
* cross-boundary coupling;
* semantic relationships;
* Git change coupling.

Primary question:

> **Where might this codebase be naturally separable?**

---

## v0.5 — Refactor

Planned:

* proposed module moves;
* simulated boundary changes;
* dependency impact analysis;
* before/after coupling;
* refactoring sequence generation;
* architecture baselines and diffs.

Primary question:

> **How can this structure be improved safely?**

---

## v0.6 — Context

Planned:

* compact architecture briefings;
* task-specific neighborhoods;
* configurable context budgets;
* Markdown and JSON outputs;
* agent-oriented interfaces;
* possible MCP server.

Primary question:

> **What does an AI agent actually need to know before making this change?**

---

# Non-goals

At least initially, this project is **not** intended to be:

* a general-purpose Python linter;
* a type checker;
* a security scanner;
* a replacement for tests;
* a full control-flow analyzer;
* an automatic refactoring engine;
* an autonomous architecture designer;
* a universal code-quality scoring system.

The initial domain is narrower:

> **software structure, dependency relationships, architectural boundaries, and refactoring context.**

---

# Technical direction

Likely early technologies include:

* Python;
* `ast` for initial parsing;
* NetworkX for graph algorithms;
* Graphviz for static graph export;
* Streamlit for interactive exploration;
* `pytest` for testing.

Tree-sitter may be evaluated later if symbol-level analysis, incomplete-code parsing, incremental analysis, or additional language support makes it worthwhile.

The project should avoid reimplementing mature graph algorithms when established libraries provide them.

Engineering effort should concentrate on the harder question:

> **What does the graph tell us about the architecture?**

---

# Current status

**Early prototype / research phase.**

The current implementation goal is intentionally small:

```text
Python repository
      ↓
discover modules
      ↓
parse local imports
      ↓
dependency graph
      ↓
Graphviz DOT
```

This establishes the structural model required for later analysis.

---

# Guiding questions

As the project develops, new features should be evaluated against a few recurring questions:

1. Does this help reveal architecture rather than merely describe source files?
2. Does this help distinguish intentional coupling from accidental coupling?
3. Does this help identify a cleaner functional boundary?
4. Does this reduce the amount of code a developer must inspect?
5. Does this reduce the amount of context an AI agent needs?
6. Does this make refactoring decisions easier to explain?
7. Can the result be supported by observable evidence?
8. Does the tool preserve human judgment where architectural intent is ambiguous?

If a feature does not help answer one of those questions, it may not belong in the project.

---

# Project hypothesis

This project is ultimately testing a broader idea:

> **AI-assisted software development becomes more effective when agents receive a compact model of architectural intent and structural constraints rather than simply receiving more source code.**

And, before that can happen:

> **Messy codebases need tools that help recover and improve their architecture instead of assuming the current structure is already the right one.**

# sources:
https://www.sciencedirect.com/science/article/pii/S095058492300174X
https://www.sciencedirect.com/science/article/abs/pii/S0950584920302147
https://www.sciencedirect.com/org/science/article/pii/S194781862200045X
https://www.lattix.com/solutions/refactoring/
