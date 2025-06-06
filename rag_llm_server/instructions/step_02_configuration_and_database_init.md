# Step 2: Configuration and Database Initialization #
   
[Reasoning behind the code](reasoning/r2.md)
   
1.  **Install Dependencies:**
    ```bash
    pip install pydantic-settings python-dotenv sqlalchemy "databases[sqlite]" chromadb
    ```
    *   `pydantic-settings`: For handling application settings using Pydantic models.
    *   `python-dotenv`: To load environment variables from a `.env` file (used automatically by `pydantic-settings`).
    *   `sqlalchemy`: A powerful SQL toolkit and Object-Relational Mapper (ORM). We'll use its Core component initially.
    *   `databases[sqlite]`: An async database access library that works well with SQLAlchemy Core and FastAPI's async nature. The `[sqlite]` part installs the necessary driver (`aiosqlite`).
    *   `chromadb`: The vector database client library.

2.  **Create Project Structure:**
    Let's organize our project better:
    ```
    rag_llm_server/
    ├── app/
    │   ├── __init__.py
    │   └── main.py         # <--- Move main.py here
    ├── core/
    │   ├── __init__.py
    │   └── config.py       # <--- New: Settings definition
    ├── db/
    │   ├── __init__.py
    │   └── database.py     # <--- New: DB connection/setup logic
    │   └── models.py       # <--- New: DB table definitions (SQLAlchemy Metadata)
    ├── .env                # <--- New: Environment variables (optional)
    ├── .gitignore
    └── venv/               # Or your virtual environment name
    ```
    *   Move the existing `main.py` into the `app/` directory.
    *   Create empty `__init__.py` files in `app/`, `core/`, and `db/` to make them Python packages.

---

   ## For testing purposes, we are implmenting Swagger UI.
   
   ### What is Swagger UI?
   
   **Swagger UI** is a **web-based interface** that lets you:
   - Visualize your API endpoints
   - Read what each endpoint does
   - **Send requests** and see responses **without writing code**
   
   It’s like Postman, but built into your app. If you're building an API and want a "try it out" button for every route, Swagger UI gives you that—automatically.
   
   ---
   
   ### Why It’s Awesome for Development
   
   1. **Live testing** without Postman or writing curl commands.
   2. **Auto-generated docs** from your FastAPI route definitions.
   3. **Data validation** shown clearly (you’ll see the expected request and response models).
   4. Makes it easy to **debug** and see if your API is working as expected.
   
   ---
   
   ### Where does it come from in FastAPI?
   
   FastAPI **automatically generates** Swagger UI when it sees your route definitions and Pydantic models.
   
   By default, once your server is running, go to:
   ```
   http://localhost:8000/docs
   ```
   
   That URL is the Swagger UI!
   
   ---
   
   ### How Swagger UI Works in Your Application
   
   Let’s say you’re building a chat app backend. Here’s what Swagger UI does for **you**:
   
   #### Location:
   In `main.py`, when you define routes like this:
   
   ```python
   app.include_router(
       sessions.router,
       prefix="/api/v1/sessions",
       tags=["Chat Sessions"]
   )
   ```
   
   FastAPI:
   - Reads the endpoints from `sessions.py`
   - Reads the `summary`, `response_model`, and `status_code` from each route
   - Uses Pydantic models (`SessionCreateRequest`, `ChatMessageResponse`, etc.)
   - Auto-generates docs for each route
   
   #### Example in Action
   
   If you go to `http://localhost:8000/docs`, you’ll see a section called **Chat Sessions**. That’s because of the `tags=["Chat Sessions"]`.
   
   Inside it, you’ll see:
   - `POST /api/v1/sessions/` → Creates a new session  
   - `GET /api/v1/sessions/` → Lists all sessions  
   - `POST /api/v1/sessions/{session_id}/messages` → Adds a message  
   - `GET /api/v1/sessions/{session_id}/messages` → Lists messages  
   - `DELETE /api/v1/sessions/{session_id}` → Deletes a session
   
   #### Try It Out
   
   For each endpoint, there’s a big blue button: **“Try it out”**.
   
   Example:
   - Click **Try it out** next to `POST /api/v1/sessions/`
   - Enter some test data:
   ```json
   {
     "name": "First Test Session",
     "rag_document_ids": ["doc-123"]
   }
   ```
   - Click **Execute**
   - You’ll get a live response from your running API (like a `201 Created` with session metadata).
   
   ---
   
   ### Bonus: Swagger + Security
   
   Later, when you add authentication (like API keys or tokens), Swagger UI can also prompt users to enter those. You can protect certain routes and still test them live.
   
   ---
   
   ### Summary
   
   | Feature | What Swagger UI Does |
   |--------|----------------------|
   | Docs | Shows every route, method (GET, POST, etc.), and expected input/output |
   | Interactivity | Lets you test API endpoints right in the browser |
   | Debugging | Helps spot errors in requests or responses |
   | Auto-generated | Powered by FastAPI using your route + Pydantic model definitions |

---

3.  **Define Settings (`core/config.py`):**

    ```python
    # core/config.py
    import os
    from pydantic_settings import BaseSettings
    from pathlib import Path

    # Define the base directory of the project
    # BASE_DIR = Path(__file__).resolve().parent.parent
    # If config.py is in core/, then BASE_DIR needs to go up two levels
    BASE_DIR = Path(__file__).resolve().parent.parent


    class Settings(BaseSettings):
        # --- General Settings ---
        APP_NAME: str = "RAG LLM Server"
        BASE_DIR: Path = BASE_DIR

        # --- Database Settings ---
        # Option 1: Define full path directly
        # SQLITE_DB_FILE: Path = BASE_DIR / "data" / "rag_server.db"
        # CHROMA_DATA_PATH: Path = BASE_DIR / "data" / "chroma_db"

        # Option 2: Define directory and filenames separately
        DATA_DIR: Path = BASE_DIR / "data"
        SQLITE_DB_NAME: str = "rag_server.db"
        CHROMA_DB_DIR_NAME: str = "chroma_db"

        # --- Calculated Paths ---
        @property
        def SQLITE_DATABASE_URL(self) -> str:
            # Ensure the directory exists
            db_dir = self.DATA_DIR
            db_dir.mkdir(parents=True, exist_ok=True)
            # Use absolute path for the database file
            db_path = db_dir / self.SQLITE_DB_NAME
            # Use file:// prefix for absolute paths with aiosqlite/databases
            # Note: For Windows, the format might need adjustment if issues arise,
            # but 'sqlite+aiosqlite:///C:/path/to/db' usually works.
            # For Posix (Linux/macOS): 'sqlite+aiosqlite:////path/to/db'
            # The `databases` library expects the URL format.
            return f"sqlite+aiosqlite:///{db_path.resolve()}"

        @property
        def CHROMA_PERSIST_PATH(self) -> str:
            # Ensure the directory exists
            chroma_path = self.DATA_DIR / self.CHROMA_DB_DIR_NAME
            chroma_path.mkdir(parents=True, exist_ok=True)
            # ChromaDB's client expects a string path to the directory
            return str(chroma_path.resolve())

        # --- LLM Settings (Placeholders) ---
        DEFAULT_MODEL_NAME: str = "placeholder-model" # Example, will be configurable later
        HUGGINGFACE_HUB_CACHE: Path = BASE_DIR / "models_cache" # Example cache path

        # --- Server Settings ---
        SERVER_HOST: str = "0.0.0.0"
        SERVER_PORT: int = 8000

        # --- ChromaDB Settings ---
        # Default collection names, can be overridden
        DOCUMENT_COLLECTION_NAME: str = "rag_documents"
        SESSION_COLLECTION_NAME: str = "chat_sessions"


        # Pydantic settings configuration
        class Config:
            # Load .env file if it exists
            env_file = '.env'
            env_file_encoding = 'utf-8'
            # Allow extra fields (though we try to define all needed)
            extra = 'ignore'

    # Create a single instance of the settings to be imported elsewhere
    settings = Settings()

    # --- Ensure necessary directories exist on import ---
    settings.DATA_DIR.mkdir(parents=True, exist_ok=True)
    settings.HUGGINGFACE_HUB_CACHE.mkdir(parents=True, exist_ok=True)
    # Access properties to trigger directory creation if not already done
    _ = settings.SQLITE_DATABASE_URL
    _ = settings.CHROMA_PERSIST_PATH

    print(f"--- Configuration Loaded ---")
    print(f"SQLite DB URL: {settings.SQLITE_DATABASE_URL}")
    print(f"ChromaDB Path: {settings.CHROMA_PERSIST_PATH}")
    print(f"--------------------------")

    ```
    *   We use `pydantic-settings` `BaseSettings` to define our configuration schema.
    *   Type hints (`str`, `Path`, `int`) provide validation.
    *   We calculate the full SQLite URL and ChromaDB path using properties.
    *   The `Config` inner class tells Pydantic to load variables from a `.env` file.
    *   We create a single `settings` instance that can be imported anywhere.
    *   We explicitly create the necessary data directories when the module is loaded.

4.  **Create `.env` file (Optional, in the project root):**
    This file is for environment-specific overrides or secrets (though we don't have secrets yet).
    ```.env
    # Example .env file
    # You can override settings defined in config.py here
    # SERVER_PORT=8001
    # DEFAULT_MODEL_NAME="another-model"
    ```
    *Remember to add `.env` to your `.gitignore`!*

5.  **Define Database Models/Schema (`db/models.py`):**
    We'll start with the `sessions` table.

    ```python
    # db/models.py
    import sqlalchemy
    import datetime
    from sqlalchemy.sql import func # For default timestamp

    # SQLAlchemy MetaData object serves as a container for Table objects and their associations.
    metadata = sqlalchemy.MetaData()

    # Define the 'sessions' table
    sessions_table = sqlalchemy.Table(
        "sessions",
        metadata,
        sqlalchemy.Column("id", sqlalchemy.String, primary_key=True), # Session ID (UUID or similar string)
        sqlalchemy.Column("name", sqlalchemy.String, nullable=True), # User-friendly name for the session
        sqlalchemy.Column("created_at", sqlalchemy.DateTime(timezone=True), server_default=func.now()),
        sqlalchemy.Column("last_updated_at", sqlalchemy.DateTime(timezone=True), server_default=func.now(), onupdate=func.now()),
        sqlalchemy.Column("llm_model_name", sqlalchemy.String, nullable=True), # Model used for the session
        sqlalchemy.Column("rag_document_ids", sqlalchemy.JSON, nullable=True), # Store list of document IDs used
        sqlalchemy.Column("metadata_tags", sqlalchemy.JSON, nullable=True), # Auto-generated tags for searching sessions
        # We might add more fields later, like a summary or status
    )

    # Define the 'chat_messages' table (related to sessions)
    # We store prompts and responses here linked to a session
    chat_messages_table = sqlalchemy.Table(
        "chat_messages",
        metadata,
        sqlalchemy.Column("id", sqlalchemy.Integer, primary_key=True), # Auto-incrementing ID for each message
        sqlalchemy.Column("session_id", sqlalchemy.String, sqlalchemy.ForeignKey("sessions.id"), nullable=False, index=True),
        sqlalchemy.Column("timestamp", sqlalchemy.DateTime(timezone=True), server_default=func.now()),
        sqlalchemy.Column("role", sqlalchemy.String), # e.g., 'user', 'assistant', 'system'
        sqlalchemy.Column("content", sqlalchemy.Text), # The actual prompt or response text
        sqlalchemy.Column("metadata", sqlalchemy.JSON, nullable=True), # Any extra info (e.g., latency, token count)
    )


    # Define the 'documents' table (for metadata about uploaded docs)
    documents_table = sqlalchemy.Table(
        "documents",
        metadata,
        sqlalchemy.Column("id", sqlalchemy.String, primary_key=True), # Document ID (e.g., UUID)
        sqlalchemy.Column("filename", sqlalchemy.String, nullable=False),
        sqlalchemy.Column("source_path", sqlalchemy.String, nullable=True), # Original path if uploaded from local disk
        sqlalchemy.Column("upload_time", sqlalchemy.DateTime(timezone=True), server_default=func.now()),
        sqlalchemy.Column("processing_status", sqlalchemy.String, default="pending"), # e.g., pending, chunking, embedding, completed, failed
        sqlalchemy.Column("error_message", sqlalchemy.Text, nullable=True),
        sqlalchemy.Column("metadata", sqlalchemy.JSON, nullable=True), # Extracted metadata (author, title, etc.)
        sqlalchemy.Column("chunk_count", sqlalchemy.Integer, nullable=True), # Number of chunks created
        sqlalchemy.Column("vector_count", sqlalchemy.Integer, nullable=True), # Number of vectors embedded
    )

    # Define the 'document_chunks' table (optional, but good for referencing specific chunks)
    # This relates chunks stored in ChromaDB back to the source document and its text
    document_chunks_table = sqlalchemy.Table(
        "document_chunks",
        metadata,
        sqlalchemy.Column("id", sqlalchemy.String, primary_key=True), # Chunk ID (can be the same as ChromaDB ID)
        sqlalchemy.Column("document_id", sqlalchemy.String, sqlalchemy.ForeignKey("documents.id"), nullable=False, index=True),
        sqlalchemy.Column("chunk_index", sqlalchemy.Integer, nullable=False), # Order of the chunk within the document
        sqlalchemy.Column("text_content", sqlalchemy.Text, nullable=False), # The actual text of the chunk
        sqlalchemy.Column("metadata", sqlalchemy.JSON, nullable=True), # Any chunk-specific metadata
        sqlalchemy.Column("vector_id", sqlalchemy.String, nullable=True, index=True), # Reference to the vector in ChromaDB (optional, might just use chunk ID)
    )


    # Add more tables as needed (e.g., for model configurations, users, etc.)

    ```
    *   We use `SQLAlchemy Core` which defines tables using `sqlalchemy.Table`. This gives us fine-grained control without the full ORM overhead initially.
    *   `metadata` object collects all our table definitions.
    *   We define `sessions`, `chat_messages`, `documents`, and `document_chunks` tables with appropriate columns, types, primary/foreign keys, and defaults (like timestamps).
    *   Using `sqlalchemy.JSON` is convenient for flexible fields like document IDs or tags.

6.  **Set Up Database Connection Logic (`db/database.py`):**

    ```python
    # db/database.py
    import sqlalchemy
    import databases
    import chromadb

    from core.config import settings # Import our settings instance
    from .models import metadata # Import the MetaData object containing our table definitions

    # --- SQLite Setup (using 'databases' library for async) ---
    # Use the URL from our settings
    DATABASE_URL = settings.SQLITE_DATABASE_URL
    # Create the 'databases' instance
    database = databases.Database(DATABASE_URL)

    # --- SQLAlchemy Core Engine Setup (needed for metadata creation) ---
    # We need a synchronous engine just to create tables from metadata
    # The actual queries will use the async 'database' object
    # Remove the '+aiosqlite' part for the synchronous engine URL
    sync_engine_url = DATABASE_URL.replace("+aiosqlite", "")
    sync_engine = sqlalchemy.create_engine(sync_engine_url)

    def create_metadata_tables():
        """Creates all tables defined in the metadata."""
        print(f"Attempting to create tables in database: {settings.SQLITE_DB_NAME}")
        try:
            # Use the synchronous engine to create tables
            metadata.create_all(bind=sync_engine)
            print("Tables created successfully (if they didn't exist).")
        except Exception as e:
            print(f"Error creating tables: {e}")
            raise # Re-raise the exception to signal failure

    # --- ChromaDB Setup ---
    # Configure and initialize ChromaDB persistent client
    try:
        # Create a Settings object specifically for ChromaDB initialization if needed,
        # often path is sufficient for PersistentClient.
        chroma_settings = chromadb.Settings(
            anonymized_telemetry=False,
            is_persistent=True,
             persist_directory=settings.CHROMA_PERSIST_PATH # Optional here if passed to client
            # Note: chroma_db_impl is often inferred based on PersistentClient
        )
    
        chroma_client = chromadb.PersistentClient(
            path=settings.CHROMA_PERSIST_PATH,
            settings=chroma_settings
        )
        print(f"ChromaDB persistent client initialized. Path: {settings.CHROMA_PERSIST_PATH}")
    
    except Exception as e:
        print(f"FATAL: Error initializing ChromaDB client: {e}")
        # Depending on the app's needs, you might want to exit or handle this gracefully
        raise # Re-raise to stop the server startup if ChromaDB is essential

    # --- Database Connection Management for FastAPI ---
    async def connect_db():
        """Connects to the SQLite database."""
        try:
            await database.connect()
            print("Database connection established.")
        except Exception as e:
            print(f"Error connecting to database: {e}")
            # Handle connection error appropriately (e.g., exit, retry)
            raise

    async def disconnect_db():
        """Disconnects from the SQLite database."""
        try:
            await database.disconnect()
            print("Database connection closed.")
        except Exception as e:
            print(f"Error disconnecting from database: {e}")
            # Log error, but don't prevent shutdown

    # --- ChromaDB Collection Management ---
    def get_or_create_collections():
        """Ensures the necessary ChromaDB collections exist."""
        try:
            # Document collection
            doc_collection_name = settings.DOCUMENT_COLLECTION_NAME
            chroma_client.get_or_create_collection(
                name=doc_collection_name,
                # Optionally add metadata like embedding function here later
                # metadata={"hnsw:space": "cosine"} # Example: set distance function
            )
            print(f"ChromaDB document collection '{doc_collection_name}' ensured.")

            # Session collection (for chat history embedding)
            session_collection_name = settings.SESSION_COLLECTION_NAME
            chroma_client.get_or_create_collection(
                name=session_collection_name,
                # metadata={"hnsw:space": "cosine"}
            )
            print(f"ChromaDB session collection '{session_collection_name}' ensured.")

        except Exception as e:
            print(f"Error setting up ChromaDB collections: {e}")
            raise

    # --- Helper to get ChromaDB client (useful for dependency injection later) ---
    def get_chroma_client() -> chromadb.Client:
        return chroma_client

    # --- Helper to get SQLite database connection (useful for dependency injection later) ---
    def get_sqlite_db() -> databases.Database:
        return database

    ```
    *   We import our `settings` and `metadata`.
    *   We create the `databases.Database` instance for async SQLite operations.
    *   We create a *synchronous* SQLAlchemy engine (`create_engine`) specifically for the `metadata.create_all()` call, as `databases` doesn't handle table creation directly.
    *   `create_metadata_tables()` function encapsulates table creation.
    *   We initialize the `chromadb.Client` using the persistence path from settings and disable telemetry.
    *   `connect_db` and `disconnect_db` are async functions to manage the SQLite connection lifecycle.
    *   `get_or_create_collections` ensures our core Chroma collections exist on startup.
    *   `get_chroma_client` and `get_sqlite_db` are simple functions that return the initialized clients/databases. These will be useful for FastAPI's dependency injection system later.

7.  **Update Main Application (`app/main.py`):**
    Modify the main application file to use the new structure, settings, and database setup.

    ```python
    # app/main.py
    import uvicorn
    from fastapi import FastAPI
    from fastapi.middleware.cors import CORSMiddleware
    import datetime
    import sys
    import os
    
    # --- FastAPI Application Setup ---
    app = FastAPI(
        title=settings.APP_NAME,
        description="API server for managing RAG documents, interacting with LLMs, and managing chat sessions.",
        version="0.1.0",
    )

    origins = [
        "http://localhost",
        "http://localhost:5173",
        "http://127.0.0.1:5173",
    ]
    
    app.add_middleware(
        CORSMiddleware,
        allow_origins=origins,
        allow_credentials=True,
        # --- Specify Methods and Headers ---
        allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"], # Explicitly allow POST and OPTIONS
        allow_headers=["Content-Type", "Authorization", "Accept", "Origin"], # Explicitly allow Content-Type and others
        # --- End Specification ---
    )
    print(f"CORS Middleware enabled for origins: {origins}")

    # Add project root to the Python path (adjust based on your run environment)
    # This allows imports like `from core.config import settings`
    project_root = os.path.abspath(os.path.join(os.path.dirname(__file__), '..'))
    if project_root not in sys.path:
        sys.path.insert(0, project_root)

    # Now we can import from our modules
    from core.config import settings
    from db.database import (
        connect_db,
        disconnect_db,
        create_metadata_tables,
        get_or_create_collections
    )
    # Import routers later when we create them
    # from app.api.endpoints import documents, sessions, models, system

    # --- Create database tables and Chroma collections ---
    # This runs when the module is loaded (i.e., on server start)
    # It's okay here for simplicity, but for complex apps, consider Alembic/migrations
    try:
        print("Initializing database schema...")
        create_metadata_tables()
        print("Initializing ChromaDB collections...")
        get_or_create_collections()
        print("Initialization complete.")
    except Exception as e:
        print(f"FATAL: Failed to initialize database or ChromaDB: {e}")
        # Exit if critical initialization fails
        sys.exit(1)

    # --- Event Handlers for Database Connections ---
    @app.on_event("startup")
    async def startup_event():
        """Connect to database on server startup."""
        print("Server starting up...")
        await connect_db()
        # Perform other startup tasks if needed
        print("Server startup complete.")


    @app.on_event("shutdown")
    async def shutdown_event():
        """Disconnect from database on server shutdown."""
        print("Server shutting down...")
        await disconnect_db()
        # Perform other shutdown tasks if needed
        print("Server shutdown complete.")


    # --- Basic Root Endpoint ---
    @app.get("/")
    async def read_root():
        """
        Root endpoint providing basic server information.
        """
        return {
            "message": f"Welcome to the {settings.APP_NAME}!",
            "status": "running",
            "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat()
        }

    # --- Placeholder for Future Routers ---
    # Example: app.include_router(documents.router, prefix="/api/v1/documents", tags=["Documents"])
    # Example: app.include_router(sessions.router, prefix="/api/v1/sessions", tags=["Sessions"])
    # Example: app.include_router(models.router, prefix="/api/v1/models", tags=["Models"])
    # Example: app.include_router(system.router, prefix="/api/v1/system", tags=["System Status"])


    # --- Running the Server (for direct execution) ---
    if __name__ == "__main__":
        # Correct path for uvicorn when running main.py directly
        # Need to tell uvicorn where the app object is relative to the CWD
        # If running `python app/main.py` from the project root:
        # uvicorn.run("app.main:app", ...)
        # If running from inside `app` directory (less common):
        # uvicorn.run("main:app", ...)

        # Assume running from project root directory `python -m app.main` or similar
        print(f"Starting server on http://{settings.SERVER_HOST}:{settings.SERVER_PORT}")
        uvicorn.run(
            "app.main:app", # Point to the app instance within the 'app' package
            host=settings.SERVER_HOST,
            port=settings.SERVER_PORT,
            reload=True,
             # Watch specific directories for reload
            reload_dirs=[str(settings.BASE_DIR / 'app'), str(settings.BASE_DIR / 'core'), str(settings.BASE_DIR / 'db')]
        )

    ```
    *   We add code to adjust `sys.path` to ensure modules in `core` and `db` can be imported correctly. This can sometimes be handled by how you run the application (e.g., `python -m app.main` from the root), but explicit path modification is safer in some setups.
    *   We import our settings and database functions.
    *   **Crucially**: We call `create_metadata_tables()` and `get_or_create_collections()` *before* defining the FastAPI app. This ensures the necessary structures exist right at the start. For production systems, database migrations (e.g., using Alembic) are preferred over `create_all`.
    *   We use FastAPI's `startup` and `shutdown` events to manage the async database connection pool (`connect_db`, `disconnect_db`).
    *   The root endpoint now uses the app name from settings.
    *   The `if __name__ == "__main__":` block is updated to reflect the new file location (`app.main:app`) and uses settings for host/port. We also add `reload_dirs` to make `--reload` more effective with our subdirectory structure.

8.  **Update `.gitignore`:**
    Ensure the data directory is ignored.
    ```gitignore
    # Add/update this line:
    data/
    ```

9.  **Run the Updated Server:**
    Navigate back to the project root directory (`rag_llm_server/`) in your terminal.
    ```bash
    # Ensure your virtual environment is active
    # source venv/bin/activate or venv\Scripts\activate

    # Run using uvicorn directly (recommended)
    uvicorn app.main:app --reload --reload-dirs app --reload-dirs core --reload-dirs db

    # Or run the main script (if __name__ == "__main__" is set up correctly)
    # python app/main.py
    ```
    You should see output indicating:
    *   Configuration loaded (SQLite URL, Chroma Path).
    *   Attempting to create tables.
    *   Tables created successfully.
    *   ChromaDB collections ensured.
    *   Server starting up...
    *   Database connection established.
    *   Server startup complete.
    *   Uvicorn running on `http://0.0.0.0:8000`.

    Check the `rag_llm_server/data/` directory. You should see `rag_server.db` (the SQLite file) and a `chroma_db/` subdirectory containing ChromaDB's persistence files (like `chroma.sqlite3` and potentially parquet files later). Accessing `http://localhost:8000` and `http://localhost:8000/docs` should still work.

**Summary of Changes:**

*   Introduced structured configuration using `pydantic-settings`.
*   Defined SQLite table schemas using SQLAlchemy Core (`db/models.py`).
*   Set up async SQLite connection management (`databases`) and ChromaDB client initialization (`db/database.py`).
*   Used FastAPI startup/shutdown events for database connection handling.
*   Organized the code into `app`, `core`, and `db` directories.
*   Ensured necessary data directories and database structures are created on startup.



---

## Flow of a User Request (Upload Document & Create Chat Session)  
  
Here's a high-level walkthrough of a user uploading a document and starting a chat session in your RAG LLM server setup, step by step:  

![User Process Flow for Document Uploading and CHat](process_flow.png)
  
### **1. User sends a request**
- **POST /api/v1/documents** with a document (PDF, TXT, etc.).
- Or **POST /api/v1/sessions** to start a new chat session.

---

### **2. FastAPI App Receives Request**
- The `app/main.py` FastAPI instance receives the request.
- It routes to the appropriate handler (you'll define in `app/api/endpoints/` later).

---

### **3. Settings Loaded**
- `from core.config import settings`
- Ensures all directories exist.
- Provides paths and names for DB, ChromaDB, model config.

---

### **4. Document Endpoint Logic**
Example steps in document upload:
```python
# Pseudocode for upload handler
document_id = generate_uuid()
save_file_to_disk(file, settings.DATA_DIR / document_id)
```

---

### **5. Save Document Metadata to SQLite**
- Insert a row into `documents` table in `rag_server.db`:
```sql
INSERT INTO documents (id, filename, upload_time, ...) VALUES (...);
```

---

### **6. Extract & Chunk Text**
- The file is read, split into chunks (e.g., 500 tokens each).

---

### **7. Embed Chunks into Vectors**
- Each chunk is passed to a model (e.g., Hugging Face transformer).
- Vector embeddings are generated.

---

### **8. Save Chunks to ChromaDB**
- Chunks with their metadata (chunk index, source doc ID) are saved using:
```python
chroma_client.get_or_create_collection("rag_documents")
collection.add(documents=[...], embeddings=[...])
```

---

### **9. Create Chat Session (optional but common)**
- A `POST /api/v1/sessions` call inserts a new row in `sessions`.
- Stores model name, timestamp, related document IDs.

---

### **10. Server Responds**
- FastAPI sends a JSON response:
```json
{
  "status": "success",
  "session_id": "abc123",
  "document_id": "xyz789"
}
```

---

### **11. Next User Prompt Flow**
When user sends a prompt:
- Find matching documents by querying ChromaDB for similar embeddings.
- Use them to generate a RAG-style response from your LLM.
- Save chat exchange in `chat_messages`.

---

# High-Level Design Philosophy  

The architecture you’re building is based on several **best practices and architectural theories** that aim to create **scalable, maintainable, modular, and async-friendly** systems. Let's break this down into the core theoretical principles and design motivations behind each major decision:

---

## 1. **Separation of Concerns (SoC)**

> **Theory:** Systems should separate logic into distinct sections with minimal overlap in responsibilities.

### How it's applied:
- `core/config.py` handles **only settings**.
- `db/database.py` deals with **connections and setup**.
- `db/models.py` defines **only the schema**.
- `app/main.py` runs the **application** logic.
- `api/endpoints/` (later) handles **routes and user interactions**.

> **Why:** Makes your codebase easier to understand, test, and maintain. If one part breaks (like the database), it doesn’t break your whole app logic.

---

## 2. **Asynchronous I/O for Scalability**

> **Theory:** Use non-blocking I/O to handle many concurrent connections without locking the entire process.

### How it's applied:
- Using `databases` and `aiosqlite` lets you handle requests without waiting for the DB to finish before serving another.

> **Why:** Python's `async` is perfect for web APIs where you often wait on network, file, or DB operations. This improves **performance** and **user experience**, especially when many users are interacting at once.

---

## 3. **Modularity + Reusability**

> **Theory:** Design reusable, single-purpose components (i.e., loosely coupled modules).

### How it's applied:
- `settings` can be reused across any part of your app (`settings.SERVER_PORT`, etc.).
- `get_chroma_client()` and `get_sqlite_db()` act as clean injection points.

> **Why:** Makes it easy to swap out components (e.g., switch to Postgres or Redis later) with minimal refactoring.

---

## 4. **Declarative Configuration with Pydantic**

> **Theory:** Use declarative (not imperative) code for configs to reduce complexity and centralize logic.

### How it's applied:
- All settings are typed, validated, and overridable via `.env`.

> **Why:** Prevents bugs due to misconfiguration. Easy to extend and introspect (e.g., environment overrides for staging vs production).

---

## 5. **Persistence and Fault Tolerance**

> **Theory:** Systems must persist important state and recover gracefully on restart.

### How it's applied:
- ChromaDB and SQLite persist to disk (`data/` folder).
- You ensure tables and folders exist on startup.

> **Why:** This design makes your server **stateful and restart-safe**, which is key for handling long-lived sessions or storing uploaded documents.

---

## 6. **Retrieval-Augmented Generation (RAG) Theory**

> **Theory (RAG):** Combine the power of retrieval (from documents) with generation (from LLMs) for grounded, dynamic responses.

### How it's applied:
- Upload → chunk → embed → store in ChromaDB.
- On prompt: query vector DB → inject retrieved docs into LLM prompt.

> **Why:** This allows your LLM to answer based on **custom, dynamic knowledge**, without fine-tuning the model.

> Source: [Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks” (2020)](https://arxiv.org/abs/2005.11401)

---

## 7. **Initialization on Import**

> **Theory:** Some systems should be initialized when the module loads (like config or ensuring folders exist).

### How it's applied:
- On import, `config.py` and `database.py` initialize necessary directories and ChromaDB collections.

> **Why:** Prevents runtime errors due to missing folders or tables. Think of it like setting up your tools before using them.

---

## 8. **Minimal MVP, Maximum Scalability**

> **Theory:** Start small and clean, but plan for scale.

### How it's applied:
- SQLite + ChromaDB are lightweight, but you’ve modularized everything (you can swap for Postgres + Milvus later).

> **Why:** It’s easier to scale when your foundation is well-structured.

---


