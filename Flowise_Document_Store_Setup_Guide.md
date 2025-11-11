# Flowise Document Store & Chatflow Setup Guide

A simple guide to set up a Document Store with PostgreSQL vector database and create a RAG (Retrieval Augmented Generation) chatflow in Flowise.

---

## Prerequisites

Before starting, ensure you have:
- Flowise installed and running
- PostgreSQL installed (version 12 or higher)
- Ollama installed and running (for local LLM models)
- A fine-tuned chat model (or use any Ollama model)

---

## Part 1: Install pgvector Extension

### Step 1: Install pgvector

**On macOS (with Homebrew):**
```bash
brew install pgvector
```

**On Ubuntu/Debian:**
```bash
sudo apt install postgresql-XX-pgvector
# Replace XX with your PostgreSQL version (e.g., 15, 16)
```

**Using Docker (Easiest):**
```bash
docker run -d \
  --name postgres-pgvector \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=flowise \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### Step 2: Restart PostgreSQL

**On macOS:**
```bash
brew services restart postgresql
```

**On Ubuntu/Debian:**
```bash
sudo systemctl restart postgresql
```

**Docker:**
```bash
docker restart postgres-pgvector
```

### Step 3: Enable pgvector Extension

Connect to your PostgreSQL database using pgAdmin, psql, or any SQL client:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Verify installation:
```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```

If you see a row returned, pgvector is successfully enabled.

---

## Part 2: Install Ollama Models

### Step 1: Install Embedding Model

You need a dedicated embedding model (NOT your chat model):

```bash
# Recommended for technical/cybersecurity content
ollama pull nomic-embed-text
```

**Alternative embedding models:**
```bash
# More accurate but slower
ollama pull mxbai-embed-large

# Fastest option
ollama pull all-minilm
```

### Step 2: Verify Your Chat Model

Check if your fine-tuned model is available:

```bash
ollama list
```

You should see your model (e.g., `f5-cyberguru:latest`).

### Step 3: Keep Ollama Running

Start Ollama server (keep this terminal open):

```bash
ollama serve
```

---

## Part 3: Create PostgreSQL Credentials in Flowise

### Step 1: Navigate to Credentials

In Flowise UI:
1. Click **"Credentials"** in the left sidebar
2. Click **"Add Credential"**
3. Select **"Postgres"** from the list

### Step 2: Configure Database Connection

Fill in your PostgreSQL connection details:
- **Credential Name**: `flowise-postgres-vdb` (or any name)
- **Host**: `localhost` (or your server address)
- **Port**: `5432`
- **Database**: `postgres` (or your database name)
- **Username**: `postgres` (or your username)
- **Password**: Your PostgreSQL password

### Step 3: Save Credential

Click **"Save"** - this credential will be used in both Document Store and Chatflow.

---

## Part 4: Create Document Store

### Step 1: Navigate to Document Stores

1. Click **"Document Stores"** in the left sidebar
2. Click **"Add New Document Store"**

### Step 2: Configure Document Store

**Store Settings:**
- **Store Name**: `threat-analysis-kb` (or any descriptive name)
- **Description**: "F5 cybersecurity threat intelligence knowledge base"

**Vector Store Configuration:**
- **Vector Store Type**: Select **"Postgres"**
- **Credential**: Select `flowise-postgres-vdb` (from Part 3)
- **Table Name**: `flowise_vectors` (or any name - Flowise will create it)

**Embeddings Configuration:**
- **Embedding Type**: Select **"Ollama Embeddings"**
- **Base URL**: `http://localhost:11434`
- **Model Name**: `nomic-embed-text` (NOT your chat model!)

### Step 3: Save Document Store

Click **"Create"** to save the document store configuration.

---

## Part 5: Upload Documents to Store

### Step 1: Select Document Store

1. Go to **"Document Stores"** in the left sidebar
2. Click on your newly created store (`threat-analysis-kb`)

### Step 2: Upload Documents

1. Click **"Upload Document"** or **"Insert into Store"**
2. Select your document type:
   - **Text files** (.txt, .md)
   - **PDF files**
   - **Word documents** (.docx)
   - **CSV files**
   - **JSON files**
3. Click **"Upload"** and select your files
4. Click **"Process"** or **"Insert"**

### Step 3: Wait for Processing

- The system will chunk your documents
- Generate embeddings using `nomic-embed-text`
- Store vectors in the PostgreSQL database
- This may take a few minutes depending on document size

### Step 4: Verify Upload

Check your PostgreSQL database to confirm documents were stored:

```sql
SELECT COUNT(*) FROM flowise_vectors;
```

You should see the number of document chunks stored.

---

## Part 6: Create RAG Chatflow

### Step 1: Create New Chatflow

1. Click **"Chatflows"** in the left sidebar (NOT Agentflows)
2. Click **"Add New"**
3. Give it a name: "F5 Threat Intelligence Assistant"

### Step 2: Add Required Nodes

You need 4 nodes total. Search and drag these onto the canvas:

**Node 1: Ollama Embeddings**
- **Base URL**: `http://localhost:11434`
- **Model Name**: `nomic-embed-text`

**Node 2: Postgres (Vector Store)**
- **Connect Credential**: Select `flowise-postgres-vdb`
- **Table Name**: `flowise_vectors` (same as Document Store)
- **Database**: `postgres`

**Node 3: ChatOllama (Your LLM)**
- **Base URL**: `http://localhost:11434`
- **Model Name**: `f5-cyberguru` (or your model name)
- **Temperature**: `0.2` (low temperature reduces hallucination)
- **Top P**: `0.6` (optional, for more focused responses)

**Node 4: Conversational Retrieval QA Chain**
- This is the main chain that connects everything

### Step 3: Connect the Nodes

Make these connections:

```
[Ollama Embeddings] 
       ↓
   (connect to Embeddings input)
       ↓
[Postgres Vector Store] ────► [Conversational Retrieval QA Chain]
                                        ↑
                                        │
                          (in node settings, select Chat Model)
                                        │
                              [ChatOllama: f5-cyberguru]
```

**Detailed connections:**
1. Connect **Ollama Embeddings** output → **Postgres** Embeddings input
2. Connect **Postgres** output → **Conversational Retrieval QA Chain** Vector Store input
3. Click on **Conversational Retrieval QA Chain** node
4. In the node settings, select **ChatOllama** as the Chat Model

### Step 4: Optional - Add Memory

For conversation history:
1. Add a **Buffer Memory** or **Conversation Buffer Memory** node
2. Connect it to the **Memory** input of the Conversational Retrieval QA Chain

### Step 5: Save and Test

1. Click **"Save"** in the top right
2. Click the **"Chat"** button to test your chatflow
3. Ask questions about your documents

---

## Part 7: Testing Your Setup

### Test Questions to Try:

1. **Simple retrieval**: "What threats are mentioned in the documents?"
2. **Specific query**: "What are the recommended security controls for ransomware?"
3. **Complex analysis**: "Summarize the top 3 threats and their mitigation strategies"

### Expected Behavior:

- The system searches your document store for relevant information
- Retrieved context is passed to your fine-tuned model
- Your model generates responses based on the retrieved documents
- Responses should be factual and based on your documents (not hallucinated)

---

## Troubleshooting Common Issues

### Issue 1: "extension 'vector' is not available"

**Solution**: pgvector not installed properly
- Reinstall pgvector for your PostgreSQL version
- Restart PostgreSQL
- Run `CREATE EXTENSION vector;` again

### Issue 2: "column 'id' does not exist" or "column 'embedding' does not exist"

**Solution**: Table has wrong schema
```sql
DROP TABLE IF EXISTS flowise_vectors CASCADE;
```
- Restart Flowise
- Try inserting documents again - Flowise will recreate the table correctly

### Issue 3: "model does not support tools" (in Agentflows)

**Solution**: You're in the wrong flow type
- Use **Chatflows** (not Agentflows) for RAG with fine-tuned models
- Agentflows require models with tool/function calling support

### Issue 4: Ollama connection errors

**Solution**: Ollama not running
```bash
# Start Ollama
ollama serve

# Verify it's running
curl http://localhost:11434
```

### Issue 5: Using chat model for embeddings

**Symptom**: Slow processing, errors, or incorrect dimensions

**Solution**: 
- Make sure Ollama Embeddings uses `nomic-embed-text`
- Your chat model (f5-cyberguru) should ONLY be used in ChatOllama node
- These are separate models for different purposes!

### Issue 6: High hallucination in responses

**Solution**: Lower the temperature
- Set Temperature to `0.1` - `0.3` in ChatOllama node
- Lower temperature = more factual, less creative
- Higher temperature = more creative, more hallucination

---

## Key Concepts to Remember

### Embedding Model vs Chat Model

**Embedding Model** (`nomic-embed-text`):
- Converts text to vectors for semantic search
- Used to store documents and search queries
- Fast and specialized for similarity matching

**Chat Model** (`f5-cyberguru` or any LLM):
- Generates human-like responses
- Receives retrieved documents as context
- Your fine-tuning applies here!

### How RAG Works

1. User asks a question
2. Question is converted to vector using embedding model
3. Similar document chunks are retrieved from vector database
4. Retrieved context + question are sent to chat model
5. Chat model generates response based on provided context
6. Result: Factual answers grounded in your documents

### Temperature Settings for RAG

- **0.0 - 0.2**: Very deterministic, sticks closely to context (best for technical/factual)
- **0.3 - 0.5**: Balanced between accuracy and natural language
- **0.6 - 1.0**: More creative, higher risk of hallucination (not recommended for RAG)

**Recommendation for cybersecurity**: Use 0.1 - 0.3

---

## Summary Checklist

- [ ] PostgreSQL installed with pgvector extension enabled
- [ ] Ollama installed with `nomic-embed-text` embedding model
- [ ] Ollama chat model available (fine-tuned or standard)
- [ ] PostgreSQL credentials created in Flowise
- [ ] Document Store created with correct embedding model
- [ ] Documents uploaded and processed successfully
- [ ] Chatflow created with all 4 nodes properly connected
- [ ] Temperature set to 0.2 in ChatOllama node
- [ ] Chatflow tested with sample questions
- [ ] Responses are factual and based on documents

---

## Next Steps

Once your setup is working:

1. **Add more documents**: Upload additional threat intelligence reports
2. **Refine prompts**: Add system messages to guide model behavior
3. **Add memory**: Enable conversation history for multi-turn interactions
4. **Create API**: Use Flowise's built-in API endpoint for integration
5. **Monitor performance**: Check execution logs for retrieval quality
6. **Tune parameters**: Adjust chunk size, overlap, and retrieval count

---

## Additional Resources

- **Flowise Documentation**: https://docs.flowiseai.com
- **pgvector GitHub**: https://github.com/pgvector/pgvector
- **Ollama Models**: https://ollama.ai/library
- **RAG Best Practices**: Focus on document quality, chunk size, and embedding model selection

---

*Document created based on hands-on setup experience with F5 cybersecurity threat intelligence system using fine-tuned Llama 3.1 model.*
