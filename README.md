
# CampusGenie: My RAG Application Development Journey

This repository documents my journey building and improving a Retrieval-Augmented Generation (RAG) system over time. The project demonstrates my expertise in building production-ready AI applications that combine vector search, LLM integration, and database management.

## Project Overview

CampusGenie is an AI-powered question that uses Retrieval-Augmented Generation to provide accurate, contextual responses. The application retrieves relevant information from a knowledge base and uses OpenAI's models to generate comprehensive answers.

### Core Technologies
- **OpenAI API**: Powers the language generation capabilities
- **Pinecone**: Vector database for semantic search and retrieval
- **MySQL**: Relational database for user management, conversation history, and rate limiting
- **Streamlit**: Frontend framework for the user interface
- **Tiktoken**: Token counting for accurate API usage tracking

## System Architecture

The system consists of two main components:
1. **Document Processing Pipeline**: Ingests documents, chunks them, generates embeddings, and stores them in Pinecone
2. **Query Engine**: Handles user queries, performs similarity search, and generates contextually relevant responses

### Architecture Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Document       │────▶│  Embedding      │────▶│  Vector         │
│  Processing     │     │  Generation     │     │  Database       │
│                 │     │                 │     │  (Pinecone)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        │
                                                        ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  User           │────▶│  Query          │────▶│  LLM            │
│  Interface      │     │  Processing     │     │  (OpenAI)       │
│  (Streamlit)    │◀────│                 │◀────│                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## First Implementation: CampusGenie Legacy

The first implementation of CampusGenie established the foundation for the entire system. Here's a breakdown of the key components:

### Query Engine (campus_genie.py)

```python
# Core functionality:
# 1. User authentication system
# 2. Token management and rate limiting
# 3. Context retrieval from Pinecone vector DB
# 4. LLM integration with OpenAI
# 5. Conversation management
```

#### Key Features

- **Authentication System**: Secure login using email and password
- **Token Management**: Tracks and limits token usage per user
- **Rate Limiting**: Implements question rate limiting (25 questions per 3-hour window)
- **Vector Search**: Retrieves relevant context using similarity search with a configurable threshold
- **Conversation History**: Maintains conversation context for better responses
- **Source Attribution**: Displays sources of information used in responses
- **Error Handling**: Robust error handling for API rate limits and connection issues

#### Technical Highlights

1. **Efficient Context Retrieval**:
   ```python
   # Pinecone similarity search with threshold filtering
   relevant_matches = [item for item in res.matches if item.score >= similarity_threshold]
   ```

2. **Token Optimization**:
   ```python
   # Token counting for both user input and model responses
   question_token_count = num_tokens_from_messages(
       [{"role": "user", "content": prompt}], st.session_state["openai_model"]
   )
   ```

3. **Asynchronous Processing**:
   ```python
   # Asynchronous API calls for better performance
   async for chunk in handle_requests(
       {"role": "user", "content": augmented_query},
       messages,
       st.session_state["openai_model"],
       temperature=0.2,
   ):
   ```

4. **Database Connection Pooling**:
   ```python
   # Connection pooling for efficient DB access
   db = get_connection()
   ```

### Document Processing Pipeline (llama_parse_integration.py)

The document processing pipeline handles:
1. Document ingestion from various sources
2. Text extraction and chunking
3. Embedding generation using OpenAI's embedding models
4. Storage in Pinecone vector database for efficient retrieval

## Evolution and Improvements

This section will document the improvements made to the system over time, showcasing my growth as a developer and the evolution of the application.

### Version 2: [Future Implementation]

_This section will be updated with details about the improvements made in Version 2._

### Version 3: [Future Implementation]

_This section will be updated with details about the improvements made in Version 3._

## Lessons Learned

Throughout the development of CampusGenie, I've gained valuable insights and expertise:

1. **Effective Vector Database Management**: Optimizing vector search for accurate and performant retrieval
2. **Token Economy**: Managing the balance between context length and API costs
3. **User Experience Design**: Creating intuitive interfaces for AI-powered applications
4. **System Scalability**: Building systems that can handle increasing user load
5. **Security Best Practices**: Implementing proper authentication and data protection

## Future Directions

My roadmap for future improvements includes:

1. Implementing a more sophisticated chunking strategy
2. Exploring hybrid search approaches (combining sparse and dense retrievers)
3. Adding support for multi-modal content (images, audio)
4. Implementing fine-tuned models for domain-specific knowledge
5. Enhancing the analytics dashboard for better insights

## Skills Demonstrated

This project showcases my expertise in:

- **RAG System Architecture**: Designing and implementing production-ready RAG systems
- **Vector Databases**: Efficient use of vector databases for semantic search
- **LLM Integration**: Working with state-of-the-art language models
- **Full-Stack Development**: From database design to user interface
- **Asynchronous Programming**: Building responsive applications with async capabilities
- **System Optimization**: Managing token usage, rate limits, and performance bottlenecks
- **Security Implementation**: User authentication and data protection

---

_This documentation will be continuously updated as the project evolves._
