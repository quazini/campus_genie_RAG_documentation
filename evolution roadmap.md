## Version X: [Campus Genie Version 1]

**Release Date:** [September 2024]

### Key Improvements
- [No key improvements, this is version 1]

### Technical Enhancements
1. [No key enhancements, this is version 1]


### Implementation Details

[                           
# sample code for the RAG query 
# 🧠 Initialize Pinecone Index and Retrieve Relevant Information

```python
import os
import logging
from pinecone import Pinecone, ServerlessSpec

# Get Pinecone API key from environment variables
pinecone_api_key = os.environ.get("PINECONE_API_KEY")

# Set a similarity threshold
similarity_threshold = 0.7  # Adjust this value based on your requirements

if pinecone_api_key:
    # Initialize Pinecone client
    pc = Pinecone(api_key=pinecone_api_key)

    # Get cloud and region, use defaults if not set
    cloud = os.environ.get("PINECONE_CLOUD") or "aws"
    region = os.environ.get("PINECONE_REGION") or "us-east-1"
    spec = ServerlessSpec(cloud=cloud, region=region)

    # Define embedding model and index name
    embed_model = "text-embedding-ada-002"
    index_name = "campus-genie"

    # Create the index if it doesn't exist
    if index_name not in pc.list_indexes().names():
        pc.create_index(
            name=index_name,
            dimension=1536,
            metric="cosine",
            spec=spec,
        )

    # Connect to the index
    index = pc.Index(index_name)
else:
    logging.error("Pinecone API key not found. Pinecone search functionality will be disabled.")

# Retrieve relevant information from Pinecone
if pinecone_api_key:
    # Create embedding for the input prompt
    response = await client.embeddings.create(
        input=[prompt], model=embed_model
    )
    query_embedding = response.data[0].embedding

    # Query the Pinecone index
    res = index.query(
        vector=query_embedding,
        top_k=3,
        include_metadata=True,
    )

    # Filter the results based on the similarity threshold
    relevant_matches = [item for item in res.matches if item.score >= similarity_threshold]

    if relevant_matches:
        contexts = []
        sources = set()

        for item in relevant_matches:
            contexts.append(item.metadata["chunk"])
            source = item.metadata.get("source")
            if source:
                sources.add(source)

            # Print the metadata for each relevant match
            logging.info(f"Metadata for match {item.id}:")
            for key, value in item.metadata.items():
                logging.info(f"{key}: {value}")
            logging.info("---")

        # Combine relevant chunks with the prompt
        augmented_query = "\n\n---\n\n".join(contexts) + "\n\n-----\n\n" + prompt
        source_info = "Sources: " + ", ".join(sources)
    else:
        augmented_query = prompt
        source_info = ""

    logging.info(f"Augmented query: {augmented_query}")

```
                  
]

### Performance Metrics
[no comparisons for now]

### Lessons Learned
[## Lessons Learned from Creating a RAG Application

1. **Data Quality is Everything**  
   Clean, structured, and well-chunked documents significantly improve retrieval relevance and the quality of generated responses.

2. **Chunking Strategy Matters**  
   Overlapping and context-aware chunking helps preserve semantic meaning and improves the accuracy of retrieved data.

3. **Embedding Model Selection Impacts Accuracy**  
   Choosing the right embedding model (e.g., `text-embedding-ada-002`) influences both performance and cost. The best model depends on your specific use case.

4. **Similarity Threshold Tuning is Crucial**  
   Setting an appropriate similarity threshold helps filter out irrelevant matches while retaining useful ones. Test with real prompts to find the best value.

5. **Vector Database Design Affects Scalability**  
   Proper configuration of the vector index (dimensions, metric, region, etc.) ensures good performance and cost efficiency, especially at scale.

6. **Augmented Query Quality Drives Model Output**  
   The structure and clarity of the augmented query (context + prompt) directly influence the quality of the generated answers.

7. **Secure Environment Configs**  
   Use environment variables for sensitive information like API keys. Avoid hardcoding credentials in your codebase.

8. **Logging and Observability Help Debug Quickly**  
   Logging retrieved chunks, scores, and metadata is essential for diagnosing issues and optimizing system performance.

9. **Async Handling Enhances Performance**  
   Use asynchronous calls for embedding and querying to keep your application responsive, especially under load.

10. **Continuous Evaluation is Needed**  
   Regularly evaluate retrieval accuracy and generation quality. As your data or user behavior changes, your system should evolve accordingly.
]