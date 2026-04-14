# Pinecone Vector Database — Past CEQA Documents

We maintain a vector database of past CEQA environmental documents (EIRs, MNDs, Initial Studies, etc.) indexed for semantic search. You can query this to find similar past projects and ground your predictions in real precedent.

## Connection Details

```
Index Name:    ceqa-docs
Host:          ceqa-docs-ur013bm.svc.aped-4627-b74a.pinecone.io
API Key:       
Dimension:     1536 (OpenAI text-embedding-ada-002 / text-embedding-3-small compatible)
Metric:        cosine
Total Vectors: ~9.3 million chunks
```

## Record Schema

Each vector in the index represents a chunk of text from a CEQA document. The metadata fields are:

| Field | Type | Description |
|-------|------|-------------|
| `text_preview` | string | The first ~500 characters of the chunk text |
| `sch` | string | State Clearinghouse number (unique project identifier) |
| `project_title` | string | Name of the CEQA project |
| `lead_agency` | string | Lead agency responsible for the project |
| `doc_type` | string | Document type (EIR, MND, SIR, NOP, etc.) |
| `s3_key` | string | Internal storage key (not useful externally) |
| `chunk_index` | integer | Position of this chunk within the document |

## How to Query

The index uses 1536-dimensional embeddings. To query it:

1. Generate an embedding for your query text using OpenAI's `text-embedding-ada-002` or `text-embedding-3-small` model (both produce 1536-dim vectors).
2. Send the embedding to Pinecone's query endpoint.
3. Use metadata filters to narrow results if needed.

### Python Example (using pinecone-client and openai)

```python
from pinecone import Pinecone
from openai import OpenAI

# Initialize clients
pc = Pinecone(api_key="pcsk_747bna_Hriks9CvcRiZh93ScdpZNcdXGWqhKKMEkkx6h3PPWw8R6vMZ2vZmuwnSZW3jX2n")
index = pc.Index("ceqa-docs")
openai_client = OpenAI()  # uses OPENAI_API_KEY env var

# Generate embedding for your query
query = "residential development near wetlands biological resources impacts"
embedding = openai_client.embeddings.create(
    input=query,
    model="text-embedding-ada-002"
).data[0].embedding

# Query Pinecone
results = index.query(
    vector=embedding,
    top_k=10,
    include_metadata=True
)

# Print results
for match in results.matches:
    meta = match.metadata
    print(f"Score: {match.score:.3f}")
    print(f"Project: {meta.get('project_title', 'Unknown')}")
    print(f"Agency: {meta.get('lead_agency', 'Unknown')}")
    print(f"SCH: {meta.get('sch', 'N/A')}")
    print(f"Type: {meta.get('doc_type', 'N/A')}")
    print(f"Text: {meta.get('text_preview', '')[:200]}...")
    print("---")
```

### Filtering by Document Type

You can filter results to only return specific document types:

```python
results = index.query(
    vector=embedding,
    top_k=10,
    include_metadata=True,
    filter={"doc_type": {"$in": ["EIR", "MND"]}}
)
```

### Common Document Types

- `EIR` — Environmental Impact Report (most comprehensive)
- `MND` — Mitigated Negative Declaration
- `NOP` — Notice of Preparation
- `SIR` — Supplemental/Subsequent EIR
- `NOD` — Notice of Determination

## Tips

- Query with specific environmental topics ("traffic impacts residential subdivision creek") rather than generic project descriptions for better results.
- The `text_preview` field is truncated — it gives you enough context to know if a chunk is relevant, but not the full text.
- Use the `sch` number to group chunks from the same project — a single project may have hundreds of chunks across its documents.
- Filter by `doc_type: "EIR"` to get the most detailed environmental analyses.
- The database covers projects across California from many different lead agencies and project types.

## Rate Limits

This is a shared API key. Please be reasonable with query volume — a few hundred queries during your project is fine. Don't run batch jobs that fire thousands of queries.
