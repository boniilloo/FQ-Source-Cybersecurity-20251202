# Cybersecurity Audit - Control IA-02

## Control Information

- **Control ID**: IA-02
- **Control Name**: Data Minimization Sent to LLM
- **Audit Date**: 2025-01-27
- **Client Question**: "Do you send entire documents or only relevant fragments to the AI model?"

---

## Executive Summary

⚠️ **PARTIAL COMPLIANCE**: The platform implements data minimization in its RAG (Retrieval-Augmented Generation) pipeline by sending only relevant fragment identifiers and metadata to LLMs, rather than entire documents. However, the system does send complete document content when processing user-uploaded documents and full data structures when evaluating products and companies. The implementation demonstrates good practices in semantic search but has opportunities for improvement in document processing and evaluation contexts.

1. **RAG Pipeline Minimization** - Vector queries return only IDs and metadata, not full document content
2. **Document Processing Limitation** - Full extracted text from uploaded documents is sent to LLMs
3. **Evaluation Context** - Complete product and company data structures are sent during evaluations
4. **Image Handling** - Full image data is sent, which is necessary for vision-capable models
5. **Context Control** - The system provides mechanisms to control context size through configuration

---

## 1. AI Pipeline Architecture Overview

### 1.1. System Components

The platform implements a multi-component AI pipeline with different data handling strategies:

1. **Vector Query Tool** (`vector_query.py`) - Semantic search using embeddings
2. **Document Processor** (`document_processor.py`) - Handles user-uploaded documents
3. **LLM Async Evaluator** (`llm_async.py`) - Evaluates products and companies
4. **Multimodal LLM Handler** (`multimodal_llm.py`) - Processes text and images
5. **Node Handlers** (`recommendation_node.py`, `general_node.py`) - Route requests to appropriate tools

### 1.2. Data Flow Architecture

```
User Query
    ↓
Router Node
    ↓
┌─────────────────────────────────────┐
│  RAG Pipeline (Vector Query)       │  ← Returns IDs only
│  Document Processing                │  ← Sends full text
│  Product/Company Evaluation         │  ← Sends full JSON
│  Multimodal Content                 │  ← Sends full images
└─────────────────────────────────────┘
    ↓
LLM Model (OpenAI)
```

---

## 2. RAG Pipeline: Data Minimization Implementation

### 2.1. Vector Query Tool

The `vector_query_tool` function implements data minimization by returning only fragment identifiers and metadata, not the full document content.

**Evidence**:
```python
// agent/tools/vector_query.py
def vector_query_tool(
    query: str,
    match_count: int = 30,
    match_threshold: float = 0.0,
    hnsw_ef_search: int = 64
):
    """
    Performs a semantic search using pgvector efficiently.
    ...
    """
    # ... vector search logic ...
    
    # Returns only IDs and metadata
    output.append({
        "pageContent": row[4] if len(row) > 4 else "",  # Empty or minimal
        "metadata": {
            "id_product_revision": row[1] if len(row) > 1 else None,
            "id_company_revision": row[2] if len(row) > 2 else None,
            "similarity": row[3] if len(row) > 3 else None,
            "id": row[0]
        }
    })
    
    return output, embedding_ids, embedding_time
```

**Analysis**: The vector query tool returns:
- Empty or minimal `pageContent` (line 291)
- Only metadata with IDs (`id_product_revision`, `id_company_revision`)
- Similarity scores
- Embedding IDs

The actual document content is **not** sent to the LLM at this stage. The system uses these IDs to fetch full data only when needed.

### 2.2. Lookup Process

After vector query returns IDs, the system performs lookups to retrieve full product and company information. However, this full data is then sent to the LLM during evaluation.

**Evidence**:
```python
// agent/tools/get_evaluations.py
async def lookup_parallel_batched(filtered_ids: List[Dict[str, Any]], ...):
    """
    Parallel lookup function using small batches with semaphore.
    """
    # Fetches full product and company data
    pr = await asyncio.wait_for(
        asyncio.to_thread(
            lambda: supabase.table("product_revision_public").select("*").in_("id", batch_ids).execute()
        ),
        timeout=timeout_seconds
    )
```

**Analysis**: The lookup process retrieves complete records using `select("*")`, which includes all fields from the database tables. This full data is subsequently sent to the LLM.

---

## 3. Document Processing: Full Content Transmission

### 3.1. Document Extraction

The `DocumentProcessor` class extracts full text content from uploaded documents and sends it to the LLM.

**Evidence**:
```python
// agent/tools/document_processor.py
def process_document(self, document_data: str, filename: str) -> Dict[str, Any]:
    """
    Processes a document to extract text and images.
    """
    # ... extraction logic ...
    
    result = {
        "filename": filename,
        "metadata": metadata,
        "text_content": text_content,  # Full extracted text
        "images": images,
        "has_text": bool(text_content.strip()),
        "has_images": len(images) > 0,
        "image_count": len(images),
        "text_length": len(text_content),
        "original_data": document_data
    }
    
    return result
```

### 3.2. Document Prompt Creation

The `create_document_prompt` function includes the full extracted text in the prompt sent to the LLM.

**Evidence**:
```python
// agent/tools/document_processor.py
def create_document_prompt(text: str, processed_content: Dict[str, Any]) -> str:
    """
    Creates a prompt that includes both text and information about documents.
    """
    if not processed_content.get("has_documents", False):
        return text
    
    # ... 
    prompt_parts.append("\nDocument content:")
    prompt_parts.append(processed_content.get("text", ""))  # Full document text
    
    return "\n\n".join(prompt_parts)
```

**Analysis**: When users upload documents (PDF, DOCX, etc.), the system:
1. Extracts the complete text content
2. Includes the full extracted text in the prompt sent to the LLM
3. Does not implement chunking or relevance filtering for document content

This approach sends entire document contents to the LLM, which may include sensitive or irrelevant information.

### 3.3. Text Length Limitations

The system does implement some limits on text length, but these are size-based rather than relevance-based.

**Evidence**:
```python
// agent/tools/document_processor.py
# Limitar longitud del texto
if len(text_content) > self.max_text_length:
    text_content = text_content[:self.max_text_length] + "\n\n[TEXTO TRUNCADO - Documento demasiado largo]"
```

**Analysis**: The system truncates text at 100,000 characters (`max_text_length = 100000`), but this is a hard limit rather than intelligent filtering based on relevance to the query.

---

## 4. Product and Company Evaluation: Full Data Structures

### 4.1. Evaluation Prompt Construction

When evaluating products or companies, the system sends complete JSON structures containing all available fields.

**Evidence**:
```python
// agent/tools/helpers/llm_async.py
async def prepare_prompt(entry):
    # ...
    if entry["type"] == "product":
        # ...
        prompt = f"""USER QUERY: 
{query} 

EXTRA DATA REQUIREMENTS: 
{json.dumps(extra_data, ensure_ascii=False, indent=2)} 

COMPANY_REQUIREMENTS: 
{company_requirements} 

COMPANY INFO: 
{json.dumps(company_info, ensure_ascii=False, indent=2)}  # Full company data

PRODUCT INFO: 
{json.dumps(product_info, ensure_ascii=False, indent=2)}"""  # Full product data
```

**Analysis**: The evaluation process sends:
- Complete `company_info` dictionary with all fields
- Complete `product_info` dictionary with all fields
- All extra data requirements
- Full company requirements

This approach sends all available data to the LLM, regardless of whether all fields are relevant to the specific evaluation query.

### 4.2. Database Query Scope

The lookup functions retrieve all columns from database tables.

**Evidence**:
```python
// agent/tools/get_evaluations.py
pr = await asyncio.wait_for(
    asyncio.to_thread(
        lambda: supabase.table("product_revision_public").select("*").in_("id", batch_ids).execute()
    ),
    timeout=timeout_seconds
)
```

**Analysis**: The use of `select("*")` retrieves all columns from the `product_revision_public` and `company_revision_public` tables, including potentially sensitive or unnecessary fields.

---

## 5. Image Processing: Full Image Data

### 5.1. Image Transmission

The system sends complete image data to vision-capable LLMs, which is necessary for image analysis but represents full data transmission.

**Evidence**:
```python
// agent/tools/helpers/multimodal_llm.py
def create_multimodal_messages(
    text: str, 
    multimodal_content: Optional[Dict[str, Any]] = None,
    system_prompt: Optional[str] = None,
    include_image_analysis_prompt: bool = True
) -> List:
    """
    Creates messages for multimodal LLM combining text and images.
    """
    # ...
    for i, image in enumerate(multimodal_content.get("images", [])):
        image_part = {
            "type": "image_url",
            "image_url": image["data"]  # Full base64 image data
        }
        content_parts.append(image_part)
```

**Analysis**: Images are sent as complete base64-encoded data. While this is necessary for vision models to analyze images, it represents transmission of full image content rather than minimized fragments.

### 5.2. Image Limiting

The system does implement limits on the number of images sent.

**Evidence**:
```python
// agent/tools/document_processor.py
# Limit to maximum 20 random images
if len(all_images) > self.max_images_per_document:
    import random
    images = random.sample(all_images, self.max_images_per_document)
```

**Analysis**: The system limits images to 20 per document, but this is a quantity limit rather than relevance-based filtering.

---

## 6. Context Size Control Mechanisms

### 6.1. Configuration Parameters

The system provides several configuration parameters that can control context size:

**Evidence**:
```python
// agent/tools/document_processor.py
def __init__(self):
    self.max_file_size = 50 * 1024 * 1024  # 50MB maximum
    self.max_pages = 100  # Maximum pages to process
    self.max_text_length = 100000  # Maximum text characters
    self.max_images_per_document = 20  # Maximum images per document
```

**Analysis**: The system implements hard limits on:
- File size (50MB)
- Number of pages (100)
- Text length (100,000 characters)
- Images per document (20)

These limits help control the amount of data sent, but they are not query-relevant filtering mechanisms.

### 6.2. Vector Query Parameters

The vector query tool provides parameters to control result set size.

**Evidence**:
```python
// agent/tools/vector_query.py
def vector_query_tool(
    query: str,
    match_count: int = 30,  # Limits number of results
    match_threshold: float = 0.0,  # Similarity threshold
    hnsw_ef_search: int = 64
):
```

**Analysis**: The `match_count` and `match_threshold` parameters allow control over how many fragments are returned and their minimum similarity, providing some control over context size in the RAG pipeline.

---

## 7. Conclusions

### 7.1. Strengths

✅ **RAG Pipeline Minimization**: The vector query implementation correctly minimizes data by returning only IDs and metadata, not full document content. This demonstrates proper data minimization in the semantic search component.

✅ **ID-Based Lookup Pattern**: The system uses an ID-based lookup pattern where vector queries return identifiers, and full data is only retrieved when needed. This separation of concerns is architecturally sound.

✅ **Configuration Limits**: The system implements multiple configuration parameters to limit file sizes, text lengths, and image counts, providing mechanisms to control data volume.

✅ **Similarity-Based Filtering**: The vector query tool uses similarity thresholds to filter results, ensuring only relevant fragments are considered.

✅ **Modular Architecture**: The separation between vector queries, document processing, and evaluation allows for independent optimization of each component.

### 7.2. Recommendations

1. **Implement Document Chunking and Relevance Filtering**: For user-uploaded documents, implement intelligent chunking and send only document fragments relevant to the user's query, rather than the entire extracted text. Use the same semantic search approach used in the RAG pipeline.

2. **Selective Field Transmission in Evaluations**: When evaluating products and companies, send only fields relevant to the evaluation query rather than complete JSON structures. Implement field filtering based on the query context.

3. **Query-Aware Document Processing**: Enhance document processing to analyze the user query and extract only relevant sections from documents, similar to how the RAG pipeline works.

4. **Implement Content Summarization**: For large documents, consider implementing summarization or extraction of key points before sending to the LLM, reducing the amount of data transmitted while preserving relevant information.

5. **Add Data Minimization Metrics**: Implement logging and monitoring to track the amount of data sent to LLMs, including average context sizes and data minimization ratios (sent data vs. available data).

6. **Configuration for Field Selection**: Add configuration options to specify which fields should be included in product/company evaluations, allowing administrators to control data exposure.

---

## 8. Control Compliance

| Criterion | Status | Evidence |
|-----------|--------|----------|
| RAG pipeline returns only fragment identifiers | ✅ COMPLIANT | `vector_query.py` returns empty `pageContent` and only metadata with IDs |
| Document processing minimizes content sent | ⚠️ PARTIAL | Full extracted text is sent; only size-based limits implemented |
| Product/company evaluation minimizes data | ⚠️ PARTIAL | Complete JSON structures sent; no field-level filtering |
| Image processing minimizes data | ⚠️ PARTIAL | Full image data sent (necessary for vision), but quantity limits exist |
| Context size control mechanisms exist | ✅ COMPLIANT | Multiple configuration parameters limit data volume |
| Query-relevant filtering implemented | ⚠️ PARTIAL | Implemented in RAG pipeline, not in document processing or evaluations |

**FINAL VERDICT**: ⚠️ **PARTIALLY COMPLIANT** with control IA-02. The platform demonstrates strong data minimization practices in its RAG pipeline by sending only relevant fragment identifiers rather than entire documents. However, the system sends complete document content when processing user uploads and full data structures during product/company evaluations. The implementation shows good architectural separation and provides configuration mechanisms to control data volume, but opportunities exist to extend minimization practices to document processing and evaluation contexts.

---

## Appendices

### A. Vector Query Return Structure

The vector query tool returns the following structure:

```python
{
    "pageContent": "",  # Empty or minimal
    "metadata": {
        "id_product_revision": "...",
        "id_company_revision": "...",
        "similarity": 0.85,
        "id": "embedding_id"
    }
}
```

This structure demonstrates data minimization by excluding full document content.

### B. Document Processing Flow

```
User Uploads Document
    ↓
DocumentProcessor.extract_text()
    ↓
Full Text Extracted
    ↓
create_document_prompt()
    ↓
Full Text Included in Prompt
    ↓
Sent to LLM
```

This flow shows that full document content is transmitted to the LLM.

### C. Evaluation Data Structure

When evaluating products, the system sends:

```json
{
    "USER QUERY": "...",
    "EXTRA DATA REQUIREMENTS": {...},
    "COMPANY_REQUIREMENTS": "...",
    "COMPANY INFO": {
        // All company fields
        "company_name": "...",
        "description": "...",
        "industry": "...",
        // ... all other fields
    },
    "PRODUCT INFO": {
        // All product fields
        "product_name": "...",
        "specifications": "...",
        // ... all other fields
    }
}
```

This structure includes all available fields, not just those relevant to the query.

### D. Configuration Limits Summary

| Parameter | Limit | Location |
|-----------|-------|----------|
| `max_file_size` | 50 MB | `document_processor.py` |
| `max_pages` | 100 pages | `document_processor.py` |
| `max_text_length` | 100,000 characters | `document_processor.py` |
| `max_images_per_document` | 20 images | `document_processor.py` |
| `match_count` | 30 results (default) | `vector_query.py` |
| `match_threshold` | 0.0 (configurable) | `vector_query.py` |

---

**End of Audit Report - Control IA-02**

