---
title:  "[RAG Advanced] Building RAG From Scratch and Advanced Techniques"
layout: post
---
[last updated: 18 Dec 2025]

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

---

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

---

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

---

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

---

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

---

## The method: as_result
```python
def as_result(self, document):
```
This is an instance method of `Chunk`.
* `self` → the current chunk
* `document` → metadata about the source document

---

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

---

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

---

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

---

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

---

## Analogy 🧠
**Chunk = edited article with title and abstract**
 **Result = indexed library entry**
