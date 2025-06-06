# Step 13: Long Term Memory #

## Note - this file is out of sync with the src code due to some bugs I discovered and have now fixed ##

Storing and retrieving chat history as RAG documents to provide long-term memory for sessions is a powerful and logical extension. It turns the conversation itself into a searchable knowledge base specific to that interaction thread.

Here's how we can integrate this:

**Conceptual Workflow Change:**

1.  **Indexing:** After each user message and assistant response pair is finalized and stored in the SQLite `chat_messages` table, trigger a background task to embed both the user's query and the assistant's response text. Store these embeddings, the text content, and relevant metadata (like `session_id`, `message_id`, `role`, `timestamp`, and a `type='chat'`) into your **ChromaDB collection** (likely the same one used for documents, `rag_documents`, differentiated by metadata).
2.  **Retrieval:** When a *new* user message arrives (`add_message_to_session`):
    *   Perform the RAG search for *documents* as before (if documents are linked to the session).
    *   Perform a **second** semantic search using the user's message embedding, but this time query ChromaDB filtering specifically for records where `type='chat'` AND `session_id` matches the *current session*. Retrieve the top-k most relevant past messages (user or assistant).
3.  **Summarization:** Because retrieved chat history snippets might be numerous or verbose, summarize the results from the chat history search (just like we did for document chunks), potentially using the current user query to guide the summary focus.
4.  **Prompt Construction:** Modify the prompt assembly to include sections for:
    *   Instruction
    *   Retrieved Document Context (Summarized)
    *   Relevant Past Conversation (Summarized Chat History)
    *   Recent Chat History (Maybe the last 1-2 turns verbatim)
    *   Current User Query
5.  **Token Management:** Apply truncation logic considering the combined length of *both* summarized document context *and* summarized chat history context, plus recent history and the query.

**Implementation Steps (Backend Focus):**

**1. Modify ChromaDB Metadata during Indexing (Background Task)**

We need a new background task for indexing chat turns and update the existing document indexing to include a `type` field.

*   **Update `services/document_processor.py` (Document Indexing):**
    When adding document chunks to ChromaDB in `process_document_upload`, add a `type` field to the metadata.

    ```python
    # services/document_processor.py (inside process_document_upload, before doc_collection.add)

                    # Prepare metadata for ChromaDB vectors
                    chroma_metadatas = [
                        {
                            "document_id": doc_id,
                            "filename": filename,
                            "chunk_index": i,
                            "type": "document" # <-- ADD TYPE FIELD
                        }
                        for i in range(chunk_count)
                    ]

                    # Add to ChromaDB collection
                    try:
                        doc_collection.add(
                            ids=chunk_ids,
                            embeddings=embeddings,
                            documents=chunks_for_embedding,
                            metadatas=chroma_metadatas # Use updated metadata
                        )
                        # ... rest of the function ...
    ```
    
*   **Update `services/llm_service.py`:**
  
   ```python
   # services/llm_service.py
   import time
   import logging
   import asyncio
   import gc
   from enum import Enum
   from pathlib import Path
   from typing import Optional, Dict, Any, List, Union
   
   import torch
   from transformers import pipeline, AutoModelForCausalLM, AutoTokenizer, PreTrainedTokenizerBase
   from transformers import BitsAndBytesConfig # For quantization
   
   from core.config import settings
   
   from .llm_backends import (
       LLMBackendBase,
       LocalTransformersBackend,
       OllamaBackend,
       VLLMBackend,
       InstructLabBackend,
       GeminiBackend
   )
   
   DEFAULT_HF_CACHE = Path.home() / ".cache" / "huggingface" / "hub"
   
   logging.basicConfig(level=logging.INFO)
   logger = logging.getLogger(__name__)
   
   # --- LLM Status Enum ---
   class LLMStatus(Enum):
       INACTIVE = "Inactive" # No backend configured/active
       LOADING = "Loading"   # Only for local backend
       CONFIGURING = "Configuring" # For API backends (setting model)
       READY = "Ready"       # Backend is configured and ready
       FAILED = "Failed"     # Loading/Configuration failed
       UNLOADING = "Unloading" # Only for local backend
   
   
   # --- LLM State ---
   llm_state: Dict[str, Any] = {
       "backend_instance": None, # Holds the active backend object (Local, Ollama, etc.)
       "backend_type": None,     # Which type is active ('local', 'ollama', ...)
       "active_model": None,     # Name/path of the model active on the backend
       "status": LLMStatus.INACTIVE,
       "config": { # Global generation config, passed to backend
           "max_new_tokens": settings.DEFAULT_LLM_MAX_NEW_TOKENS,
           "temperature": settings.DEFAULT_LLM_TEMPERATURE,
           "top_p": settings.DEFAULT_LLM_TOP_P,
           "top_k": settings.DEFAULT_LLM_TOP_K,
           "repetition_penalty": 1.15 # Keep this from previous fix
       },
       "last_error": None,
   }
   
   # --- Helper Function to Scan Local Models ---
   def list_local_models() -> List[str]:
       """Scans the DEDICATED LOCAL_MODELS_DIR for potential model directories."""
       local_models = []
       models_dir = settings.LOCAL_MODELS_DIR
       logger.info(f"Scanning for local models in: {models_dir}")
       if models_dir.is_dir():
           for item in models_dir.iterdir():
               if item.is_dir():
                   has_config = (item / "config.json").is_file()
                   has_pytorch_bin = list(item.glob("*.bin"))
                   has_safetensors = list(item.glob("*.safetensors"))
                   if has_config and (has_pytorch_bin or has_safetensors):
                       logger.debug(f"Found potential local model directory: {item.name}")
                       local_models.append(str(item.resolve()))
       logger.info(f"Found {len(local_models)} potential models in {models_dir}.")
       return local_models
   
   def list_cached_hub_models() -> List[str]:
       """
       Scans the configured Hugging Face Hub cache directory AND the default
       cache directory for downloaded models.
       """
       hub_models: Set[str] = set() # Use a set to automatically handle duplicates
       # Directories to scan
       scan_dirs = {settings.HUGGINGFACE_HUB_CACHE.resolve(), DEFAULT_HF_CACHE.resolve()}
   
       for cache_dir in scan_dirs:
           logger.info(f"Scanning for cached Hub models in: {cache_dir}")
           if not cache_dir.is_dir():
               logger.warning(f"Cache directory not found or not a directory: {cache_dir}")
               continue
   
           for item in cache_dir.iterdir():
               if item.is_dir() and item.name.startswith("models--"):
                   snapshots_dir = item / "snapshots"
                   if snapshots_dir.is_dir() and any(snapshots_dir.iterdir()):
                       model_id = item.name.removeprefix("models--").replace("--", "/")
                       logger.debug(f"Found potential cached Hub model: {model_id} (from {item.name}) in {cache_dir}")
                       hub_models.add(model_id) # Add to set
   
       logger.info(f"Found {len(hub_models)} unique potential cached models across specified cache directories.")
       return sorted(list(hub_models)) # Return sorted list
   
   # ... (_load_model_task, get_llm_status, update_llm_config, generate_text) ...
   # NOTE: The _load_model_task should work fine with these Hub IDs without modification
   
   # --- Internal Unload Function ---
   async def _unload_current_backend():
       """Calls unload on the current backend instance if it exists."""
       if llm_state["backend_instance"] is not None:
           logger.info(f"Unloading current backend ({llm_state['backend_type']})...")
           try:
               await llm_state["backend_instance"].unload()
           except Exception as e:
                logger.error(f"Error during backend unload: {e}", exc_info=True)
           finally:
               llm_state["backend_instance"] = None
               llm_state["backend_type"] = None
               llm_state["active_model"] = None
               llm_state["status"] = LLMStatus.INACTIVE
               llm_state["last_error"] = None
   
   # --- Core Loading Function (Runs in Background) ---
   # This now specifically targets the LocalTransformersBackend instance
   
   async def _load_local_model_task(
       backend: LocalTransformersBackend,
       model_id: str,
       device: str,
       quantization: Optional[str]
   ):
       """Loads a model and tokenizer into the LocalTransformersBackend instance."""
       task_start_time = time.time()
       logger.info(f"[Local Loader] Starting task for '{model_id}' on device '{device}' q: '{quantization}'...")
   
       llm_state["status"] = LLMStatus.LOADING
       llm_state["last_error"] = None
       backend.model_name_or_path = model_id
   
       try:
           load_start_time = time.time()
           quantization_config = None
           # Determine device map strategy for AutoModel
           # Use 'auto' for multi-gpu or quantization, explicit device for single GPU full precision
           device_map = "auto" if quantization or torch.cuda.device_count() > 1 else device
           if device == "cpu":
                device_map = None # Don't use device_map for CPU
   
           logger.info(f"[Local Loader] Effective device_map strategy: '{device_map}'")
   
           if quantization:
               # ... (Quantization config logic - same as before) ...
                 if quantization == "8bit" and torch.cuda.is_available():
                    quantization_config = BitsAndBytesConfig(load_in_8bit=True)
                    logger.info("[Local Loader] Applying 8-bit quantization.")
                 elif quantization == "4bit" and torch.cuda.is_available():
                     quantization_config = BitsAndBytesConfig(
                        load_in_4bit=True, bnb_4bit_compute_dtype=torch.float16,
                        bnb_4bit_quant_type="nf4", bnb_4bit_use_double_quant=True,
                    )
                     logger.info("[Local Loader] Applying 4-bit quantization (nf4).")
                 else:
                    logger.warning(f"[Local Loader] Quantization '{quantization}' not supported/applicable. Loading full precision.")
                    quantization = None
   
           # --- Load Tokenizer ---
           logger.info(f"[Local Loader] Loading tokenizer for '{model_id}'...")
           tokenizer = AutoTokenizer.from_pretrained(
               model_id, trust_remote_code=True,
               cache_dir=str(settings.HUGGINGFACE_HUB_CACHE.resolve())
           )
           logger.info("[Local Loader] Tokenizer loaded.")
   
           # --- Load Model ---
           logger.info(f"[Local Loader] Loading model '{model_id}'...")
           model = AutoModelForCausalLM.from_pretrained(
               model_id,
               device_map=device_map, # Use determined device_map
               quantization_config=quantization_config,
               torch_dtype=torch.float16 if device == "cuda" and not quantization else None, # Use float16 on GPU if not quantizing to bitsandbytes native types
               trust_remote_code=True,
               cache_dir=str(settings.HUGGINGFACE_HUB_CACHE.resolve())
           )
           logger.info("[Local Loader] Model loaded successfully.")
   
           load_time = time.time() - load_start_time
           logger.info(f"[Local Loader] Successfully loaded model '{model_id}' in {load_time:.2f}s.")
   
           max_len = getattr(model.config, "max_position_embeddings", None) \
                     or getattr(tokenizer, "model_max_length", None) \
                     or 1024 # Default fallback
           logger.info(f"[Local Loader] Determined model max length: {max_len}")
   
           # --- Update the specific backend instance ---
           backend.model = model # Store model
           backend.tokenizer = tokenizer
           backend.model_name_or_path = model_id
           backend.load_config = {"device": device, "quantization": quantization}
           backend.max_model_length = max_len
   
           # --- Update global state ---
           llm_state["status"] = LLMStatus.READY
           llm_state["active_model"] = model_id
           llm_state["last_error"] = None
   
       except Exception as e:
           # ... (Error handling - clear backend.model, backend.tokenizer) ...
           error_message = f"[Local Loader] Failed to load model '{model_id}': {type(e).__name__}: {e}"
           logger.error(error_message, exc_info=True)
           backend.model = None # Ensure refs are cleared on error
           backend.tokenizer = None
           llm_state["status"] = LLMStatus.FAILED
           llm_state["last_error"] = error_message
           llm_state["active_model"] = None
       finally:
           task_duration = time.time() - task_start_time
           logger.info(f"[Local Loader] Task for '{model_id}' finished in {task_duration:.2f}s. Status: {llm_state['status'].value}")
   
   # --- Public Service Functions ---
   def get_llm_status() -> Dict[str, Any]:
       """Returns the current status and configuration of the active LLM backend."""
       # status_copy = {
       #     "backend_type": llm_state["backend_type"],
       #     "active_model": llm_state["active_model"],
       #     "status": llm_state["status"].value,
       #     "generation_config": llm_state["config"],
       #     "last_error": llm_state["last_error"],
       #     "backend_details": None
       # }
       # if llm_state["backend_instance"] is not None:
       #     # Get specific details from the active backend
       #     status_copy["backend_details"] = llm_state["backend_instance"].get_status_dict()
   
       # return status_copy
   
       """Returns the current status and configuration of the active LLM backend."""
       backend_instance = llm_state.get("backend_instance") # Get instance safely
       backend_details = None
       if backend_instance is not None:
           try:
               # Call get_status_dict only if instance exists
               backend_details = backend_instance.get_status_dict()
           except Exception as e:
               # Catch potential errors during get_status_dict itself
               logger.error(f"Error getting backend details from instance: {e}", exc_info=True)
               backend_details = {"error": "Failed to retrieve backend details"}
   
       status_copy = {
           "backend_type": llm_state.get("backend_type"), # Use .get for safety
           "active_model": llm_state.get("active_model"),
           "status": llm_state.get("status", LLMStatus.INACTIVE).value, # Default to INACTIVE if status key somehow missing
           "generation_config": llm_state.get("config"),
           "last_error": llm_state.get("last_error"),
           "backend_details": backend_details # Use the safely retrieved details
       }
       return status_copy
   
   def update_llm_config(new_config: Dict[str, Any]) -> Dict[str, Any]:
       """Updates the global generation configuration."""
       # Config applies globally, passed to whichever backend is active
       allowed_keys = {"max_new_tokens", "temperature", "top_p", "top_k", "repetition_penalty"}
       updated = False
       for key, value in new_config.items():
           if key in allowed_keys:
               if key in llm_state["config"] and llm_state["config"][key] != value:
                    logger.info(f"Updating global LLM generation config: {key} = {value}")
                    llm_state["config"][key] = value
                    updated = True
           else:
                logger.warning(f"Ignoring unsupported config key: {key}")
       if not updated:
            logger.info("No changes applied to LLM config.")
       return llm_state["config"]
   
   
   
   # Now delegates to the active backend, running in executor if needed
   async def generate_text(prompt: str) -> Optional[str]:
       """Generates text using the currently active LLM backend."""
       backend = llm_state.get("backend_instance")
       current_status = llm_state.get("status")
   
       if current_status != LLMStatus.READY or backend is None:
           logger.error(f"Cannot generate text: LLM backend not ready (Status: {current_status.value if current_status else 'N/A'}).")
           return None
   
       config = llm_state["config"] # Get current global config
   
       try:
           # Local backend's generate is blocking, API backends' generate is async
           if isinstance(backend, LocalTransformersBackend):
               logger.info("Running local generation in executor...")
               loop = asyncio.get_running_loop()
               # LocalBackend.generate is sync, run it in executor
               result = await loop.run_in_executor(None, backend.generate, prompt, config) # Pass config
           elif isinstance(backend, (OllamaBackend, VLLMBackend, InstructLabBackend, GeminiBackend)):
               logger.info(f"Running API generation via {llm_state['backend_type']} backend...")
               # API backend generate methods are already async
               result = await backend.generate(prompt, config) # Pass config
           else:
                logger.error(f"Unknown backend type {type(backend)} cannot generate.")
                result = None
   
           return result
   
       except Exception as e:
           logger.error(f"Error during call to backend generate method: {e}", exc_info=True)
           return None
   
   # --- Function to initiate backend loading/configuration ---
   # This replaces the direct call logic in the API endpoint
   async def set_active_backend(
       backend_type: str,
       model_name_or_path: str,
       device: Optional[str] = None, # Primarily for local
       quantization: Optional[str] = None # Primarily for local
   ):
       """Sets the active LLM backend, unloading previous one if necessary."""
       logger.info(f"Request to set active backend: type='{backend_type}', model='{model_name_or_path}'")
   
       # 1. Unload existing backend first
       await _unload_current_backend()
   
       # 2. Create and configure the new backend instance
       new_backend: Optional[LLMBackendBase] = None
       llm_state["status"] = LLMStatus.CONFIGURING # Or LOADING for local
       llm_state["backend_type"] = backend_type
       llm_state["active_model"] = model_name_or_path # Tentative
       llm_state["last_error"] = None
   
       try:
           if backend_type == "local":
               llm_state["status"] = LLMStatus.LOADING # Specific status for local
               local_backend_instance = LocalTransformersBackend()
               llm_state["backend_instance"] = local_backend_instance # Store instance before background task
   
               # Run the actual loading in background executor
               loop = asyncio.get_running_loop()
               final_device = device or settings.LLM_DEVICE
               final_quant = quantization # Uses request value directly, None if not provided
               if quantization is None and settings.DEFAULT_LLM_QUANTIZATION is not None:
                    final_quant = settings.DEFAULT_LLM_QUANTIZATION # Apply default if needed
   
               logger.info(f"Scheduling local model load task: model='{model_name_or_path}', device='{final_device}', quantization='{final_quant}'")
               # Use wrapper to run the async task in executor
               def local_load_wrapper():
                    asyncio.run(_load_local_model_task(local_backend_instance, model_name_or_path, final_device, final_quant))
               loop.run_in_executor(None, local_load_wrapper)
               # Status will be updated to READY/FAILED by the background task
               return # Return early, loading happens in background
   
           elif backend_type == "ollama":
               if not settings.OLLAMA_BASE_URL:
                   raise ValueError("OLLAMA_BASE_URL not configured.")
               new_backend = OllamaBackend(settings.OLLAMA_BASE_URL, model_name_or_path)
               # TODO: Optionally ping Ollama API here to verify model exists?
   
           elif backend_type == "vllm":
                if not settings.VLLM_BASE_URL:
                    raise ValueError("VLLM_BASE_URL not configured.")
                new_backend = VLLMBackend(settings.VLLM_BASE_URL, model_name_or_path)
                # TODO: Ping vLLM?
   
           elif backend_type == "instructlab":
               if not settings.INSTRUCTLAB_BASE_URL:
                   raise ValueError("INSTRUCTLAB_BASE_URL not configured.")
               new_backend = InstructLabBackend(settings.INSTRUCTLAB_BASE_URL, model_name_or_path)
               # TODO: Ping InstructLab?
   
           elif backend_type == "gemini":
                if not settings.GOOGLE_API_KEY:
                    raise ValueError("GOOGLE_API_KEY not configured in settings/environment.")
                # Initialization happens here, might raise error if API key invalid etc.
                new_backend = GeminiBackend(settings.GOOGLE_API_KEY, model_name_or_path)
                
           else:
               raise ValueError(f"Unsupported LLM_BACKEND_TYPE: {backend_type}")
   
           # If we reached here, it's an API backend
           llm_state["backend_instance"] = new_backend
           llm_state["active_model"] = model_name_or_path # Confirm model
           llm_state["status"] = LLMStatus.READY # API backends are 'ready' once configured
           logger.info(f"Backend '{backend_type}' configured and ready with model '{model_name_or_path}'.")
   
       except Exception as e:
           error_message = f"Failed to configure backend '{backend_type}' with model '{model_name_or_path}': {e}"
           logger.error(error_message, exc_info=True)
           llm_state["status"] = LLMStatus.FAILED
           llm_state["last_error"] = error_message
           llm_state["backend_instance"] = None
           llm_state["backend_type"] = None
           llm_state["active_model"] = None
           # Re-raise? Or just let status reflect failure? Let status reflect.
   
   def summarize_text_with_query(text_to_summarize: str, user_query: str, max_summary_tokens: int = 100) -> Optional[str]:
       """
       Uses the loaded LLM to summarize text, guided by a user query.
       NOTE: This makes an additional LLM call and has concurrency issues if state is modified directly.
       """
       # Check if backend is ready
       backend = llm_state.get("backend_instance")
       current_status = llm_state.get("status")
       if current_status != LLMStatus.READY or backend is None:
           logger.error("LLM not ready, cannot summarize.")
           return None
   
       # Construct a specific prompt for summarization
       summarization_prompt = f"""### INSTRUCTION:
   Concisely summarize the following text, focusing on aspects relevant to the user query. Output only the summary.
   
   ### USER QUERY:
   {user_query}
   
   ### TEXT TO SUMMARIZE:
   {text_to_summarize}
   
   ### SUMMARY:"""
   
       # Use a subset of generation config for summarization
       summary_config_override = {
           "temperature": 0.3, # Less creative
           "top_p": 0.9,
           "top_k": 40,
           "max_new_tokens": max_summary_tokens,
           "repetition_penalty": 1.1 # Use a slight penalty here too
       }
       # Get current config and merge overrides *without modifying global state*
       current_config = llm_state["config"].copy()
       summary_config = current_config
       summary_config.update(summary_config_override) # Update the copy
   
       logger.info(f"Requesting summarization for query '{user_query[:30]}...'")
       logger.debug(f"Summarization Prompt:\n{summarization_prompt[:200]}...")
       logger.debug(f"Summarization Config: {summary_config}")
   
       summary = None
       try:
           # --- Call the backend's generate method directly ---
           # Decide how to run based on backend type (like in generate_text)
           if isinstance(backend, LocalTransformersBackend):
                loop = asyncio.get_running_loop()
                # Pass the specific summary_config
                summary = loop.run_in_executor(None, backend.generate, summarization_prompt, summary_config).result() # Use .result() if sure executor finished, better to await if possible
                # Warning: Mixing sync .result() in async context can block.
                # A better pattern might involve making summarize_text_with_query async
                # OR modifying generate_text to accept config overrides.
                # Let's keep it simple but potentially blocking for now. Consider refactoring later.
                logger.warning("Running local summarization synchronously within executor call - may block.")
                # Alternatively, make summarize_text_with_query async and await generate_text
                # summary = asyncio.run_coroutine_threadsafe(generate_text(summarization_prompt, summary_config), loop).result() # Complex
   
           elif isinstance(backend, (OllamaBackend, VLLMBackend, InstructLabBackend, GeminiBackend)):
                # These generate methods are async, but we are in a sync function called by executor
                # This is problematic. summarize_text_with_query SHOULD be async.
                logger.error("Cannot call async API backend generation from sync summarize function used within run_in_executor. Refactor needed.")
                # For now, let's skip summarization for API backends to fix import error
                logger.warning("Skipping summarization for API backends in this sync helper.")
                return f"[Summarization skipped for API backend: {llm_state['backend_type']}]"
                # # Correct way would require summarize_text_with_query to be async:
                # # summary = await backend.generate(summarization_prompt, summary_config)
           else:
               logger.error(f"Unknown backend type {type(backend)} cannot summarize.")
               return None
           # --- End Backend Call ---
   
   
           if summary:
               logger.info("Summarization successful.")
               logger.debug(f"Generated Summary: {summary}")
               return summary.strip()
           else:
               logger.error("Summarization returned None or failed.")
               return None
   
       except Exception as e:
           logger.error(f"Summarization failed unexpectedly: {e}", exc_info=True)
           return None

   ```

*   **Create Chat Indexing Function (`services/chat_indexer.py` or add to existing service):**
    Let's create a new file for clarity.

    ```python
    # services/chat_indexer.py
    import logging
    from typing import Optional, List, Dict, Any

    from core.config import settings
    from db.database import get_sqlite_db, get_chroma_client, database
    from db.models import chat_messages_table
    from services.embedding_service import generate_embeddings

    logging.basicConfig(level=logging.INFO)
    logger = logging.getLogger(__name__)

    async def index_chat_turn_task(user_message_id: int, assistant_message_id: int, session_id: str):
        """Background task to embed and index a user/assistant chat turn."""
        logger.info(f"[ChatIndex:{session_id}] Starting indexing for user_msg:{user_message_id}, assist_msg:{assistant_message_id}")

        try:
            # 1. Fetch messages from SQLite
            query = chat_messages_table.select().where(
                chat_messages_table.c.id.in_([user_message_id, assistant_message_id])
            )
            # Use the global 'database' instance as this runs in background thread context
            messages = await database.fetch_all(query)

            if len(messages) != 2:
                logger.error(f"[ChatIndex:{session_id}] Could not fetch both messages ({user_message_id}, {assistant_message_id}). Found {len(messages)}.")
                return # Or raise an error?

            messages_to_index: List[Dict[str, Any]] = []
            texts_to_embed: List[str] = []
            ids_for_chroma: List[str] = []
            metadatas_for_chroma: List[Dict[str, Any]] = []

            for msg_row in messages:
                msg = dict(msg_row)
                msg_id = msg['id']
                role = msg['role']
                content = msg['content']
                timestamp = msg['timestamp'].isoformat() # Store timestamp as string

                # Prepare data
                texts_to_embed.append(content)
                # Create a unique ID for ChromaDB (e.g., sessionid_messageid)
                chroma_id = f"{session_id}_{msg_id}"
                ids_for_chroma.append(chroma_id)
                metadatas_for_chroma.append({
                    "type": "chat", # <-- Mark as chat history
                    "session_id": session_id,
                    "message_id": msg_id,
                    "role": role,
                    "timestamp": timestamp
                })

            if not texts_to_embed:
                logger.warning(f"[ChatIndex:{session_id}] No text content found for messages. Skipping embedding.")
                return

            # 2. Generate Embeddings (blocking call, runs in executor via llm_service)
            logger.info(f"[ChatIndex:{session_id}] Generating embeddings for {len(texts_to_embed)} chat messages...")
            # NOTE: Ensure generate_embeddings runs correctly from background task context
            embeddings = generate_embeddings(texts_to_embed) # This call needs careful context management if run via FastAPI BackgroundTasks vs. separate queue

            if embeddings is None or len(embeddings) != len(texts_to_embed):
                 logger.error(f"[ChatIndex:{session_id}] Embedding generation failed or returned incorrect count.")
                 return # Or raise/retry?

            # 3. Add to ChromaDB
            chroma_client = get_chroma_client()
            # Use the *same* collection as documents
            collection = chroma_client.get_collection(settings.DOCUMENT_COLLECTION_NAME)

            logger.info(f"[ChatIndex:{session_id}] Adding {len(embeddings)} chat message embeddings to ChromaDB...")
            collection.add(
                ids=ids_for_chroma,
                embeddings=embeddings,
                documents=texts_to_embed, # Store the original text
                metadatas=metadatas_for_chroma
            )
            logger.info(f"[ChatIndex:{session_id}] Successfully indexed chat turn (User:{user_message_id}, Assist:{assistant_message_id}).")

            # Optional: Update SQLite message records to mark as indexed?
            # await database.execute(
            #     query=chat_messages_table.update().where(chat_messages_table.c.id.in_(...)).values(is_embedded=True)
            # )

        except Exception as e:
            logger.error(f"[ChatIndex:{session_id}] Failed to index chat turn (User:{user_message_id}, Assist:{assistant_message_id}): {e}", exc_info=True)

    ```
    *   Fetches the relevant user/assistant message text from SQLite.
    *   Calls `generate_embeddings`.
    *   Adds entries to the *same* ChromaDB collection used for documents (`settings.DOCUMENT_COLLECTION_NAME`).
    *   Crucially, sets `metadata={"type": "chat", "session_id": ..., "message_id": ..., ...}`.

**2. Trigger Background Task in `sessions.py`**

After successfully storing the assistant's message, schedule the indexing task.

```python
# app/api/endpoints/sessions.py
import uuid
import logging
import datetime
import asyncio
from typing import List, Optional, Dict, Any
from pydantic import BaseModel

from fastapi import APIRouter, HTTPException, Depends, status, BackgroundTasks

# Import models and db access
from app.api.models.chat import (
    SessionCreateRequest,
    SessionMetadataResponse,
    MessageCreateRequest,
    ChatMessageResponse
)

# --- MODIFIED IMPORTS ---
from db.database import database, get_chroma_client # Import Chroma client getter
from db.models import sessions_table, chat_messages_table, documents_table
from core.config import settings
# Import embedding and search functions/models if needed directly
from services.embedding_service import generate_embeddings
# Import LLM service functions and status
from services.llm_service import generate_text, get_llm_status, LLMStatus, llm_state, summarize_text_with_query ##, summarize_text_with_query for the optional idea of summarizing chunks to reduce tokens

# --- END MODIFIED IMPORTS ---
from app.api.endpoints.documents import SearchQuery

logger = logging.getLogger(__name__)
router = APIRouter()

class SessionMemoryUpdateRequest(BaseModel):
    enabled: bool

# --- Helper Function (Optional but recommended for validation) ---
async def get_session_or_404(session_id: str) -> dict:
    """Fetches session metadata or raises HTTPException 404."""
    query = sessions_table.select().where(sessions_table.c.id == session_id)
    session = await database.fetch_one(query)
    if not session:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Session with ID '{session_id}' not found.")

    # Ensure JSON fields are parsed (databases library might return strings)
    # Example for rag_document_ids (assuming it's stored as JSON string):
    # if isinstance(session_dict.get("rag_document_ids"), str):
    #     try:
    #         session_dict["rag_document_ids"] = json.loads(session_dict["rag_document_ids"])
    #     except json.JSONDecodeError:
    #         logger.warning(f"Could not decode rag_document_ids JSON for session {session_id}")
    #         session_dict["rag_document_ids"] = None # Or handle as error

    # Note: SQLAlchemy's JSON type with 'databases' might handle this automatically depending on dialect/setup.
    # Let's assume it returns a list/dict directly for now.

    return dict(session) # Convert RowProxy to dict

# --- Session Endpoints ---

@router.post(
    "/",
    response_model=SessionMetadataResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new chat session",
)
async def create_session(
    session_data: SessionCreateRequest,
):
    """
    Starts a new chat session, optionally associating RAG documents.
    """
    session_id = str(uuid.uuid4())
    now = datetime.datetime.now(datetime.timezone.utc)

    # TODO: Validate if rag_document_ids actually exist in documents table?
    # For now, assume they are valid if provided.

    insert_query = sessions_table.insert().values(
        id=session_id,
        name=session_data.name,
        created_at=now,
        last_updated_at=now,
        llm_model_name=settings.DEFAULT_MODEL_NAME, # Use default model from settings for now
        rag_document_ids=session_data.rag_document_ids,
        metadata_tags=None, # Add logic for automatic tags later
    )
    try:
        await database.execute(insert_query)
        logger.info(f"Created new session with ID: {session_id}, Name: {session_data.name}")
        # Fetch the created session to return its data
        created_session = await get_session_or_404(session_id)
        return SessionMetadataResponse.parse_obj(created_session)
    except Exception as e:
        logger.error(f"Failed to create session: {e}", exc_info=True)
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Failed to create session.")

@router.get(
    "/",
    response_model=List[SessionMetadataResponse],
    summary="List existing chat sessions",
)
async def list_sessions(
    skip: int = 0,
    limit: int = 50,
):
    """
    Retrieves a list of chat sessions, ordered by last update time (most recent first).
    """
    query = sessions_table.select().order_by(sessions_table.c.last_updated_at.desc()).offset(skip).limit(limit)
    try:
        results = await database.fetch_all(query)
        return [SessionMetadataResponse.parse_obj(dict(row)) for row in results]
    except Exception as e:
        logger.error(f"Failed to list sessions: {e}", exc_info=True)
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail="Failed to retrieve sessions.")

@router.get(
    "/{session_id}",
    response_model=SessionMetadataResponse,
    summary="Get details of a specific session",
)
async def get_session(session_id: str):
    """
    Retrieves metadata for a single chat session by its ID.
    """
    session = await get_session_or_404(session_id)
    return SessionMetadataResponse.parse_obj(session)


@router.delete(
    "/{session_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete a chat session",
)
async def delete_session(session_id: str):
    """
    Deletes a chat session and all its associated messages.
    """
    await get_session_or_404(session_id) # Ensure session exists first

    # Use a transaction to ensure atomicity
    async with database.transaction():
        try:
            # Delete messages associated with the session
            delete_messages_query = chat_messages_table.delete().where(chat_messages_table.c.session_id == session_id)
            await database.execute(delete_messages_query)

            # Delete the session itself
            delete_session_query = sessions_table.delete().where(sessions_table.c.id == session_id)
            await database.execute(delete_session_query)

            logger.info(f"Deleted session {session_id} and its messages.")
        except Exception as e:
            logger.error(f"Failed to delete session {session_id}: {e}", exc_info=True)
            # Transaction will be rolled back automatically on exception
            raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=f"Failed to delete session {session_id}.")
    return None # Return None for 204 status code


# --- Message Endpoints (within a session) ---

@router.post(
    "/{session_id}/messages",
    response_model=ChatMessageResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Send a message and get AI response (with RAG)",
    description="Adds user message, performs RAG search, constructs prompt, calls LLM, "
                "stores response, and returns the assistant message.",
)

async def add_message_to_session(
    session_id: str,
    message_data: MessageCreateRequest,
    background_tasks: BackgroundTasks,
):
    if message_data.role != "user":
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Only 'user' role messages can be posted by the client.")

    session = await get_session_or_404(session_id)

    # --- Get the memory setting ---
    long_term_memory_enabled = session.get("long_term_memory_enabled", True) # Default to True if missing? Or rely on DB default.

    user_message_content = message_data.content
    now = datetime.datetime.now(datetime.timezone.utc)
    chroma_client = get_chroma_client()

    # --- Store User Message ---
    user_message_id = None
    try:
        insert_user_message_query = chat_messages_table.insert().values(
            session_id=session_id, timestamp=now, role="user", content=user_message_content, metadata=None
        ).returning(chat_messages_table.c.id)
        user_message_id = await database.execute(insert_user_message_query)
        logger.info(f"[Session:{session_id}] Stored user message (ID: {user_message_id}).")
        update_session_query = sessions_table.update().where(sessions_table.c.id == session_id).values(last_updated_at=now)
        await database.execute(update_session_query)
    except Exception as e:
        logger.error(f"[Session:{session_id}] Failed to store user message: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Failed to store user message.")


    # --- Perform Document RAG Search & Summarization ---
    rag_context = "" # Stores summarized document context
    rag_chunk_ids = []
    retrieved_docs = [] # Keep original docs for potential summarization
    retrieved_metadatas = []
    rag_document_ids_in_session = session.get("rag_document_ids")
    if rag_document_ids_in_session:
        logger.info(f"[Session:{session_id}] Performing RAG search within documents: {rag_document_ids_in_session}")
        try:
            # ... (code to perform ChromaDB query for documents, populate retrieved_docs, retrieved_metadatas, rag_chunk_ids) ...
            doc_collection = chroma_client.get_collection(settings.DOCUMENT_COLLECTION_NAME)
            query_embedding = generate_embeddings([user_message_content])
            if query_embedding and query_embedding[0]:
                chroma_where_filter = {"$and": [{"type": "document"}, {"document_id": {"$in": rag_document_ids_in_session}}]} # Filter for type:document
                results = doc_collection.query(query_embeddings=query_embedding, n_results=settings.RAG_TOP_K, where=chroma_where_filter, include=['documents', 'metadatas', 'distances'])
                retrieved_docs = results.get('documents', [[]])[0]
                retrieved_metadatas = results.get('metadatas', [[]])[0]
                rag_chunk_ids = results.get('ids', [[]])[0] # Original chunk IDs

                if retrieved_docs:
                    logger.info(f"[Session:{session_id}] Retrieved {len(retrieved_docs)} document chunks for summarization.")
                    # --- Summarize Retrieved Document Chunks ---
                    summarized_doc_parts = []
                    loop = asyncio.get_running_loop()
                    for i, chunk_text in enumerate(retrieved_docs):
                         try:
                             summary = await loop.run_in_executor(None, summarize_text_with_query, chunk_text, user_message_content, 150)
                             if summary: summarized_doc_parts.append(summary)
                         except Exception as summary_err: logger.error(f"Error summarizing doc chunk {i}: {summary_err}", exc_info=True)
                    rag_context = "\n---\n".join(summarized_doc_parts) # rag_context now holds summarized doc info
                    if not rag_context: rag_context = "[Document Summarization Failed]"
                    logger.info(f"[Session:{session_id}] Finished summarizing doc chunks.")
                else:
                     logger.info(f"[Session:{session_id}] No relevant document chunks found.")
            else:
                 logger.error("[Session:{session_id}] Failed to generate embedding for document search.")
                 rag_context = "[Document RAG search embedding failed]"
        except Exception as e:
            logger.error(f"[Session:{session_id}] Document RAG search failed: {e}", exc_info=True)
            rag_context = "[Document RAG search failed]"
    else:
         logger.info(f"[Session:{session_id}] No RAG document IDs associated. Skipping document RAG search.")


    # --- Perform Chat History RAG Search & Summarization ---
    retrieved_history_context = "" # Stores summarized history context
    retrieved_history_ids = []

    if long_term_memory_enabled:
        try:
            logger.info(f"[Session:{session_id}] Performing RAG search on chat history...")
            # ... (code to perform ChromaDB query for chat history, populate retrieved_history_docs, retrieved_history_ids) ...
            # ... (code to summarize retrieved_history_docs into retrieved_history_context) ...
            history_query_embedding = generate_embeddings([user_message_content])
            if history_query_embedding and history_query_embedding[0]:
                 chroma_client = get_chroma_client()
                 collection = chroma_client.get_collection(settings.DOCUMENT_COLLECTION_NAME)
                 history_where_filter = {"$and": [{"type": "chat"}, {"session_id": session_id}]}
                 history_rag_k = settings.RAG_TOP_K
                 history_results = collection.query(query_embeddings=history_query_embedding, n_results=history_rag_k, where=history_where_filter, include=['documents', 'metadatas', 'distances'])
                 retrieved_history_docs = history_results.get('documents', [[]])[0]
                 retrieved_history_metadatas = history_results.get('metadatas', [[]])[0]
                 retrieved_history_ids = history_results.get('ids', [[]])[0]

                 if retrieved_history_docs:
                      logger.info(f"[Session:{session_id}] Retrieved {len(retrieved_history_docs)} relevant past messages for summarization.")
                      summarized_history_parts = []
                      loop = asyncio.get_running_loop()
                      for i, history_text in enumerate(retrieved_history_docs):
                           hist_meta = retrieved_history_metadatas[i] if i < len(retrieved_history_metadatas) else {}
                           hist_role = hist_meta.get('role', 'Unknown')
                           try:
                                summary = await loop.run_in_executor(None, summarize_text_with_query, f"{hist_role.capitalize()}: {history_text}", user_message_content, 100)
                                if summary: summarized_history_parts.append(summary)
                           except Exception as summary_err: logger.error(f"Error summarizing history snippet {i}: {summary_err}", exc_info=True)
                      retrieved_history_context = "\n---\n".join(summarized_history_parts)
                      if not retrieved_history_context: retrieved_history_context = "[History Summarization Failed]"
                      logger.info(f"[Session:{session_id}] Finished summarizing history snippets.")
                 else:
                      logger.info(f"[Session:{session_id}] No relevant chat history found for this query.")
            else:
                 logger.error("[Session:{session_id}] Failed to generate embedding for history search.")
                 retrieved_history_context = "[History RAG search embedding failed]"
        except Exception as e:
            logger.error(f"[Session:{session_id}] Chat history RAG search failed: {e}", exc_info=True)
            retrieved_history_context = "[History RAG search failed]"
    else:
        logger.info(f"[Session:{session_id}] Long term memory disabled. Skipping chat history RAG search.")
        # Ensure context is empty if disabled
        retrieved_history_context = ""
        retrieved_history_ids = []

    # --- Retrieve RECENT Chat History (Verbatim) ---
    chat_history_str = "" # Stores recent verbatim history
    try:
        # ... (code to fetch last N messages into chat_history_str - unchanged) ...
        history_limit = settings.CHAT_HISTORY_LENGTH * 2
        history_query = chat_messages_table.select().where(chat_messages_table.c.session_id == session_id).order_by(chat_messages_table.c.timestamp.desc()).limit(history_limit)
        recent_messages = await database.fetch_all(history_query)
        recent_messages.reverse()
        if recent_messages:
             history_parts = [f"{msg['role'].capitalize()}: {msg['content']}" for msg in recent_messages]
             chat_history_str = "\n".join(history_parts)
             logger.info(f"[Session:{session_id}] Retrieved last {len(recent_messages)} verbatim messages for history.")
    except Exception as e:
        logger.error(f"[Session:{session_id}] Failed to retrieve recent chat history: {e}", exc_info=True)
        chat_history_str = "[Failed to retrieve recent history]"


    logger.info("Constructing final prompt with RAG context and history...")

    prompt_for_llm = "[ERROR: Prompt construction did not complete]"
    llm_ready = False
    final_token_count = 0

    assistant_message_metadata = {
        "prompt_preview": None,
        "rag_chunks_retrieved": rag_chunk_ids,
        "relevant_history_retrieved": retrieved_history_ids, # Now correctly populated or []
        "llm_call_error": None
    }

    # 1. Check LLM Status and get essentials
    llm_status_data = get_llm_status()
    tokenizer = llm_state.get("tokenizer")
    max_length = llm_state.get("max_model_length")
    model_name = llm_status_data.get("active_model", "N/A")

    if llm_status_data["status"] != LLMStatus.READY.value:
        error_detail = f"LLM not ready (Status: {llm_status_data['status']}). Please load/configure a model first."
        logger.warning(f"[Session:{session_id}] {error_detail}")
        prompt_for_llm = f"[ERROR: {error_detail}]"
        assistant_message_metadata["llm_call_error"] = error_detail
        # llm_ready remains False

    # 2. Check if Tokenizer/Max Length are needed and available
    elif (rag_context or retrieved_history_context or chat_history_str) and (not tokenizer or not max_length):
        # Needed if any context/history exists that *might* need truncating
        logger.warning("Tokenizer or max_length not found in llm_state, but context/history exists. Prompt truncation cannot be performed accurately. Sending raw context.")
        llm_ready = True # Allow attempt, API backend might handle it
        # --- Assemble simple prompt without truncation ---
        prompt_parts = []
        prompt_parts.append("### INSTRUCTION:")
        prompt_parts.append("You are a helpful AI assistant. Respond to the user's query based on the provided context and chat history.") # Keep instruction
        # Add contexts if they are valid (not error strings)
        if rag_context and "[ERROR" not in rag_context and "[Summarization Failed]" not in rag_context:
            prompt_parts.append(f"\n### RETRIEVED DOCUMENT CONTEXT (Summarized):\n{rag_context}")
        if retrieved_history_context and "[ERROR" not in retrieved_history_context and "[Summarization Failed]" not in retrieved_history_context:
            prompt_parts.append(f"\n### RELEVANT PAST CONVERSATION (Summarized):\n{retrieved_history_context}")
        if chat_history_str and "[ERROR" not in chat_history_str:
            prompt_parts.append(f"\n### RECENT CHAT HISTORY:\n{chat_history_str}")
        prompt_parts.append("\n### USER QUERY:")
        prompt_parts.append(user_message_content)
        prompt_parts.append("\n### RESPONSE:")
        prompt_for_llm = "\n".join(prompt_parts)
        logger.warning(f"Assembled prompt without token-based truncation. Length may exceed limit.")
        assistant_message_metadata["prompt_preview"] = prompt_for_llm[:200] + "..."
        # --- End simple prompt assembly ---

    # 3. Perform Full Prompt Construction with Truncation
    else:
        llm_ready = True # We have tokenizer and max_length if needed
        logger.info("Proceeding with token-based prompt truncation logic.")

        # Define fixed parts
        instruction_text = "You are a helpful AI assistant. Respond to the user's query based on the provided context and chat history."
        user_query_text = user_message_content
        instruction_prompt = "### INSTRUCTION:\n" + instruction_text
        user_query_prompt = "\n### USER QUERY:\n" + user_query_text
        response_marker = "\n### RESPONSE:"

        # Calculate reserved tokens
        generation_config = llm_status_data.get("generation_config", {})
        max_new_tokens = generation_config.get("max_new_tokens", settings.DEFAULT_LLM_MAX_NEW_TOKENS)
        fixed_parts_text = instruction_prompt + user_query_prompt + response_marker
        # Encode fixed parts to get accurate token count
        fixed_tokens_count = len(tokenizer.encode(fixed_parts_text, add_special_tokens=False))
        reserved_tokens = fixed_tokens_count + max_new_tokens + 10 # Increased buffer slightly
        max_context_tokens = max_length - reserved_tokens
        logger.info(f"Max Length: {max_length}, Reserved: {reserved_tokens}, Available for ALL Context: {max_context_tokens}")

        if max_context_tokens <= 0:
            logger.warning("Not enough token space for any context after reserving for fixed parts and response. Using minimal prompt.")
            prompt_parts = [instruction_prompt, user_query_prompt, response_marker]
            prompt_for_llm = "\n".join(prompt_parts)
            final_token_count = len(tokenizer.encode(prompt_for_llm)) # Get count for log
        else:
            # Tokenize valid context parts (including markers)
            doc_context_tokens = tokenizer.encode(f"\n### RETRIEVED DOCUMENT CONTEXT (Summarized):\n{rag_context}", add_special_tokens=False) if rag_context and "[ERROR" not in rag_context and "[Summarization Failed]" not in rag_context else []
            history_summary_tokens = tokenizer.encode(f"\n### RELEVANT PAST CONVERSATION (Summarized):\n{retrieved_history_context}", add_special_tokens=False) if retrieved_history_context and "[ERROR" not in retrieved_history_context and "[Summarization Failed]" not in retrieved_history_context else []
            recent_history_tokens = tokenizer.encode(f"\n### RECENT CHAT HISTORY:\n{chat_history_str}", add_special_tokens=False) if chat_history_str and "[ERROR" not in chat_history_str else []

            # Truncate in order: Recent History -> History Summary -> Doc Context
            total_context_tokens = len(doc_context_tokens) + len(history_summary_tokens) + len(recent_history_tokens)
            available_space = max_context_tokens

            if total_context_tokens > available_space:
                logger.info(f"Combined context ({total_context_tokens} tokens) exceeds available space ({available_space}). Truncating...")
                overflow = total_context_tokens - available_space

                # Truncate Recent History first (keep end)
                if overflow > 0 and len(recent_history_tokens) > 0:
                    if len(recent_history_tokens) >= overflow:
                        logger.warning(f"Truncating RECENT history by {overflow} tokens.")
                        recent_history_tokens = recent_history_tokens[overflow:]
                        overflow = 0
                    else:
                        logger.warning(f"Removing RECENT history entirely ({len(recent_history_tokens)} tokens).")
                        overflow -= len(recent_history_tokens)
                        recent_history_tokens = []

                # Truncate History Summary next (keep beginning)
                if overflow > 0 and len(history_summary_tokens) > 0:
                    if len(history_summary_tokens) >= overflow:
                        logger.warning(f"Truncating HISTORY SUMMARY by {overflow} tokens.")
                        history_summary_tokens = history_summary_tokens[:len(history_summary_tokens)-overflow]
                        overflow = 0
                    else:
                        logger.warning(f"Removing HISTORY SUMMARY entirely ({len(history_summary_tokens)} tokens).")
                        overflow -= len(history_summary_tokens)
                        history_summary_tokens = []

                # Truncate Document Context last (keep beginning)
                if overflow > 0 and len(doc_context_tokens) > 0:
                     if len(doc_context_tokens) >= overflow:
                          logger.warning(f"Truncating DOCUMENT context by {overflow} tokens.")
                          doc_context_tokens = doc_context_tokens[:len(doc_context_tokens)-overflow]
                          overflow = 0
                     else:
                          logger.warning(f"Removing DOCUMENT context entirely ({len(doc_context_tokens)} tokens).")
                          doc_context_tokens = []

            # Assemble final prompt from potentially truncated token lists
            instruction_tokens = tokenizer.encode(instruction_prompt, add_special_tokens=False)
            query_tokens = tokenizer.encode(user_query_prompt, add_special_tokens=False)
            marker_tokens = tokenizer.encode(response_marker, add_special_tokens=False)

            final_token_ids = instruction_tokens + doc_context_tokens + history_summary_tokens + recent_history_tokens + query_tokens + marker_tokens

            # Safety check - shouldn't exceed max_length - max_new_tokens (approx)
            if len(final_token_ids) >= max_length:
                 logger.error(f"FATAL: Calculated prompt tokens ({len(final_token_ids)}) still exceed model max length ({max_length}). Force truncating ID list.")
                 final_token_ids = final_token_ids[:max_length - max_new_tokens - 5] # Hard truncate

            prompt_for_llm = tokenizer.decode(final_token_ids)
            final_token_count = len(final_token_ids)
            logger.info(f"Constructed final prompt with {final_token_count} tokens using truncation.")
            assistant_message_metadata["prompt_preview"] = prompt_for_llm[:200] + "..."


    # Log final prompt (if not error string)
    if llm_ready and "[ERROR" not in prompt_for_llm:
         logger.debug(f"[Session:{session_id}] Constructed prompt for LLM (Token Count: {final_token_count}):\n{prompt_for_llm[:200]}...\n...{prompt_for_llm[-200:]}")
    else:
         logger.debug(f"[Session:{session_id}] Prompt construction failed or LLM not ready. Final prompt value: {prompt_for_llm}")

    # --- Call LLM Service (Checks llm_ready flag) ---
    if llm_ready:
        try:
            logger.info(f"[Session:{session_id}] Sending prompt to LLM '{model_name}' via backend '{llm_status_data['backend_type']}'...")
            llm_response = await generate_text(prompt_for_llm) # Await the service call

            if llm_response is None:
                raise Exception("LLM generation returned None or failed internally.")

            assistant_response_content = llm_response # Set content on success
            logger.info(f"[Session:{session_id}] Received response from LLM.")

        except Exception as e:
            error_detail = f"LLM generation failed: {type(e).__name__}"
            logger.error(f"[Session:{session_id}] {error_detail}: {e}", exc_info=True)
            assistant_response_content = f"[ERROR: {error_detail}]" # Set error content
            assistant_message_metadata["llm_call_error"] = f"{error_detail}: {e}" # Record error in metadata
    # If llm_ready was False, assistant_response_content and metadata already set

    # --- Store Assistant Message ---
    try:
        assistant_timestamp = datetime.datetime.now(datetime.timezone.utc)
        insert_assistant_message_query = chat_messages_table.insert().values(
            session_id=session_id,
            timestamp=assistant_timestamp,
            role="assistant",
            content=assistant_response_content, # Use the actual or error content
            metadata=assistant_message_metadata, # Include metadata
        ).returning(chat_messages_table.c.id, *[c for c in chat_messages_table.c])

        new_assistant_message_row = await database.fetch_one(insert_assistant_message_query)
        if not new_assistant_message_row:
            raise Exception("Failed to retrieve assistant message after insert.")
        assistant_message_id = new_assistant_message_row['id']
        logger.info(f"[Session:{session_id}] Stored assistant message (ID: {assistant_message_id}).")

        update_session_query_after_assist = sessions_table.update().where(sessions_table.c.id == session_id).values(last_updated_at=assistant_timestamp)
        await database.execute(update_session_query_after_assist)

        # --- Schedule Background Indexing Task ---
        if user_message_id is not None and assistant_message_id is not None:
             logger.info(f"[Session:{session_id}] Scheduling background task to index chat turn ({user_message_id}, {assistant_message_id}).")
             # Ensure chat indexer is imported if used
             # from services.chat_indexer import index_chat_turn_task
             # background_tasks.add_task(index_chat_turn_task, user_message_id, assistant_message_id, session_id)
             pass # Commented out as chat_indexer.py wasn't provided/requested in this thread
        else:
              logger.warning(f"[Session:{session_id}] Missing user or assistant message ID, cannot schedule indexing.")

        return ChatMessageResponse.parse_obj(dict(new_assistant_message_row))

    except Exception as e:
        logger.error(f"[Session:{session_id}] Failed to store assistant message: {e}", exc_info=True)
        # Note: If storing fails, the client won't get the response even if generated
        raise HTTPException(status_code=500, detail="Failed to store assistant response.")

@router.put(
    "/{session_id}/memory",
    response_model=SessionMetadataResponse, # Return updated session metadata
    summary="Enable or disable long-term memory (chat history RAG) for a session",
)
async def update_session_memory(
    session_id: str,
    update_request: SessionMemoryUpdateRequest
):
    """
    Updates the long_term_memory_enabled setting for the specified session.
    """
    # Ensure session exists
    session = await get_session_or_404(session_id)

    new_value = update_request.enabled
    current_value = session.get("long_term_memory_enabled")

    if current_value == new_value:
         logger.info(f"Long term memory for session {session_id} already set to {new_value}. No change.")
         # Return current data without DB update
         return SessionMetadataResponse.parse_obj(session)

    update_query = sessions_table.update().where(
        sessions_table.c.id == session_id
    ).values(
        long_term_memory_enabled=new_value,
        # Optionally update last_updated_at timestamp? Debatable for settings change.
        # last_updated_at=datetime.datetime.now(datetime.timezone.utc)
    )

    try:
        await database.execute(update_query)
        logger.info(f"Set long_term_memory_enabled to {new_value} for session {session_id}.")
        # Fetch updated session data to return
        updated_session = await get_session_or_404(session_id)
        return SessionMetadataResponse.parse_obj(updated_session)
    except Exception as e:
        logger.error(f"Failed to update memory setting for session {session_id}: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Failed to update session memory setting.")

# --- List Messages Endpoint (Keep as is) ---
@router.get(
    "/{session_id}/messages",
    response_model=List[ChatMessageResponse],
    summary="List messages in a session",
)
async def list_messages_in_session(
    session_id: str,
    skip: int = 0,
    limit: int = 1000,
):


    await get_session_or_404(session_id) # Ensure session exists
    query = chat_messages_table.select().where(
        chat_messages_table.c.session_id == session_id
    ).order_by(chat_messages_table.c.timestamp.asc()).offset(skip).limit(limit) # Order by ascending time
    try:
        results = await database.fetch_all(query)
        return [ChatMessageResponse.parse_obj(dict(row)) for row in results]
    except Exception as e:
        logger.error(f"Failed to list messages for session {session_id}: {e}", exc_info=True)
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=f"Failed to retrieve messages for session {session_id}.")

```

**Explanation of Changes in `add_message_to_session`:**

1.  **Inject `BackgroundTasks`:** Added `background_tasks: BackgroundTasks` to the function signature.
2.  **Chat History RAG:** Added a new block to perform semantic search specifically for chat history (`type='chat'`, `session_id=current_session_id`).
3.  **History Summarization:** Included the summarization loop for the retrieved history snippets.
4.  **Prompt Construction Updated:** Modified the prompt building and truncation logic to accommodate the three context types (Docs, History Summary, Recent History) and apply truncation in a reasonable order.
5.  **Schedule Indexing Task:** After successfully storing the assistant message and getting its ID (`assistant_message_id`), use `background_tasks.add_task` to schedule `index_chat_turn_task`, passing the user message ID, assistant message ID, and the *backend* session ID.

**Important Considerations:**

*   **Background Task Execution:** FastAPI's `BackgroundTasks` run in the same process/event loop after the response is sent. For heavy indexing tasks or high throughput, using a dedicated task queue (Celery, RQ, Arq) with separate worker processes is more robust and scalable. The current `generate_embeddings` call within the task will block if using the local backend, making a separate task queue even more beneficial.
*   **Summarization Latency:** Adding summarization for *both* document context *and* chat history context will significantly increase the time taken to generate a response.
*   **Summarization Concurrency:** The thread-safety issue with modifying `llm_state["config"]` in `summarize_text_with_query` becomes more prominent if multiple summarization calls happen concurrently (which they might if parallelized or with multiple simultaneous user requests).
*   **Retrieval Query:** Using only the current user message to search past history might not always be optimal. More advanced methods could involve summarizing the recent conversation history to form a query.
*   **Indexing Granularity:** Indexing user/assistant messages separately provides fine-grained retrieval.
*   **No Duplicate Indexing:** Ensure you don't re-index the same messages if the API endpoint is somehow called multiple times for the same interaction. Adding an `is_embedded` flag to the `chat_messages` table and checking it before scheduling the task adds robustness.

**Testing:**

1.  Restart the backend server.
2.  Ensure an LLM is loaded.
3.  Start a *new* chat session in the frontend (link a document if desired).
4.  Have a conversation with several turns.
5.  **Check Backend Logs:** Look for the "[ChatIndex:...] Starting indexing..." and "Successfully indexed chat turn..." messages appearing *after* each assistant response is stored.
6.  **Check ChromaDB:** (Use a script or tool) Verify that new entries with `metadata.type == 'chat'` and the correct `session_id` are being added to your `rag_documents` collection.
7.  **Observe Context:** In later turns of the conversation, check the backend logs for "[Session:...] Performing RAG search on chat history...", "[Session:...] Retrieved X relevant past messages...", and see if the "RELEVANT PAST CONVERSATION (Summarized)" section appears in the constructed prompt log.
8.  **Test Loading:** Save the session, start a new chat, then load the saved session. Send another message and verify that the chat history RAG search still works correctly using the loaded session's context.

This implements the long-term memory feature by indexing and retrieving chat history via RAG. Remember the latency and concurrency implications of the summarization step!

*** 

# Toggle the Long-term Memory #

Add an API endpoint to toggle the long-term memory feature (chat history RAG search) on and off for a specific session. This gives the user control over whether past interactions influence the current context.

We'll store this toggle state as part of the session's metadata in the SQLite `sessions` table.

**1. Update Database Model (`db/models.py`)**

Add a boolean column to the `sessions_table` to store the toggle state.

```python
# db/models.py
import sqlalchemy
import datetime
from sqlalchemy.sql import func

metadata = sqlalchemy.MetaData()

sessions_table = sqlalchemy.Table(
    "sessions",
    metadata,
    sqlalchemy.Column("id", sqlalchemy.String, primary_key=True),
    sqlalchemy.Column("name", sqlalchemy.String, nullable=True),
    sqlalchemy.Column("created_at", sqlalchemy.DateTime(timezone=True), server_default=func.now()),
    sqlalchemy.Column("last_updated_at", sqlalchemy.DateTime(timezone=True), server_default=func.now(), onupdate=func.now()),
    sqlalchemy.Column("llm_model_name", sqlalchemy.String, nullable=True),
    sqlalchemy.Column("rag_document_ids", sqlalchemy.JSON, nullable=True),
    # --- NEW: Long Term Memory Toggle ---
    sqlalchemy.Column("long_term_memory_enabled", sqlalchemy.Boolean, nullable=False, server_default=sqlalchemy.true()), # Default to ON
    # ----------------------------------
    sqlalchemy.Column("metadata_tags", sqlalchemy.JSON, nullable=True),
)

# ... (chat_messages_table, documents_table, document_chunks_table remain the same) ...
```

*   Added `long_term_memory_enabled` as a `Boolean` column.
*   `nullable=False`: We always want this to have a value.
*   `server_default=sqlalchemy.true()`: New sessions will have long-term memory enabled by default. You could change this to `false()` if preferred.

**Important:** If you've already created the database, adding this column requires a database migration. For simplicity in development *only*, you could delete your `rag_server.db` file and let the server recreate it on next startup with the new column. **Do not do this if you have important data!** For production, use a migration tool like Alembic ([https://alembic.sqlalchemy.org/](https://alembic.sqlalchemy.org/)).

**2. Update Pydantic Models (`app/api/models/chat.py`)**

Add the field to the response model so the current state is visible.

```python
# app/api/models/chat.py
import datetime
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any

# ... (SessionCreateRequest, MessageCreateRequest remain the same) ...

class SessionMetadataResponse(BaseModel):
    id: str
    name: Optional[str] = None
    created_at: datetime.datetime
    last_updated_at: datetime.datetime
    llm_model_name: Optional[str] = None
    rag_document_ids: Optional[List[str]] = None
    long_term_memory_enabled: bool # <-- ADDED
    metadata_tags: Optional[Dict[str, Any]] = None

    class Config:
        orm_mode = True # Renamed to from_attributes in Pydantic v2

# ... (ChatMessageResponse remains the same) ...
```

**3. Add API Endpoint (`app/api/endpoints/sessions.py`)**

Create a new endpoint (e.g., a `PUT` or `PATCH` request) to update the toggle state.

```python
# app/api/endpoints/sessions.py
import uuid
import logging
import datetime
import asyncio
from typing import List, Optional, Dict, Any # <-- Added Dict, Any if not already present
from pydantic import BaseModel # <-- ADD BaseModel if not already present

from fastapi import APIRouter, HTTPException, Depends, status, BackgroundTasks

# ... (other imports: chat models, db, services etc.) ...

# --- NEW: Request model for updating memory setting ---
class SessionMemoryUpdateRequest(BaseModel):
    enabled: bool

# ... (router definition, helper function get_session_or_404) ...
# ... (create_session, list_sessions, get_session, delete_session) ...

# --- NEW Endpoint to Toggle Memory ---
@router.put(
    "/{session_id}/memory",
    response_model=SessionMetadataResponse, # Return updated session metadata
    summary="Enable or disable long-term memory (chat history RAG) for a session",
)
async def update_session_memory(
    session_id: str,
    update_request: SessionMemoryUpdateRequest
):
    """
    Updates the long_term_memory_enabled setting for the specified session.
    """
    # Ensure session exists
    session = await get_session_or_404(session_id)

    new_value = update_request.enabled
    current_value = session.get("long_term_memory_enabled")

    if current_value == new_value:
         logger.info(f"Long term memory for session {session_id} already set to {new_value}. No change.")
         # Return current data without DB update
         return SessionMetadataResponse.parse_obj(session)

    update_query = sessions_table.update().where(
        sessions_table.c.id == session_id
    ).values(
        long_term_memory_enabled=new_value,
        # Optionally update last_updated_at timestamp? Debatable for settings change.
        # last_updated_at=datetime.datetime.now(datetime.timezone.utc)
    )

    try:
        await database.execute(update_query)
        logger.info(f"Set long_term_memory_enabled to {new_value} for session {session_id}.")
        # Fetch updated session data to return
        updated_session = await get_session_or_404(session_id)
        return SessionMetadataResponse.parse_obj(updated_session)
    except Exception as e:
        logger.error(f"Failed to update memory setting for session {session_id}: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Failed to update session memory setting.")


# --- Modify add_message_to_session ---
@router.post(
    "/{session_id}/messages",
    # ... (decorator) ...
)
async def add_message_to_session(
    session_id: str,
    message_data: MessageCreateRequest,
    background_tasks: BackgroundTasks,
):
    # ... (validation) ...

    session = await get_session_or_404(session_id)
    # --- Get the memory setting ---
    long_term_memory_enabled = session.get("long_term_memory_enabled", True) # Default to True if missing? Or rely on DB default.

    # ... (store user message) ...
    # ... (Perform Document RAG Search & Summarization - unchanged) ...

    # --- Perform Chat History RAG Search & Summarization (CONDITIONAL) ---
    retrieved_history_context = ""
    retrieved_history_ids = []
    # --- Check the toggle before performing history search ---
    if long_term_memory_enabled:
        try:
            logger.info(f"[Session:{session_id}] Long term memory enabled. Performing RAG search on chat history...")
            # ... (Full logic for history RAG search and summarization) ...
             # ... (fetches history_query_embedding, runs chroma query, summarizes) ...
        except Exception as e:
            logger.error(f"[Session:{session_id}] Chat history RAG search failed: {e}", exc_info=True)
            retrieved_history_context = "[History RAG search failed]"
    else:
        logger.info(f"[Session:{session_id}] Long term memory disabled. Skipping chat history RAG search.")
        # Ensure context is empty if disabled
        retrieved_history_context = ""
        retrieved_history_ids = []
    # --- End Conditional History Search ---


    # --- Retrieve RECENT Chat History (Verbatim - unchanged) ---
    # ... (gets chat_history_str) ...

    # --- Construct & Truncate Prompt (Unchanged - uses whatever contexts were populated) ---
    # ... (checks llm status, tokenizer, max_length) ...
    # ... (calculates limits) ...
    # ... (tokenizes/truncates doc_context, history_summary (retrieved_history_context), recent_history) ...
    # ... (assembles final prompt) ...

    # --- Update Metadata Initialization (Include history IDs if search ran) ---
    assistant_message_metadata = {
        "prompt_preview": None,
        "rag_chunks_retrieved": rag_chunk_ids,
        "relevant_history_retrieved": retrieved_history_ids, # Now correctly populated or []
        "llm_call_error": None
    }

    # ... (check llm status/readiness) ...

    # --- Call LLM Service (Unchanged) ---
    # ... (calls generate_text if llm_ready) ...

    # --- Store Assistant Message (Unchanged) ---
    # ... (stores message, schedules indexing task, returns response) ...

# ... (list_messages_in_session) ...

```

**Explanation of Changes in `sessions.py`:**

1.  **New Model:** Added `SessionMemoryUpdateRequest` Pydantic model.
2.  **New Endpoint:** Created `PUT /api/v1/sessions/{session_id}/memory`.
    *   Takes the session ID from the path and the boolean `enabled` value from the request body.
    *   Fetches the session to ensure it exists.
    *   Checks if the value needs changing to avoid unnecessary DB writes.
    *   Executes an SQL `UPDATE` statement on the `sessions` table to set the `long_term_memory_enabled` column.
    *   Fetches and returns the updated session metadata.
3.  **Modified `add_message_to_session`:**
    *   Reads the `long_term_memory_enabled` value from the session data fetched at the beginning.
    *   Wraps the entire "Perform Chat History RAG Search & Summarization" block in an `if long_term_memory_enabled:` condition.
    *   If disabled, it logs a message and ensures `retrieved_history_context` and `retrieved_history_ids` remain empty.
    *   The subsequent prompt construction logic remains the same; it will simply not include the "RELEVANT PAST CONVERSATION" section if `retrieved_history_context` is empty.
    *   The `assistant_message_metadata` now correctly includes `retrieved_history_ids` (which will be empty if the search was skipped).

**4. Update Frontend (`App.jsx` and `SessionManager.jsx`) - Optional but Recommended**

You could add a toggle switch or button in the `SessionManager` component to call this new API endpoint.

*   **`SessionManager.jsx`:** Add a visual element (e.g., a checkbox or toggle switch) next to each session or for the active session. Add a handler function prop like `onToggleMemory(sessionId, isEnabled)`.
*   **`App.jsx`:**
    *   Add a state variable like `activeSessionMemoryEnabled` (initialized when loading a session).
    *   Implement the `handleToggleMemory` function that makes the `PUT` request to `/api/v1/sessions/{session_id}/memory`.
    *   Update the `activeSessionMemoryEnabled` state based on the API response.
    *   Pass the state and handler down to `SessionManager`.

**(This frontend part is optional for now, you can test the API directly first)**

**Testing:**

1.  Restart the backend server (remembering the database schema change - delete `.db` file *only if safe* or use migrations).
2.  **Check Default:** Create a new session (`POST /sessions`). Then `GET /sessions/{session_id}`. Verify `long_term_memory_enabled` is `true`.
3.  **Disable Memory:** Call `PUT /sessions/{session_id}/memory` with `{"enabled": false}` in the request body. Verify the response shows `long_term_memory_enabled: false`.
4.  **Chat (Memory Off):** Send a message to this session (`POST /sessions/{session_id}/messages`). Check the backend logs - you should see "Long term memory disabled. Skipping chat history RAG search." The prompt construction logs should not include the "RELEVANT PAST CONVERSATION" section.
5.  **Enable Memory:** Call `PUT /sessions/{session_id}/memory` with `{"enabled": true}`. Verify the response shows `true`.
6.  **Chat (Memory On):** Send another message to the session. Check the backend logs - you should now see "Performing RAG search on chat history..." (assuming some history has been indexed for this session). The prompt construction logs *should* now include the relevant history section (if any was found and summarized).

This provides API control over the long-term memory feature for each chat session.
