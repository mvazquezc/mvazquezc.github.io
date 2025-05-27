---
title:  "A Beginner’s Guide to RAG: What I Wish Someone Told Me"
author: "Mario"
tags: [ "artificialintelligence", "ai", "llm", "llms", "rag" ]
url: "/rag-beginners-guide"
draft: false
date: 2025-05-27
lastmod: 2025-05-27
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# A Beginner’s Guide to RAG: What I Wish Someone Told Me

In this post, I'll try to provide a beginners guide to RAG, focusing on what I wish someone told me before trying to build a RAG solution.

{{<attention>}}
While I’ve made a strong effort to ensure the information is accurate, I’m far from an expert on the topic, and some details may not be entirely correct. If you notice anything missing or inaccurate, please leave a comment!
{{</attention>}}

## What is Retrieval-Augmented Generation (RAG)

RAG (Retrieval-Augmented Generation) is an AI technique that combines a language model with a search system to retrieve relevant documents and use them as context for generating more accurate, informed, and up-to-date responses. It enhances output by grounding it in external knowledge.

To put it simple, we provide the [LLM (Large Language Model)](https://linuxera.org/introduction-to-llm-concepts/) with a set of documents it has to use when answering. These documents, are processed and stored in a vector database.

## Embeddings

To make documents usable for a Retrieval-Augmented Generation (RAG) system, they must be converted into vector embeddings (numerical representations that capture semantic meaning). These embeddings are generated using a specialized embedding model, and then stored in a vector database, where they can later be searched and retrieved based on similarity to a user’s query.

The typical flow looks like this:

Document -> Embedding Model ([tokenizes internally](https://linuxera.org/introduction-to-llm-concepts/#what-are-tokens-in-the-context-of-llms)) -> Vector Database

## Context Window

We described Context Window in more detail in a [previous blog post](https://linuxera.org/introduction-to-llm-concepts/#what-is-the-context-windowcontext-lengthmodel-max-length), but here’s a quick refresher.

The Context Window defines the maximum number of tokens (words, subwords, or characters depending on the tokenizer) the model can process in a single input sequence (including both the prompt and any output generated during a single pass).

For example, if a model has a context window of 2048 tokens, and the prompt uses 1000 tokens, the model has 1048 tokens left for generating a response.

### Why is the Context Window important for RAG?

In Retrieval-Augmented Generation (RAG), once relevant documents are retrieved from the vector database, they are injected into the context window along with the user’s query. This is how the model gains access to external knowledge at inference time.

If the combined length of the query and retrieved content exceeds the context window, the model won’t be able to see the full input. This can result in:

- Truncated documents
- Missing critical context
- Inaccurate or hallucinated responses

In summary, the context window acts as the model's working memory. Keeping the inputs within its limits is essential for accurate, high-quality answers in RAG systems.

## Key Considerations for RAG

### Document Quality over Quantity

"Garbage in, garbage out" applies especially to RAG systems. Even the most advanced LLM can't compensate for low-quality source material. If you wouldn't hand a document to a new hire to explain a topic, don’t hand it to your model either.

To improve the quality of your documents, consider the following strategies:

- Collaborate with content owners to update and improve documentation.
- Use LLMs to summarize, clean up, or extract key points — there are specialized models for creating FAQs, concise summaries, etc.

Also, prepare documents with retrievability in mind. Remove elements that are hard to parse or irrelevant for text-based models:

- Diagrams, images, screenshots
- Long blocks of low-readability content (e.g., raw tracebacks or logs)

In summary, don’t try to go from zero to hero. Start with high-quality documents you trust, and gradually expand your ingestion pipeline as you validate the results.

### Unstructed vs Structured

In RAG, it's critical to understand the type of data you’re working with:

- Unstructured data: Free-form text like articles, web pages, manuals, wikis, and documentation. No strict schema.
- Structured data: Information with a predefined schema, such as database rows, spreadsheets, or API responses.

If you’re working with structured data, classic RAG may not be the best fit. Instead, consider using Agentic RAG, where the LLM is given access to tools (like APIs or SQL queries) that retrieve structured data on-demand and inject it into the context window.

Agentic RAG also works well when you need to blend structured and unstructured sources in a single system.

## RAG Searches

Retrieval-Augmented Generation (RAG) systems rely on effective document retrieval techniques to surface relevant context for the LLM. There are three main search strategies you should be familiar with: semantic search, keyword search, and hybrid approaches.

### Semantic Search

Semantic search focuses on understanding the meaning behind a query rather than matching specific words. It uses vector embeddings to represent both the query and documents in a high-dimensional space, then measures their similarity.

This is especially useful when the user’s query uses different phrasing than the source material — for example, searching "How to get a refund?" could match documents about "return policies."

Common similarity metrics include:

- Cosine Similarity: Measures the angle between two vectors — good for checking direction/semantic similarity.
- Dot Product: Takes into account both similarity and vector magnitude — useful when embedding magnitude carries importance. (Be cautious: unnormalized dot product can favor longer text regardless of relevance.)
- Euclidean Distance: Measures literal distance — often used in clustering or when you want to group semantically close documents.

#### Semantic Search Algorithms

##### Cosine Similarity

Compare two texts to see if they talk about the same thing, even if one is longer.

- Doc A: "The capital of France is Paris."
- Doc B: "Paris is the capital city of France, known for the Eiffel Tower.”

Despite different lengths and wordings, their cosine similarity will be high because they are semantically aligned.

##### Dot Product

Prioritize documents that not only match the query direction but also carry stronger signals (e.g., longer content, weighted terms).

- Doc A: "200 words about EU regulations on emissions."
- Doc B: "2000 words with comprehensive policy analysis, including emissions and carbon tax."

Both match query direction, but doc B has a higher magnitude. Possibly due to length, or emphasis.

##### Euclidean Distance

Grouping documents with similar embeddings into clusters, regardless of angle. Closer means more similar overall vector positions.

- You embed 1000 support tickets, and you want to cluster them into 5 issue types. Euclidean will help you do K-Means Clustering to find natural groupings.

### Keyword / Full-text Search

Keyword-based search is more literal. It tries to find exact words or phrases within documents — like traditional search engines or grep. It’s ideal when the user is looking for a specific term, phrase, or entity.

A commonly used algorithm here is:

- BM25 (Best Match 25): A probabilistic model that ranks documents based on the occurrence and frequency of query terms, adjusted by document length.

While it doesn’t capture semantic meaning, keyword search is fast, simple, and precise — especially for technical content with consistent terminology.

### Hybrid Search

Hybrid search combines both semantic and keyword search to maximize recall and precision. A typical approach might:

1. Use keyword search (e.g: BM25) to quickly narrow down a large document set.
2. Use semantic search to rank or rerank the candidates based on relevance and meaning.

This approach gives you the best of both worlds: the precision of keywords with the flexibility of semantics.

The order of operations in hybrid search isn’t fixed — it depends on your use case:

- Keyword First → Semantic Rerank: Good when you have a lot of documents and the question uses specific terms. It’s fast and works well.
- Semantic First → Keyword Filter or Boost: Good when people might phrase things differently. Helps find relevant results even if the wording doesn’t match exactly.

## Improving RAG Searches

A RAG system is only as good as the documents it retrieves. Even if your documents are high-quality, poor retrieval can lead to weak answers. These strategies help improve document retrieval and relevance.

### Reranking

When your system retrieves documents — whether using semantic, keyword, or hybrid search — you typically get a list of candidates with scores. Reranking helps reorder these results to better match your specific use case.

Here are common reranking strategies:

- LLM-based Reranking: Use a language model to evaluate which results best answer the user’s question.
- Cross-Encoder Reranking: Use a model that takes both the query and document together to assign a new relevance score.
- Custom Rerankers: Create your own logic to boost or demote certain results based on metadata (e.g., date, author, document type, etc.).

### Refine user's questions

Sometimes the original question doesn’t provide enough context, especially in ambiguous cases. For example:

- User query: Where can I buy a mouse? 
- LLM: Do they mean a computer peripheral or a pet?

To avoid confusion, you can refine the query using an LLM by injecting context. If you know your documents cover internal hardware policies, your refinement prompt might look like:

- "You're a helpful assistant specialized in helping users buy computer hardware following company policy."

Make sure to include the user’s original question in the final prompt. This helps keep the user’s intent while clarifying the meaning for better retrieval.

### Use the right algorithm for semantic search

Different similarity algorithms can give different retrieval behaviors. Choose based on what matters most in your case.

You can also combine them, for example:

1. Use [Cosine Similarity](#cosine-similarity) to get meaning-aligned docs.
2. Use [Dot Product](#dot-product) to emphasize importance or weight.

## Improving RAG Answers

Once your RAG system retrieves the right documents, the next step is generating accurate and trustworthy answers. Here are some key practices to improve answer quality.

### Always Include Soruce Links

Make sure the system outputs links or references to the documents used to generate an answer. This improves transparency and helps users verify the information.

### Limit the Number of Documents Used

Allow your system to restrict the number of retrieved documents passed into the context window — for example, the top 3 most relevant ones.

Fewer, more relevant documents often lead to better answers than many loosely related ones.

### Manage the Context Window Wisely

The context window is finite — and balance is key. When injecting documents into the prompt:

If the documents are small and fit entirely into the context window, include the full document:

- But don’t overload the context —> this can confuse the model (overstuffing).
- And don’t underload —> the model may hallucinate or miss key details (undersupplying).

Example:

- Want a summary of Chapter 1 of a book? ➜ Send only Chapter 1.
- Want to know how many times a character appears? ➜ Send the whole book (if it fits).

## Embeddings Strategies

In a RAG system, how you generate embeddings has a big impact on both retrieval performance and answer quality. The two most common approaches are full-length and chunk-level embeddings — and each has trade-offs.

### Full-Length Embeddings

Full-length embeddings are vector representations of entire documents or large text blocks. Their goal is to capture the overall semantic meaning of the content.

- Efficient retrieval: Fewer vectors to store and search (e.g., 1 vector per document).
- Lower granularity: May miss fine details buried deep in the text.

This approach is useful when you're trying to quickly identify which document is most relevant — like finding the right manual or report.

### Chunk-Level Embeddings

Chunk-level embeddings break documents into smaller parts — sentences, paragraphs, or sections — and generate vectors for each chunk. These embeddings focus on local details.

- Higher precision: Great for surfacing exact sections relevant to a query.
- More expensive: You’ll need to store and search many more vectors. While more vectors improve granularity, they increase index size, memory use, and retrieval latency — especially for large corpora.

This approach shines when specific answers are scattered throughout large documents — like answering “What ports does this device support?” from a 100-page manual.

### Combine the Two

A smart strategy is to combine both approaches:

1. Start with full-length embeddings to quickly identify relevant documents. This narrows down your search space (e.g., 1,000 vectors instead of 10,000).
2. Then, search within those documents using chunk-level embeddings to locate the most relevant sections.

This hybrid method may require using different embedding models optimized for different levels of granularity — but the payoff is better retrieval and more accurate answers.

## RAG vs Fine-Tuning

RAG injects relevant external documents into the model’s context at inference time to improve responses without changing the model itself. It’s flexible and keeps up with changing information.

Fine-tuning modifies the model's internal weights by training it on task-specific data. This can improve performance but updates require retraining and are less responsive to real-time changes compared to RAG.

In summary, RAG adds knowledge dynamically at inference time; fine-tuning bakes knowledge into the model permanently.

## RAFT (RAG + Fine-Tuning)


RAFT (Retrieval-Augmented Generation + Fine-Tuning) brings together the strengths of both approaches to maximize performance and flexibility.

- Fine-tune your LLM on high-quality, stable knowledge (e.g., internal documentation, product specs, historical data).
- Use RAG to inject dynamic or frequently updated information (e.g., current policies, real-time data) at query time.

This hybrid strategy is especially effective when your system needs to reason over both historical and real-time knowledge.

### Vague Recollection vs Working Memory

To understand how RAFT works, it helps to think of your LLM like a person:

- Fine-tuned knowledge is like vague recollection — facts learned and internalized over time. Stored in the model’s weights.
- RAG-injected knowledge is like working memory — recent or immediate context added temporarily. Stored in the context window.

Example:

- Summarizing a book you read months ago? That's vague recollection (fine-tuned knowledge).
- Summarizing a chapter you just read? That’s working memory (RAG).

By combining both, RAFT enables your LLM to be knowledgeable, adaptable, and context-aware.

## Useful Resources

- [Major's Hayden blog on RAG](https://major.io/p/dont-tell-me-rag-is-easy/)
- [Embedding models leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
- [Effective context window size for models (NVIDIA Ruler)](https://github.com/NVIDIA/RULER)
- [Repository with advanced RAG techniques](https://github.com/NirDiamant/RAG_Techniques)
- [LlamaIndex Introduction to RAG](https://docs.llamaindex.ai/en/stable/understanding/rag/)
- [LLamaIndex Production RAG](https://docs.llamaindex.ai/en/stable/optimizing/production_rag/)
- [r/RAG](https://www.reddit.com/r/Rag/)
- [SML RAG Arena](https://huggingface.co/spaces/aizip-dev/SLM-RAG-Arena)

## What's Next?

Go and try building a RAG system by using existing frameworks like [LLamaIndex](https://www.llamaindex.ai/) or [LangChain](https://www.langchain.com/).

In the next days I'll be publishing a test app I made while trying to learn about RAG internals.