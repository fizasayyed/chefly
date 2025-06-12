# Chefly - Recipe Assistant

## Overview
Chefly is a conversational recipe assistant that uses RAG (Retrieval-Augmented Generation) to answer questions about recipes.

## System Architecture

### Setup
- Libraries are installed
- PDF path is set

### Data Ingestion & Preparation
- The recipe PDF is loaded (PyPDFLoader)
- Text is split into chunks (RecursiveCharacterTextSplitter)
- The SentenceTransformer model (intfloat/multilingual-e5-large) is loaded
- Qdrant client connects to the local Qdrant server, and two collections are created/recreated:
  - RecipeBot: Stores recipe chunks
  - ConversationMemory: Stores conversation history
- The E5EmbeddingsWrapper is defined to adapt the sentence transformer for LangChain
- Each recipe chunk is embedded using e5_embeddings and stored in the RecipeBot Qdrant collection

### LLM and Prompting Setup
- The ChatOllama LLM (mistral-small3.1) is initialized
- A detailed ChatPromptTemplate is created, instructing the LLM on:
  - Its role
  - How to use context and chat history
  - Desired response style

### Core RAG Components Initialization
- A Qdrant vector store (vectorstore) and retriever (e5_retriever) are set up for recipe documents
- Another Qdrant vector store (memory_vectorstore) and retriever (memory_retriever) are set up for conversation memory
- VectorStoreRetrieverMemory (memory) is initialized, using memory_retriever to store/retrieve conversation history
- The ConversationalRetrievalChain (qa_chain) is created, linking:
  - The LLM
  - The recipe retriever (e5_retriever)
  - The memory
  - The custom prompt

### Application Logic
- The RAGApplication class wraps the qa_chain and manages a simple list-based chat history

### User Interaction (Gradio UI)
When a user types a question in the Gradio UI and hits submit:
1. The chat function in Gradio is called
2. `rag_app.run(user_question)` is invoked
3. Inside `rag_app.run()`:
   - The RAGApplication's local chat history is formatted into LangChain messages
   - The qa_chain is called with the current question and this formatted history
   - Inside ConversationalRetrievalChain:
     1. **Rephrase Question** (if rephrase_question=True):
        - The memory loads relevant past turns
        - An LLM call rephrases the user's potentially ambiguous follow-up question into a standalone question
     2. **Retrieve Recipe Docs**:
        - The (rephrased) question is embedded
        - The e5_retriever fetches the top 3 similar recipe chunks from the RecipeBot Qdrant collection
     3. **Load Chat History** (for main prompt):
        - The memory loads relevant chat history from ConversationMemory Qdrant collection
     4. **Construct Prompt**:
        - The retrieved recipe chunks (context), loaded chat history, and user's current question are inserted into the ChatPromptTemplate
     5. **LLM Call**:
        - The final prompt is sent to mistral-small3.1 (via Ollama)
     6. **Get Answer**:
        - The LLM generates an answer
     7. **Save to Memory**:
        - The current question and the LLM's answer are saved to the VectorStoreRetrieverMemory
4. The `rag_app.run()` method returns the LLM's answer
5. The Gradio chat function updates its history and clears the input box
6. The Chatbot UI refreshes
