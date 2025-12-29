---
title:  "[RAG Advanced] Building a Custom RAG Pipeline for Granular Performance Tuning"
layout: post
---
[last updated: 18 Dec 2025]

## OVERVIEW
This project features a high-performance RAG pipeline built from the ground up without high-level frameworks like LangChain or LlamaIndex. By "mimicking" these frameworks through custom logic, this implementation provides full control over every component, allowing for precise performance fine-tuning and the application of advanced optimization techniques.

## Core Pipeline Architecture - Key Steps
* **Standardized Output:** Implemented `Pydantic` classes to enforce schema validation and ensure consistent data flow across the pipeline.
* **Document Ingestion:** Developed custom connectors to fetch and process documents from local knowledge bases.
* **LLM-Powered Chunking:** Leveraged a `"Parser LLM"` to transform raw text into structured, semantically coherent objects defined by the Pydantic schema.
* **Vector Storage:** Integrated an `Encoder LLM` to transform chunks into embeddings, stored in a local vector database for high-dimensional retrieval.
* **Performance Evaluation:** Built an automated evaluation loop that benchmarks generated responses against a curated dataset of `test_questions` and `test_answers`. (Evaluation is illusrated in a seperate post)

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

## STEP 1 - Create Pydantic Class for Standard Output Format

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
* `Chunks` — a container holding multiple Chunk objects.
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
> "Sales Department Overview"


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
> "Alice Johnson leads enterprise sales and manages key accounts."


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
* `Chunks` is a Pydantic model that says:
   - “Valid JSON must contain a key called `chunks`, whose value is a list of `Chunk` objects.”
* Often used for:

   - structured LLM output
   - batch processing
   - validation

Example:
```python
Chunks(
    chunks=[Chunk(...), Chunk(...)]
)
```


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

## STEP 2 - Fetch Document from Knowledge Base in the Local Folders

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

## What this code is doing - Big picture
`fetch_documents` is a customized function that mimics what LangChain DirectoryLoader does, which is to load the document in the local drives, but with:

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


## Open each file safely
```python
with open(file, "r", encoding="utf-8") as f:
```
* Opens the file in **read mode** -> `“r”`
* Uses UTF-8 encoding
* with ensures the file closes automatically


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


## Print a summary
```python
print(f"Loaded {len(documents)} documents")
```
* Uses an f-string
* Reports how many files were loaded
* Helpful for debugging


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


## Call the function
```python
documents = fetch_documents()
```
* Executes the function
* Stores the result in `documents`

---

## STEP 3 - Implement a classic LLM-assisted chunking pipeline

```python
MODEL = "gpt-4o-mini"
KNOWLEDGE_BASE_PATH = Path("knowledge-base")
AVERAGE_CHUNK_SIZE = 500

chunks = create_chunks(documents)

def create_chunks(documents):
   chunks = []
   for doc in tqdm(documents):
       chunks.extend(process_document(doc))
   return chunks

def process_document(document):
   messages = make_messages(document)
   response = completion(model=MODEL, messages=messages, response_format=Chunks)
   reply = response.choices[0].message.content
   doc_as_chunks = Chunks.model_validate_json(reply).chunks
   return [chunk.as_result(document) for chunk in doc_as_chunks]
  
def make_messages(document):
   return [
       {"role":"user", "content": make_prompt(document)}
   ]


def make_prompt(document):
   how_many = (len(document["text"]) // AVERAGE_CHUNK_SIZE) + 1
   return f"""
You make a document and you split the document into overlapping chunks for a KnowledgeBase.
The document is of type: {document["type"]}
The document has been retrieved from: {document["source"]}
This document should probably be split into {how_many} chunks, but you can have more or less as appropriate.
Here is the document:
{document["text"]}
Respond with the chunks.
"""
```

## What this code is doing - Big picture
**This pipeline:**
1. Takes raw documents (text + metadata) from step 2
2. Asks an LLM to **split each document into chunks**
3. Parses the LLM’s structured response
4. Converts each chunk into a retrieval-ready object

**Flow:**
```python
documents
  ↓
create_chunks
  ↓
process_document (per document)
  ↓
LLM (structured output)
  ↓
Chunk objects
  ↓
Result objects
```

## Entry point
```python
chunks = create_chunks(documents)
```

`documents` is a list of dicts like:
```python
 {
    "type": "contracts",
    "source": "knowledge-base/contracts/nda.md",
    "text": "Full document text..."
}
```
* `chunks` will become a **flat list of chunked results** across all documents.


## create_chunks(documents)
```python
def create_chunks(documents):
    chunks = []
    for doc in tqdm(documents):
        chunks.extend(process_document(doc))
    return chunks
```
### What’s happening
* `chunks = []`
   - Initialize the final output list

* `for doc in tqdm(documents):`
   - Loop over every document
   -`tqdm` adds a **progress bar** (purely visual)

* `chunks.extend(process_document(doc))`
   - `process_document(doc)` returns a **list of chunks**

`.extend()` flattens them into one list

⚠️ Why `extend` and not `append`?
* `append` → list of lists ❌
* `extend` → one flat list ✅


## process_document(document)
```python
def process_document(document):
    messages = make_messages(document)
    response = completion(
        model=MODEL,
        messages=messages,
        response_format=Chunks
    )
    reply = response.choices[0].message.content
    doc_as_chunks = Chunks.model_validate_json(reply).chunks
    return [chunk.as_result(document) for chunk in doc_as_chunks]
```

### Step 3.1: Build messages
```python
messages = make_messages(document)
```
This prepares the prompt sent to the LLM.


## make_messages(document)
```python
def make_messages(document):
    return [
        {"role": "user", "content": make_prompt(document)}
    ]
```
* Returns a **Chat API–compatible message list**
* Only a `user` message (no system prompt here)
* The content is generated by `make_prompt`


## make_prompt(document)
```python
def make_prompt(document):
    how_many = (len(document["text"]) // AVERAGE_CHUNK_SIZE) + 1
    return f"""
You make a document and you split the document into overlapping chunks for a KnowledgeBase.
The document is of type: {document["type"]}
The document has been retrieved from: {document["source"]}
This document should probably be split into {how_many} chunks, but you can have more or less as appropriate.
Here is the document:
{document["text"]}
Respond with the chunks.
"""
```

### What this does
Calculates an **approximate chunk count**
```python
 how_many = text_length // average_chunk_size + 1
```
* This is **guidance**, not a hard rule
* Provides:
   - document type
   - source
   - full text

* Instructs the LLM to return chunks

This is **prompt-engineered chunking**, not mechanical chunking.


## Call the LLM
```python
response = completion(
    model=MODEL,
    messages=messages,
    response_format=Chunks
)
```
### Key points
* `completion(...)` calls the LLM
* `response_format=Chunks` tells the LLM:

< “Return JSON that matches the Chunks Pydantic model”

So the model is expected to output something like:
```python
{
  "chunks": [
    {
      "headline": "...",
      "summary": "...",
      "original_text": "..."
    }
  ]
}
```


## Extract the model’s reply
```python
reply = response.choices[0].message.content
```
* Standard ChatCompletion structure
* `reply` is a **JSON string**


## Validate and parse structured output
```python
doc_as_chunks = Chunks.model_validate_json(reply).chunks
```
### What happens here
1. `model_validate_json(reply)`

   - Parses JSON
   - Validates it against the `Chunks` schema
   - Raises errors if malformed

2. `.chunks`
   - Extracts the list of `Chunk` objects


Now you have:
```python
List[Chunk]
```
Each `Chunk` has:
* `headline`
* `summary`
* `original_text`


## Convert chunks into retrieval-ready results
```python
return [chunk.as_result(document) for chunk in doc_as_chunks]
```
This uses the method you defined earlier:
`chunk.as_result(document)`

Which:
* Combines headline + summary + original text
* Adds metadata `(source, type)`
* Produces a `Result` object
Final output of `process_document`:
`List[Result]`


## Final result of the whole pipeline
```python
chunks = create_chunks(documents)
```
Now:
`chunks == List[Result]`

Each `Result` looks like:
```python
Result(
    page_content="Headline\n\nSummary\n\nOriginal text",
    metadata={
        "source": "...",
        "type": "contracts"
    }
)
```
These are ready for:
* embedding
* vector storage
* retrieval

---

## STEP 4 - Create Vector Storage for Semantic Search

``` python
DB_NAME = "preprocessed_db"
collection_name = "docs"
embedding_model = "text-embedding-3-large"

create_embeddings(chunks)

def create_embeddings(chunks):
   chroma = PersistentClient(path=DB_NAME)
   if collection_name in [c.name for c in chroma.list_collections()]:
       chroma.delete_collection(collection_name)

   texts = [chunk.page_content for chunk in chunks]
   emb = openai.embeddings.create(model=embedding_model, input=texts).data
   vectors = [e.embedding for e in emb]

   collection = chroma.get_or_create_collection(collection_name)

   ids = [str(i) for i in range(len(chunks))]
   metas = [chunk.metadata for chunk in chunks]

   collection.add(ids = ids, embeddings=vectors, documents=texts, metadatas=metas)
   print(f"Vectorstore created with {collection.count()} documents")
```

## Big-picture overview
This code:
1. Embeds your chunks
2. Stores them in Chroma
3. Attaches metadata
4. Persists everything to disk

It is the foundation of semantic search and RAG

Flow:
```python
chunks (Result objects)
   ↓
text embeddings (numeric vectors)
   ↓
Chroma collection on disk
```
After this runs, you have a persistent vector database ready for semantic search.


## Configuration variables
```python
DB_NAME = "preprocessed_db"
collection_name = "docs"
embedding_model = "text-embedding-3-large"
```
### What these mean
* `DB_NAME`
   - Folder on disk where Chroma stores vectors
   - Will be created if it doesn’t exist

* `collection_name`
   - Logical name for a group of vectors
   - Similar to a table name in a database

* `embedding_model`
   - Name of the OpenAI embedding model
   - Produces high-dimensional vectors (3,072 dims)


## Entry point
```python
create_embeddings(chunks)
```
* `chunks` is a `List[Result]`
* Each `Result` has:
   - `page_content` (text)
   - `metadata` (dict)


## Create or connect to Chroma
```python
chroma = PersistentClient(path=DB_NAME)
```
* Creates a **persistent Chroma client**
* Vectors are stored on disk, not just in memory
* You can reload them later


## Remove old collection (optional reset)
```python
if collection_name in [c.name for c in chroma.list_collections()]:
    chroma.delete_collection(collection_name)
```
### What this does
* Lists all existing collections
* If `"docs"` already exists:
   - Deletes it
   - Starts fresh

⚠️ This **destroys old embeddings** — useful during development, risky in production.


## Extract text from chunks
```python
texts = [chunk.page_content for chunk in chunks]
```
* List comprehension
* Produces:


- `List[str]`

Each string is what gets embedded.


## Call OpenAI Embeddings API
```python
emb = openai.embeddings.create(
    model=embedding_model,
    input=texts
).data
```
### What happens
* Sends all texts to OpenAI
* OpenAI returns one embedding per text
* `emb` is a list of objects like:
```python
{
    "embedding": [0.012, -0.456, ...],
    "index": 0
}
```


## Extract raw vectors
```python
vectors = [e.embedding for e in emb]
```
Now you have:
`List[List[float]]`
One vector per chunk.


## Create or get a collection
```python
collection = chroma.get_or_create_collection(collection_name)
```
* Creates `"docs"` if missing


*Otherwise reuses it


## Generate document IDs
```python
ids = [str(i) for i in range(len(chunks))]
```
* Chroma requires **string IDs**
* One ID per vector

Example:
> ["0", "1", "2", ...]


## Extract metadata
```python
metas = [chunk.metadata for chunk in chunks]
```
Produces:
`List[dict]`
Metadata is stored **alongside** vectors but **not embedded**.

## Add everything to Chroma
```python
collection.add(
    ids=ids,
    embeddings=vectors,
    documents=texts,
    metadatas=metas
)
```
### What gets stored
| Field          | Purpose                 | 
|----------------|-------------------------|
| ids            | unique identifier       | 
| embeddings     | numeric vectors         | 
| documents      | original text           |
| metadatas      | filtering & provenance  |

This is the **moment your vector database is created.**

## Print confirmation
```python
print(f"Vectorstore created with {collection.count()} documents")
```
* Confirms how many vectors were stored
* Useful sanity check


## Behind the scenes (important)
After this completes:
* Embeddings are persisted to disk
* You can reload them later with:

```python
chroma = PersistentClient(path=DB_NAME)
collection = chroma.get_collection("docs")
```
No re-embedding needed.

---

### Mental model 🧠
Chunks → Numbers → Stored Geometry

### Final takeaway
This code:

Embeds your chunks

Stores them in Chroma

Attaches metadata

Persists everything to disk

It is the foundation of semantic search and RAG


