---
title: "[Data Engineering] - An Evolving Role Beyond BI and Classical ML"
layout: post
---

# Data Engineering for Agentic Systems/LLM vs Traditional ML/AI and BI

Agentic systems represent a **fundamental shift** in how data engineering is practiced. This is not just *LLMs + tools* — it is a move toward **systems that plan, act, observe, and learn autonomously**.

This document compares **Data Engineering across four eras**:
- BI
- Classical ML / AI
- **LLM Applications** (RAG, etc)
- **Agentic Systems**

But first, this is where most organizations actually are:

**Dominant maturity level: BI-first, ML-adjacent**

Most companies have strong capabilities in:
* Data warehouses / lakehouses
* Batch ETL (Airflow, dbt, Spark)
* Star schemas, facts & dimensions
* Dashboards, KPIs, reporting
* Basic governance & access control

Some have:
* Classical ML pipelines (features → models → predictions)
* MLOps for a small number of models

Very few have:

* Document-centric data platforms
* Real-time knowledge ingestion
* Memory systems
* Retrieval observability
* Agent safety pipelines

---

## Summary

- **BI / Classical ML** → Data supports *analysis & prediction*
- **LLM applications (RAG)** → Data supports *reasoning & answering*
- **Agentic systems** → Data supports *planning, acting, and learning*

> This shift fundamentally changes what data engineering optimizes for.

---

## 2. High-level Comparison

| Dimension | BI | Classical ML / AI | LLM Apps (RAG) | **Agentic Systems** |
|--------|----|------------------|---------------|--------------------|
| Core goal | Reporting | Prediction | Answering | **Autonomous task completion** |
| Data consumer | Humans | Models | LLMs | **LLMs + tools + policies** |
| Data type | Structured | Structured | Unstructured text | **Stateful, event-driven, multi-modal** |
| Data lifecycle | Batch | Batch / offline | Near real-time | **Continuous, loop-based** |
| Output risk | Wrong metric | Wrong prediction | Hallucination | **Runaway actions / compounding errors** |
| Feedback loop | Slow | Slow | Fast | **Immediate & recursive** |

---

## 3. What Fundamentally Changes for Data Engineering

### 3.1 From Static Data to **Agent State**

**Before**
- Tables, features, documents
- Mostly immutable

**Agentic systems**
- Short-term memory (working context)
- Long-term memory (knowledge & experience)
- Episodic memory (past actions & outcomes)
- External state (tools, APIs, environments)

> You are engineering **memory systems**, not datasets.

---

### 3.2 ETL → **Sense → Think → Act → Reflect Pipelines**

**Classical pipeline**

> ETL → Model → Output


**Agentic pipeline**

> Observe → Retrieve → Plan → Act → Log → Reflect → Update Memory


Data engineers must design and operate:
- Action logs
- Decision traces
- Tool outputs
- Reflection artifacts

---

### 3.3 Feature Engineering → **World Modeling**

**Traditional ML**
- Features approximate reality statistically

**Agentic systems**
- Agents require:
  - Entity representations
  - Constraints
  - Capabilities
  - Dependencies
  - Temporal relationships

Often implemented with:
- Lightweight, dynamic knowledge graphs
- Structured tool schemas
- State machines
- Event sourcing patterns

---

### 3.4 Data Quality Becomes **Safety & Controllability**

New failure modes:
- Feedback loops reinforcing bad behavior
- Partial or corrupted state causing incorrect plans
- Stale memory overriding fresh reality
- Tool misuse due to incorrect metadata

Data quality now includes:
- Temporal correctness
- Source authority ranking
- Confidence scoring
- Conflict resolution rules
- Expiration and decay policies

---

## 4. Governance & Compliance Escalate Sharply

| Area | Traditional Systems | Agentic Systems |
|----|--------------------|----------------|
| Access control | Table / column | **Action & capability level** |
| Lineage | Data lineage | **Decision & action lineage** |
| Auditing | Queries | **Why this action happened** |
| Rollback | Re-run job | **Undo / compensate actions** |

You must be able to answer:
> *"Why did the agent do this, using that data, at that time?"*

---

## 5. Observability Is Now First-Class Data Engineering

**Traditional metrics**
- Data freshness
- Accuracy
- Drift

**Agentic metrics**
- Plan success rate
- Tool-call accuracy
- Action reversibility
- Memory relevance
- Error compounding depth

> Logs are **training data for behavior**, not just debugging artifacts.

---

## 6. Data Schemas Become **Behavioral Contracts**

Instead of schemas that describe rows, you define:
- Tool input/output contracts
- Allowed action spaces
- Preconditions & postconditions
- Guardrails encoded as data

Bad schemas don’t just break pipelines —  
**they change agent behavior**.

---

## 7. What Matters Less Than Before

- Large historical datasets
- Deep offline training cycles
- Static feature stores
- Perfect normalization

---

## 8. What Matters More Than Ever

- State consistency
- Memory scoping
- Temporal ordering
- Explicit constraints
- Safe defaults
- Human-in-the-loop correction data

---

## 9. The Biggest Mindset Shift for Data Engineers

> **You are no longer supporting intelligence, 
you are shaping behavior.**

Agentic data engineering is closer to:
- Distributed systems
- Control theory
- Workflow orchestration
- Safety engineering

…than to classical analytics.

---

## 10. Final Summary

| Era | Data Engineering Focus |
|----|------------------------|
| BI | Metrics & truth |
| Classical ML | Signals & prediction |
| LLM applications | Context & grounding |
| **Agentic systems** | **State, behavior, and safety** |




