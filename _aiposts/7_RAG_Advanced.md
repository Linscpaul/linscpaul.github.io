---
title:  "[RAG Advanced] Building a Custom RAG Pipeline for Granular Performance Tuning"
layout: post
---
[last updated: 18 Dec 2025]

## Overview
This project features a high-performance RAG pipeline built from the ground up without high-level frameworks like LangChain or LlamaIndex. By "mimicking" these frameworks through custom logic, this implementation provides full control over every component, allowing for precise performance fine-tuning and the application of advanced optimization techniques.

## Core Pipeline Architecture
* **Standardized Output:** Implemented `Pydantic` classes to enforce schema validation and ensure consistent data flow across the pipeline.
* **Document Ingestion:** Developed custom connectors to fetch and process documents from local knowledge bases.
* **LLM-Powered Chunking:** Leveraged a `"Parser LLM"` to transform raw text into structured, semantically coherent objects defined by the Pydantic schema.
* **Vector Storage:** Integrated an `Encoder LLM` to transform chunks into embeddings, stored in a local vector database for high-dimensional retrieval.
* **Performance Evaluation:** Built an automated evaluation loop that benchmarks generated responses against a curated dataset of `test_questions` and `test_answers`.

## Advanced RAG Optimization Techniques
This implementation serves as a sandbox for R&D across the following advanced RAG strategies:
* **Semantic Chunking:** Moving beyond fixed-size windows to chunk data based on thematic shifts.
* **Encoder R&D:** Benchmarking various embedding models to find the optimal balance between latency and retrieval accuracy.
* **Context Engineering:** Refining prompts to include dynamic variables like current date, metadata, and conversation history.
* **LLM-Driven Pre-processing:** Using a separate LLM to clean and summarize text specifically for the encoding phase.
* **Query Rewriting:** Translating ambiguous user queries into optimized search terms for better vector matching.
* **Multi-Query Expansion:** Generating multiple search perspectives from a single prompt to increase retrieval recall.
* **LLM Re-ranking:** Applying a "Cross-Encoder" approach to filter and prioritize the most relevant results from initial retrieval.
* **Hierarchical Retrieval:** Navigating a parent-child document structure (summaries first, deep-dives second).
* **GraphRAG Integration:** Connecting retrieved documents via a knowledge graph to surface contextually related nodes.
* **Agentic RAG:** Orchestrating the pipeline with autonomous agents capable of using tools (e.g., SQL) and long-term memory.

---

## Step 1 - Create Pydantic Class for Standard Output Format

``` python
class Result(BaseModel):
   page_content: str
   metadata: dict

class Chunk(BaseModel):
   headline: str = Field(description="A brief heading for this chunk, typically a few words, that is most likely to be surfaced in a query")
   summary: str = Field(description="A few sentences summarizing the content of this chunk to answer common questions")
   original_text: str = Field(description="The original text of this chunk from the provided document, exactly as is, not changed in any way")

   def as_result(self, document):
       metadata = {"source":document["source"], "type": document["type"]}
       return Result(page_content=self.headline + "\n\n" + self.summary + "\n\n" +self.original_text, metadata=metadata)

class Chunks(BaseModel):
   chunks: list[Chunk]
```

## What this code is doing - Big picture
This code defines three data models:
* `Chunk` — a structured representation of a document chunk.
* `Result` — a retrieval-ready document (text + metadata).
*`Chunks` — a container holding multiple Chunk objects.
It also defines a conversion method `as_result` that turns a Chunk into a retrievable document.


## class Result(BaseModel)
```python
class Result(BaseModel):
    page_content: str
    metadata: dict
```
### What this represents
This mirrors the LangChain Document structure:
* `page_content`: the text used for search or LLM context
* `metadata`: structured info about the source
Think of this as:
“A finalized document ready for retrieval or display.”

### Why this exists
* Standardizes output format
* Makes sure every result has:
  - text
  - metadata
* Easier validation and serialization


## class Chunk(BaseModel)
```python
class Chunk(BaseModel):
```
This represents `one chunk of a document`, but with `extra semantic structure`.
—
### Field 1: `headline`
```python
headline: str = Field(
    description="A brief heading for this chunk, typically a few words, that is most likely to be surfaced in a query"
)
```
* A short, descriptive title
* Useful for:

  - search relevance
  - UI display
  - summarization

Example:
< "Sales Department Overview"


### Field 2: summary
```python
summary: str = Field(
    description="A few sentences summarizing the content of this chunk to answer common questions"
)
```
* Condensed explanation of the chunk
* Often LLM-generated
* Improves:

  - retrieval
  - context quality

Example:
< "Alice Johnson leads enterprise sales and manages key accounts."


### Field 3: `original_text`
```python
original_text: str = Field(
    description="The original text of this chunk from the provided document, exactly as is, not changed in any way"
)
```
* The verbatim source text.
* Important for:

  - factual grounding
  - citations
  - audits


## The method: as_result
```python
def as_result(self, document):
```
This is an instance method of `Chunk`.
* `self` → the current chunk
* `document` → metadata about the source document


### Metadata creation
```python
metadata = {
    "source": document["source"],
    "type": document["type"]
}
```
Extracts selected metadata fields:
* Where the chunk came from
* What kind of document it is

Example of `metadata’
``` python
metadata = {
    "source": "contracts/nda_2023.pdf",
    "type": "contracts",
    "page": 4,
    "chunk_index": 2
}
```

### Combine text fields
``` python
return Result(
    page_content=self.headline + "\n\n" + self.summary + "\n\n" + self.original_text,
Attach metadata
)
```

Creates a `retrieval-ready text block`:
```python
<headline>

<summary>

<original_text>
```
Why this works well:
* Headline helps matching
* Summary boosts semantic recall
* Original text ensures accuracy


## class Chunks(BaseModel)
```python
class Chunks(BaseModel):
    chunks: list[Chunk]
```
### What this represents
* A wrapper containing multiple `Chunk` objects
* Often used for:

  - structured LLM output
  - batch processing
  - validation

Example:
< Chunks(
    chunks=[Chunk(...), Chunk(...)]
)


## Why use BaseModel (Pydantic)?
Using Pydantic gives you:
* Type validation
* Required fields enforcement
* Easy `.json()` / `.dict()` conversion

Clear schemas for LLM structured output
This is especially useful when:
* LLMs generate chunks
* You need predictable structure
* You later convert chunks into vector documents


## How these classes work together (flow)
```python
Raw document
   ↓
Chunk (headline, summary, original_text)
   ↓ as_result()
Result (page_content + metadata)
   ↓
Vector DB / Retriever
```


## Analogy 🧠
**Chunk = edited article with title and abstract**
**Result = indexed library entry**

---

## Step 2 - Fetch Document from Knowledge Base in the Local Folders

``` python
def fetch_documents():
   """customized version of the LangChain Directory Loader"""

   documents = []

   for folder in KNOWLEDGE_BASE_PATH.iterdir():
       doc_type=folder.name
       for file in folder.rglob("*.md"):
           with open(file, "r", encoding = "utf-8") as f:
               documents.append({"type": doc_type, "source": file.as_posix(), "text": f.read()})
   print (f"Loaded {len(documents)} documents")
   return documents

documents = fetch_documents()
```

`fetch_documents` is a customized function that mimics what LangChain DirectoryLoader does, but with:

* custom metadata
* a simpler data structure
* full control over folder logic

## Initialize the result container
```python
documents = []
```
* An empty Python list
* Will hold **one dictionary per document**
* Each dictionary represents a document + metadata

---

## Iterate over top-level folders
```python
for folder in KNOWLEDGE_BASE_PATH.iterdir():
```
**What this means**
* `KNOWLEDGE_BASE_PATH` is a `Path` object (from pathlib)
* `.iterdir()` lists **immediate subfolders/files**

Example directory structure:
```python
knowledge-base/
├── company/
├── contracts/
├── employees/
└── products/
```
Each `folder` represents:
* company
* contracts
* employees
* products

---

## Capture document type from folder name
```python
doc_type = folder.name
```
If the folder is:
< knowledge-base/contracts/

Then:
```python
doc_type == "contracts"
```
This is used as **metadata** later.

---

## Recursively find Markdown files
```python
for file in folder.rglob("*.md"):
```
* `.rglob("*.md")` means:

<  “Find all `.md` files recursively inside this folder”

So it will find:
* `contracts/nda.md`
* `contracts/legal/nda_2023.md`
etc.

---

## Open each file safely
```python
with open(file, "r", encoding="utf-8") as f:
```
* Opens the file in **read mode** -> `“r”`
* Uses UTF-8 encoding
* with ensures the file closes automatically

---

## Read file contents and append metadata
```python
documents.append({
    "type": doc_type,
    "source": file.as_posix(),
    "text": f.read()
})
```
This appends a `dictionary` with:
**🔹 type**
```python
"type": doc_type
```
* Category of document
* Example: `"contracts"`

**🔹 source**
```python
"source": file.as_posix()
```
* Full file path as a string

Example:
<  knowledge-base/contracts/nda_2023.md

**🔹 text**
```python
"text": f.read()
```
* Entire file contents as a string
* This is what will later be:
   - chunked
   - embedded
   - searched

---

## Print a summary
```python
print(f"Loaded {len(documents)} documents")
```
* Uses an f-string
* Reports how many files were loaded
* Helpful for debugging

---

## Return the documents
```python
return documents
```
* Returns a list of dictionaries
* Each element looks like:

```python
{
    "type": "contracts",
    "source": "knowledge-base/contracts/nda_2023.md",
    "text": "Full markdown content here..."
}
```
---

## Call the function
```python
documents = fetch_documents()
```
* Executes the function
* Stores the result in `documents`
