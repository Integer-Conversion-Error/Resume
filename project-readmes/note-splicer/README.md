# Note Summarization and RAG System

This project is designed to help users process, organize, and query their personal notes using advanced AI techniques. The primary goal is to transform unstructured text notes into a structured, searchable knowledge base, enabling efficient retrieval and summarization of information. This system is particularly useful for managing large volumes of personal study notes, research findings, or any text-based information that needs to be easily accessible and queryable.

The code provides a complete pipeline to:
1.  **Process Raw Text Notes**: Convert raw `.txt` files into a structured JSON format.
2.  **Structure Notes with Generative AI**: Utilize a generative AI model to enhance the structure and organization of the notes.
3.  **Build a Searchable Vector Database**: Create a persistent vector database from the structured notes, allowing for semantic search.
4.  **Query Data with Retrieval-Augmented Generation (RAG)**: Implement a RAG system that retrieves relevant information from the vector database and generates coherent answers using a Large Language Model (LLM).

## Project Structure

-   `note_processor.py`: Processes raw `.txt` notes into structured JSON files in the `Generated_Notes/` directory.
-   `splice_json.py`: Compiles the individual JSON files into a single master file, `Spliced_Notes.json`.
-   `create_vector_db.py`: Reads the master JSON file and builds a persistent ChromaDB vector database in the `vector_db/` directory.
-   `query_rag.py`: The main entry point for the RAG system. Takes a user query, retrieves relevant notes from the vector DB, and generates an answer using an LLM.
-   `requirements.txt`: Lists all Python dependencies.
-   `structure.json` / `spliced_notes_schema.json`: Schemas defining the data structures.

## How to Use the RAG System (Personal Train of Thought Note)

Since you have already run the initial setup and created the vector database, you can directly proceed to querying the system.

### Step 1: Install Dependencies (If you haven't already)

Ensure all dependencies, including the newly added `litellm`, are installed in your virtual environment.

```bash
# From the root directory of the project
./.venv/Scripts/pip install -r requirements.txt
```

### Step 2: Query Your Notes

To ask a question to your RAG system, run the `query_rag.py` script from the command line, passing your question as an argument.

```bash
# From the root directory of the project
./.venv/Scripts/python query_rag.py "Your question about the notes here"
```

**Example:**

```bash
./.venv/Scripts/python query_rag.py "What are the key differences between mitosis and meiosis?"
```

### Using a Different Language Model (e.g., Deepseek)

The `query_rag.py` script is configured to be easily adaptable. To use a local model like Deepseek (assuming it's served via a tool like Ollama), you would modify the `litellm.completion` call inside the script:

**Change this:**

```python
# For this example, we will use a widely available free model (Groq)
# to demonstrate the functionality. You can easily swap this out.
# NOTE: You may need to set the GROQ_API_KEY environment variable.
response = litellm.completion(
    model="groq/llama3-8b-8192", 
    messages=[{"content": prompt, "role": "user"}]
)
```

**To this (example for Deepseek on Ollama):**

```python
# Make sure your local Ollama server is running.
response = litellm.completion(
    model="ollama/deepseek-coder", # Or the specific model name you are using
    messages=[{"content": prompt, "role": "user"}],
    api_base="http://localhost:11434"
)
```

---

## Full Workflow (For Reference)

If you ever need to rebuild the entire database from scratch, the full workflow is:

1.  **Process Raw Notes**: `python note_processor.py`
2.  **Compile Master JSON**: `python splice_json.py`
3.  **Build Vector DB**: `python create_vector_db.py`
