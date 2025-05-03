# CampusGenie Technical Documentation

This document provides detailed technical explanations of implementation choices, code architecture, and system design for the CampusGenie RAG application.

## System Components

### 1. Query Engine (`campus_genie.py`)

The query engine is the core of the application, processing user input and generating responses with contextual information.

#### Authentication Flow

```python
# User authentication before accessing the application
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False

if not st.session_state.logged_in:
    st.subheader("Login")
    username = st.text_input("email")
    password = st.text_input("Password", type="password")
    if st.button("Login"):
        if authenticate_user(username, password):
            st.session_state.logged_in = True
            st.session_state.username = username
            st.rerun()
        else:
            st.error("Invalid email or password")
```

The authentication system ensures that only registered users can access the system, allowing for:
- User-specific token management
- Personalized conversation history
- Usage tracking and analytics

#### Token Management System

The token management system tracks token usage to:
1. Control costs associated with API usage
2. Prevent abuse of the system
3. Ensure fair allocation of resources among users

```python
# Check if user has sufficient tokens
current_token_count = get_remaining_tokens(st.session_state["user_id"])
if current_token_count < question_token_count + max_response_tokens:
    st.error(
        f"Insufficient tokens. You need at least {question_token_count + max_response_tokens} tokens for this request, but you only have {current_token_count} tokens available. Please purchase more tokens to continue."
    )
```

#### Rate Limiting Implementation

To prevent abuse and ensure system stability, a rate limiting mechanism restricts the number of questions a user can ask within a specific time window:

```python
# Rate limiting constants
MAX_QUESTIONS_PER_USER = 25
QUESTION_WINDOW_HOURS = 3
COOLDOWN_HOURS = 2

# Rate limiting check
query = """
    SELECT COUNT(*) AS question_count
    FROM ratelimit_questions
    WHERE user_id = %s
    AND created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
"""
cursor.execute(query, (user_id, QUESTION_WINDOW_HOURS))
result = cursor.fetchone()
question_count = result[0]

if question_count >= MAX_QUESTIONS_PER_USER:
    st.error(
        f"You have reached the maximum of {MAX_QUESTIONS_PER_USER} questions within the {QUESTION_WINDOW_HOURS}-hour window. Please wait for {COOLDOWN_HOURS} hours before asking more questions."
    )
```

#### Vector Search and Retrieval

The retrieval component uses Pinecone to find semantically similar content to the user's query:

```python
# Vector search with Pinecone
response = await client.embeddings.create(
    input=[prompt], model=embed_model
)
query_embedding = response.data[0].embedding
res = index.query(
    vector=query_embedding,
    top_k=3,
    include_metadata=True,
)

# Filter results based on similarity threshold
relevant_matches = [item for item in res.matches if item.score >= similarity_threshold]
```

Key design decisions:
- Using a similarity threshold (0.7) to filter out less relevant matches
- Retrieving the top 3 most similar chunks to provide sufficient context
- Including source information for attribution and transparency

#### Context Augmentation

The system augments the user's query with retrieved context:

```python
if relevant_matches:
    contexts = []
    sources = set() 
    
    for item in relevant_matches:
        contexts.append(item.metadata["chunk"])
        source = item.metadata.get("source")
        if source:
           sources.add(source)

    augmented_query = "\n\n---\n\n".join(contexts) + "\n\n-----\n\n" + prompt
    source_info = "Sources: " + ", ".join(sources)
else:
    augmented_query = prompt
    source_info = ""
```

#### Streaming Response

To improve user experience, responses are streamed in real-time:

```python
async for chunk in handle_requests(
    {"role": "user", "content": augmented_query},
    messages,
    st.session_state["openai_model"],
    temperature=0.2,
):
    content = chunk.choices[0].delta.content
    if content:
        full_response += content
        formatted_response = process_model_response(
            full_response + "▌"
        )
        message_placeholder.markdown(
            formatted_response
        )
```

### 2. Document Processing Pipeline

The document processing pipeline is responsible for:
1. Ingesting documents from various sources
2. Chunking content into manageable pieces
3. Generating embeddings for each chunk
4. Storing chunks and their embeddings in the vector database

#### Chunking Strategy

Documents are broken down into chunks with:
- Appropriate size for context retrieval (typically 512-1024 tokens)
- Semantic coherence preserved where possible
- Overlap between chunks to prevent information loss at boundaries

#### Metadata Management

Each chunk is stored with metadata including:
- Source document name
- Page number (for multi-page documents)
- Section information
- Creation timestamp

This metadata enables source attribution and more targeted retrieval.

## Database Schema

### User Management

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    token_count INT DEFAULT 0,
    overflow_tokens INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Chat Sessions

```sql
CREATE TABLE chat_sessions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Conversation History

```sql
CREATE TABLE conversations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    session_id INT NOT NULL,
    user_id INT NOT NULL,
    role ENUM('user', 'assistant') NOT NULL,
    content TEXT NOT NULL,
    token_count INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES chat_sessions(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Rate Limiting

```sql
CREATE TABLE ratelimit_questions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    question TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## Error Handling and Resilience

The system implements robust error handling:

### API Rate Limit Handling

```python
async def process_request(
    prompt_obj, messages, model, temperature=0.2, max_retries=5, initial_delay=1
):
    retry_count = 0
    while retry_count < max_retries:
        try:
            # API call code
            return chunks
        except Exception as e:
            if "Rate limit" in str(e):
                retry_count += 1
                delay = initial_delay * (2 ** (retry_count - 1))
                delay = min(delay, 60)  # Cap the delay at 60 seconds
                delay = delay + random.uniform(0, 1)  # Add jitter to the delay
                logging.warning(
                    f"Rate limit exceeded. Retrying in {delay} seconds. Retry count: {retry_count}"
                )
                await asyncio.sleep(delay)
            else:
                logging.error(f"Error processing request: {str(e)}")
                return None
    logging.error("Max retries exceeded. Request failed.")
    return None
```

This implementation uses:
- Exponential backoff with jitter for retries
- Maximum retry count to prevent infinite loops
- Logging for monitoring and debugging

### Database Connection Management

The application uses a connection pool for efficient database access:

```python
# Connection pooling is handled in utils.py
from mysql.connector.pooling import MySQLConnectionPool

def get_connection():
    try:
        conn = connection_pool.get_connection()
        return conn
    except Exception as e:
        logging.error(f"Error getting connection from pool: {str(e)}")
        return None
```

Connection pooling provides:
- More efficient resource utilization
- Faster query execution
- Better handling of concurrent requests

## Performance Optimizations

### Token Usage Optimization

The system carefully tracks token usage to minimize costs:

```python
# Calculate token count for user's question
question_token_count = num_tokens_from_messages(
    [{"role": "user", "content": prompt}], st.session_state["openai_model"]
)

# Estimate maximum response tokens
max_response_tokens = 2000

# Check if sufficient tokens are available
if current_token_count < question_token_count + max_response_tokens:
    st.error(
        f"Insufficient tokens. You need at least {question_token_count + max_response_tokens} tokens..."
    )
```

### Conversation Context Management

To manage context window limitations, the system only uses recent conversation history:

```python
# Use only the last two conversation history messages
if len(st.session_state.messages) > 3:
    recent_messages = st.session_state.messages[-3:]
else:
    recent_messages = st.session_state.messages
```

This approach:
- Reduces token usage
- Maintains conversational coherence
- Prevents context window overflow

## Security Considerations

### Authentication

User authentication is handled securely with password hashing and account management. The `authenticate_user` function in `utils.py` validates credentials against the database.

### Data Protection

All database interactions use parameterized queries to prevent SQL injection:

```python
query = """
    SELECT COUNT(*) AS question_count
    FROM ratelimit_questions
    WHERE user_id = %s
    AND created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
"""
cursor.execute(query, (user_id, QUESTION_WINDOW_HOURS))
```

### API Key Management

API keys are loaded from environment variables for security:

```python
from dotenv import load_dotenv
load_dotenv()

client = AsyncOpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
pinecone_api_key = os.environ.get("PINECONE_API_KEY")
```

## Testing Methodology

The application should be tested with:

1. **Unit Tests**: For individual components like token counting, authentication, etc.
2. **Integration Tests**: For database interactions, API calls, etc.
3. **End-to-End Tests**: For complete user flows
4. **Load Testing**: To ensure the system can handle concurrent users

## Deployment Architecture

### Recommended Deployment Stack

- **Frontend**: Streamlit hosted on a cloud platform
- **Backend**: Python application with asyncio
- **Database**: MySQL on a managed service (RDS, Cloud SQL)
- **Vector Database**: Pinecone serverless
- **Monitoring**: Logging infrastructure with alerts
- **CI/CD**: Automated testing and deployment pipeline

### Scaling Considerations

- Horizontal scaling for the application servers
- Database read replicas for high-traffic scenarios
- Caching frequently accessed data
- Optimizing expensive operations (embedding generation)