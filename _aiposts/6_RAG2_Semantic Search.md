---
title:  "[RAG Basics]: Semantic Search Using Embedding LLMs"
layout: post
---
[last updated: 18 Dec 2025]

##  Understanding RAG Basics: Semantic Search Using Embedding LLMs
Have you ever wondered how AI assistants can answer questions about specific documents or latest information? The answer often lies in a technique called RAG - Retrieval-Augmented Generation.

### What is RAG?
RAG is like giving a super-smart assistant access to a reference library. While Large Language Models (LLMs) like ChatGPT know a lot from their training, RAG connects them to external, up-to-date sources—documents, databases, or any knowledge base. This helps the AI provide more accurate, current, and context-aware answers while reducing made-up information (called "hallucinations").
Here's how it works in simple terms:
1. Retrieve: The system searches through your documents to find relevant information
2. Augment: It adds this information to the AI's "thinking process"
3. Generate: The AI produces an answer based on both its knowledge and the retrieved information

### Why Vector Databases?
In practice, RAG often uses vector databases. Why? Because they're excellent at finding meaning-based connections (semantic search) in `unstructured data` like text documents. Think of it this way: instead of searching for exact keywords, the system understands what you're looking for and finds conceptually similar content.
It's worth noting that for organized, structured data like sales figures or customer records in the enterprise environment, traditional databases like SQL are often more cost-effective. RAG really shines when working with unstructured information—documents, emails, reports, and similar content. 

> However, the process of improving RAG performance is equally useful even for structured and small-scale data. This is because the core challenge of optimizing the retrieval of relevant context from any database remains critical.

### Project Overview: A Simple RAG Implementation
In this project, I've created a basic RAG system that demonstrates how semantic search works:
1. Knowledge Base Setup: Created a local folder with various company documents (HR policies, product details, contracts, company overview) converted to .md format
2. Document Processing: Used LangChain framework to load and prepare the text
3. Vector Database Creation: Converted the text into numerical representations (embeddings) and stored them for efficient searching
4. Semantic Search Implementation: Built a system that understands the meaning behind queries

### How It Works: The Two-Agent System
Imagine two AI assistants working together:

**Agent 2 - The Researcher:**
* Receives your question
* Interprets what you're really asking
* Searches through the document database to find the most relevant information
* Passes this context to Agent 1

**Agent 1 - The Answer Expert:**
* Receives both your original question AND the relevant information from Agent 2
* Uses this combined knowledge to craft a precise, helpful answer
This teamwork is crucial! If Agent 2 finds poor or incorrect information, Agent 1's answer won't be helpful—even if Agent 1 is very capable.

### Important Notes About This Project
This is a simplified demonstration focusing on how semantic search works within RAG systems. The current setup isn't optimized for perfect search results—that requires additional evaluation and fine-tuning, which I cover in separate posts, in which I explain concepts related to LLM engineering:
1. How to evaluate and improve RAG system/fuzzy search performance
2. Using tools to handshake with other types of database
3. How to handle multiple document types (PDFs, emails, etc.)
4. How to integrate with real-world tools like Google Workspace and Outlook
5. etc

---
## Let’s Dive into the Codes

## Load Library and Package

LangChain as a community is resourceful, it provides a linkage to the wider LLM community that allow LLM engineers to access at ease. However, over time it gets heavier and heavier. LangChain has a major overhaul released on October 22, 2025. In this project, I used LangChain to ease the work, as you can see from the “load package” below.  
I have a separate project that builds the RAG pipeline without LangChain or any framework, that allows us to have the full control to finetune and optimize the RAG output. 
 
```python
import os
import glob
import tiktoken
import numpy as np
from dotenv import load_dotenv
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.document_loaders import DirectoryLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.messages import SystemMessage, HumanMessage
from sklearn.manifold import TSNE
import plotly.graph_objects as go
import gradio as gr

load_dotenv(override=True)
```

Load library and package; Using LangChain to demonstrate RAG basics.

```python
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
#embeddings = OpenAIEmbeddings(model="text-embedding-3-large")

if os.path.exists(db_name):
   Chroma(persist_directory=db_name, embedding_function=embeddings).delete_collection()

vectorstore = Chroma.from_documents(documents=chunks, embedding=embeddings, persist_directory=db_name)
```

`Chroma.from_documents(documents=chunks, embedding=embeddings, persist_directory=db_name)` handles the embedding of knowledge base, which is preprocessed to become `chunks`

```python
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = text_splitter.split_documents(documents)
```

`chunks` are created using `RecursiveCharacterTextSplitter( )` that splits `documents`.

```python
folders = glob.glob("knowledge-base/*")
documents = []
for folder in folders:
   doc_type = os.path.basename(folder)
   loader = DirectoryLoader(folder, glob="**/*.md", loader_cls=TextLoader, loader_kwargs={'encoding': 'utf-8'})
   folder_docs = loader.load()
   for doc in folder_docs:
       doc.metadata["doc_type"] = doc_type
       documents.append(doc)

print(f"Loaded {len(documents)} documents")
```

`document` are created from the knowledge base stored in the local folder at directory `knowledge-base/*`  using the LangChain `TextLoader`. 

### A bit of Gradio
```python
gr.ChatInterface(answer_question).launch()
```
This line tells Gradio: “Use answer_question as the backend function for a chat UI.”
From this point on, Gradio controls how the function is called.
When you use `gr.ChatInterface(fn)`, Gradio expects `fn` to have this signature:
``` python
def fn(message, history):
    ...
```
Gradio will automatically supply:
`Argument`
Supplied by
Meaning
question
Gradio
The latest user message
history
Gradio
Full conversation history

| Argument         | Supplied by      | Meaning                    |
|------------------|------------------|----------------------------|
| question         | Gradio           | The latest user message    |
| history          | Gradio           | Full conversation history  |

You did not create these variables (question and history) — Gradio injects them.
`question` is a **string**, It is the **latest user message** typed into the chat UI.
Every time the user sends a message:
Gradio calls `answer_question` function (refer to next part of code)
Passes the new message as `question`
history is a **list representing the conversation so far**. Gradio maintains this automatically.
You never reference history. That’s fine because:
* Gradio **still passes it**
* Python allows unused parameters
* Your chatbot is **stateless**
Gradio is designed for:
* **multi-turn chat**
* **memory-enabled bots**
**Even if you don’t use it now, you can later.**
Each answer in chat depends only on:
* the current question
* retrieved documents
This is typical for RAG systems.

## answer_question ()
```python
def answer_question(question: str, history):
   docs = retriever.invoke(question)
   context = "\n\n".join(doc.page_content for doc in docs)
   system_prompt = SYSTEM_PROMPT_TEMPLATE.format(context=context)
   response = llm.invoke([SystemMessage(content=system_prompt), HumanMessage(content=question)])
   return response.content
```
‘retriever.invoke( )’ call the Chroma to embed the `query’ (user’s question input on Gradio UI), using the encoder LLM, and search for similarity based on cosine similarity.
`docs` is List [Document], Each Document contains: `page_content`, and `metadata`. In LangChain-style RAG pipelines, `retriever.invoke(question)` standardly returns `Document` objects, and those `Document` objects standardly contain `page_content` and `metadata`. 

All `page_content` were then joined to form `context` that will be passed to the `SYSTEM_PROMPT_TEMPLATE`, together they form the complete ‘system_prompt’ for the chat LLM to respond to the user

```python
retriever = vectorstore.as_retriever()
llm = ChatOpenAI(temperature=0, model_name=MODEL)
```

```python
vectorstore = Chroma(persist_directory=DB_NAME, embedding_function=embeddings)
```
### What this does
Creates (or loads) a Chroma vector database
Uses `DB_NAME` as the on-disk storage location
Associates the vector store with the embedding function
The embedding is NOT done yet at this step
### Why `embedding_function` is needed
When you later do: ‘vectorstore.similarity_search("Who works in sales?")’
**Chroma** will:
* Use embeddings to embed the query text
* Compare that query vector to stored vectors
* Return the most similar documents
So: **The vector store doesn’t know how to embed text by itself — you must give it a model.**
```python
MODEL = "gpt-4.1-nano"

```python
DB_NAME = "vector_db"
```

**What this does**
a standard LangChain embedding interface
Creates an embedding object using a Hugging Face sentence-transformer model
`all-MiniLM-L6-v2` is a pretrained text embedding model
This object is callable and knows how to:
Take text → output a numeric vector
"Alice works in sales" → `[0.012, -0.341, ..., 0.882]  # length 384`
What kind of model is this?
`all-MiniLM-L6-v2` is:
A sentence-transformer
Optimized for semantic similarity
Lightweight and fast
Embedding dimension: 384

```python
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

SYSTEM_PROMPT_TEMPLATE = """
You are a knowledgeable, friendly assistant representing the company Insurellm.
You are chatting with a user about Insurellm.
If relevant, use the given context to answer any question.
If you don't know the answer, say so.
Context:
{context}
"""
```

`context’ is the semantic search based on user’s `query` and vector database

```python
llm = ChatOpenAI(temperature=0, model_name=MODEL)
```
