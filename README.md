# Project Documentation

## API Documentation (/api)

The API is built using FastAPI and provides endpoints for status checks and interacting with PDF-based conversational agents.

**Core Components:**

*   `api/main.py`: Initializes the FastAPI application, sets global configurations (like title, version, docs URLs based on `api/settings.py`), includes routers, and configures CORS middleware.
*   `api/settings.py`: Defines API settings using Pydantic's `BaseSettings`, allowing configuration via environment variables (e.g., `runtime_env`, `docs_enabled`, `cors_origin_list`).
*   `api/routes/v1_router.py`: Combines different route modules under a `/v1` prefix.
*   `api/routes/endpoints.py`: Defines endpoint paths as constants for consistency.

**Authentication:**

*(Currently, no explicit authentication mechanism is visible in the provided route files (`status_routes.py`, `pdf_routes.py`). If authentication (e.g., OAuth2, API Keys) is added, it should be documented here, explaining how to obtain credentials and use them with protected endpoints.)*

**Endpoints:**

### Status Routes (`api/routes/status_routes.py`)

*   **`GET /v1/ping`**
    *   **Summary:** Simple health check endpoint.
    *   **Description:** Returns a basic "pong" response to indicate the API is running. Useful for load balancers or basic availability checks.
    *   **Tags:** `status`
    *   **Response (200 OK):**
        ```json
        {
          "ping": "pong"
        }
        ```

*   **`GET /v1/health`**
    *   **Summary:** Provides a more detailed health status.
    *   **Description:** Returns the operational status, identifies the router, confirms the path, and provides the current UTC timestamp.
    *   **Tags:** `status`
    *   **Response (200 OK):**
        ```json
        {
          "status": "success",
          "router": "status",
          "path": "/health",
          "utc": "YYYY-MM-DDTHH:MM:SS.ffffffZ"
        }
        ```

### PDF Conversation Routes (`api/routes/pdf_routes.py`)

These endpoints facilitate interaction with LLM-powered conversations based on PDF documents. Two conversation types are supported: "RAG" (Retrieval-Augmented Generation) and "AUTO" (Autonomous, potentially with function calling). Conversations are persisted using `llm.storage.pdf_conversation_storage`.

*   **`POST /v1/pdf/conversation/load-knowledge-base`**
    *   **Summary:** Trigger loading of the PDF knowledge base.
    *   **Description:** Initializes the knowledge base vectors (`llm.knowledge_base.pdf_knowledge_base`) if not already loaded. This might involve fetching and processing PDFs. Called typically at application startup or on demand.
    *   **Tags:** `PDF`
    *   **Response (200 OK):**
        ```json
        {
          "message": "Knowledge base loaded"
        }
        ```

*   **`POST /v1/pdf/conversation/create`**
    *   **Summary:** Create a new PDF-based conversation.
    *   **Description:** Initializes a new conversation session for a given user and conversation type. Stores the initial conversation state and returns a unique `conversation_id`.
    *   **Tags:** `PDF`
    *   **Request Body:** (`CreateConversationRequest`)
        ```json
        {
          "user_name": "string",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** (`CreateConversationResponse`)
        ```json
        {
          "conversation_id": "string (UUID)",
          "chat_history": [] // Initially empty chat history
        }
        ```
    *   **Response (500 Internal Server Error):** If conversation creation fails in the backend.

*   **`POST /v1/pdf/conversation/chat`**
    *   **Summary:** Send a message to an existing conversation.
    *   **Description:** Sends the user's message to the specified conversation (`conversation_id`). The LLM processes the message, potentially using the knowledge base (for RAG) or function calls (for AUTO), and returns a response. Supports streaming (`text/event-stream`) or a single JSON response.
    *   **Tags:** `PDF`
    *   **Request Body:** (`ChatRequest`)
        ```json
        {
          "message": "string",
          "stream": true, // or false
          "conversation_id": "string (UUID)", // Required
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):**
        *   If `stream: true`: A `StreamingResponse` with `text/event-stream` content.
        *   If `stream: false`: A JSON object containing the LLM's complete response (structure depends on the `phi.Conversation.run` implementation).

*   **`POST /v1/pdf/conversation/history`**
    *   **Summary:** Retrieve the chat history for a conversation.
    *   **Description:** Fetches the stored message history (user inputs and assistant responses) for the specified `conversation_id`.
    *   **Tags:** `PDF`
    *   **Request Body:** (`ChatHistoryRequest`)
        ```json
        {
          "conversation_id": "string (UUID)",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** A list of chat history dictionaries, e.g.:
        ```json
        [
          {"role": "user", "content": "Hello"},
          {"role": "assistant", "content": "Hi there!"}
        ]
        ```

*   **`POST /v1/pdf/conversation/get`**
    *   **Summary:** Get metadata for a specific conversation.
    *   **Description:** Retrieves the database record (`ConversationRow`) containing metadata about the conversation (like `id`, `user_name`, `created_at`, `updated_at`, `name`, `metadata`).
    *   **Tags:** `PDF`
    *   **Request Body:** (`GetConversationRequest`)
        ```json
        {
          "conversation_id": "string (UUID)",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** A JSON object representing the `ConversationRow` (structure defined by `phi.storage.conversation.postgres.ConversationRow`).

*   **`POST /v1/pdf/conversation/get-all`**
    *   **Summary:** Get all conversation metadata for a user.
    *   **Description:** Retrieves all `ConversationRow` records associated with the specified `user_name`.
    *   **Tags:** `PDF`
    *   **Request Body:** (`GetAllConversationsRequest`)
        ```json
        {
          "user_name": "string"
        }
        ```
    *   **Response (200 OK):** A list of `ConversationRow` JSON objects.

*   **`POST /v1/pdf/conversation/get-all-ids`**
    *   **Summary:** Get all conversation IDs for a user.
    *   **Description:** Retrieves a list of just the `conversation_id` strings associated with the specified `user_name`.
    *   **Tags:** `PDF`
    *   **Request Body:** (`GetAllConversationIdsRequest`)
        ```json
        {
          "user_name": "string"
        }
        ```
    *   **Response (200 OK):** A list of strings, e.g.:
        ```json
        [
          "uuid-1",
          "uuid-2"
        ]
        ```

*   **`POST /v1/pdf/conversation/rename`**
    *   **Summary:** Manually rename a conversation.
    *   **Description:** Updates the `name` associated with a specific `conversation_id` in the database.
    *   **Tags:** `PDF`
    *   **Request Body:** (`RenameConversationRequest`)
        ```json
        {
          "name": "New Conversation Name",
          "conversation_id": "string (UUID)",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** (`RenameConversationResponse`)
        ```json
        {
          "name": "New Conversation Name",
          "conversation_id": "string (UUID)"
        }
        ```

*   **`POST /v1/pdf/conversation/autorename`**
    *   **Summary:** Automatically rename a conversation using the LLM.
    *   **Description:** Triggers an LLM call to generate a concise name for the conversation based on its content and updates the name in the database.
    *   **Tags:** `PDF`
    *   **Request Body:** (`AutoRenameConversationRequest`)
        ```json
        {
          "conversation_id": "string (UUID)",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** (`AutoRenameConversationResponse`)
        ```json
        {
          "name": "LLM Generated Name",
          "conversation_id": "string (UUID)"
        }
        ```

*   **`POST /v1/pdf/conversation/end`**
    *   **Summary:** Mark a conversation as ended.
    *   **Description:** Updates the conversation's status in the database to indicate it has concluded (e.g., sets an `ended_at` timestamp). Does not delete the conversation.
    *   **Tags:** `PDF`
    *   **Request Body:** (`EndConversationRequest`)
        ```json
        {
          "conversation_id": "string (UUID)",
          "conversation_type": "RAG" // or "AUTO"
        }
        ```
    *   **Response (200 OK):** The updated `ConversationRow` JSON object.

## Database Schema (/db, /llm)

The application uses a PostgreSQL database, configured via `db/settings.py` and accessed using SQLAlchemy sessions managed in `db/session.py`. Alembic is used for migrations (`db/migrations/`).

The database primarily stores:

1.  **Application Data:** Defined by SQLAlchemy models inheriting from `db.tables.base.Base` within the `public` schema. *(Currently, no specific application tables are defined in the provided files.)*
2.  **LLM Data:** Managed by the `phi-ai` library components, stored within the `llm` schema.

**Key Tables (in `llm` schema):**

*   **`pdf_conversations`** (Managed by `PgConversationStorage` in `llm/storage.py`):
    *   Stores details about each conversation session initiated via the API.
    *   **Columns (inferred):** `id` (UUID/String, PK), `name` (String), `user_name` (String), `created_at` (Timestamp), `updated_at` (Timestamp), `ended_at` (Timestamp, nullable), `metadata` (JSONB - stores `conversation_type`, etc.), `memory` (JSONB - stores chat history).
*   **Vector Store Tables** (Managed by `PgVector` in `llm/knowledge_base.py`):
    *   Used to store vector embeddings of PDF document chunks for efficient similarity search (RAG).
    *   **Tables:**
        *   `url_pdf_documents`: Stores embeddings for PDFs fetched from URLs (e.g., `meals-more-recipes.pdf`).
        *   `local_pdf_documents`: Stores embeddings for PDFs located in the `data/pdfs/` directory.
        *   `pdf_documents`: Stores embeddings for the combined knowledge base.
    *   **Columns (typical for PgVector):** `id` (UUID/String, PK), `document_id` (String), `content` (Text - the text chunk), `embedding` (Vector), `metadata` (JSONB - e.g., source file, page number).

## LLM Interactions (/llm)

The application integrates with OpenAI's language models (specifically `gpt-4` as configured in `llm/settings.py`) for conversational AI features related to PDF documents.

**Core Components:**

*   **`llm/settings.py`**: Defines LLM-related settings (model names, default parameters like `max_tokens`, `temperature`) via `LLMSettings`.
*   **`llm/knowledge_base.py`**:
    *   Defines the knowledge sources for the LLM.
    *   Uses `PDFUrlKnowledgeBase` to process PDFs from URLs.
    *   Uses `PDFKnowledgeBase` to process local PDFs from the `data/pdfs` directory.
    *   Combines these into a `CombinedKnowledgeBase`.
    *   Configures `PgVector` to store the document embeddings in the PostgreSQL database (`llm` schema, various tables).
*   **`llm/storage.py`**:
    *   Configures `PgConversationStorage` to save conversation history and metadata to the `llm.pdf_conversations` table in PostgreSQL.
*   **`llm/conversations/pdf_rag.py`**:
    *   Defines the `get_pdf_rag_conversation` function.
    *   Creates a `phi.Conversation` object configured for RAG.
    *   Uses `OpenAIChat` (`gpt-4`) as the LLM.
    *   Connects the `pdf_knowledge_base` for retrieving relevant context.
    *   Connects the `pdf_conversation_storage` for persistence.
    *   Includes a system prompt guiding the LLM to use the provided knowledge.
    *   Uses a specific `user_prompt_function` that formats the user message along with retrieved `references` from the knowledge base.
*   **`llm/conversations/pdf_auto.py`**:
    *   Defines the `get_pdf_auto_conversation` function.
    *   Creates a `phi.Conversation` object potentially configured for more autonomous behavior (e.g., function calling, though specific functions aren't defined here).
    *   Uses `OpenAIChat` (`gpt-4`).
    *   Connects the `pdf_knowledge_base` and `pdf_conversation_storage`.
    *   Includes a system prompt guiding the LLM.
    *   Sets `function_calls=True`.

**Interaction Flow (RAG Example):**

1.  User sends a message via the `POST /v1/pdf/conversation/chat` endpoint for a "RAG" type conversation.
2.  The API retrieves the corresponding `Conversation` object using `get_pdf_rag_conversation`.
3.  The `Conversation.run()` method is called.
4.  The user's message is used to query the `pdf_knowledge_base` (which searches the `pdf_documents` PgVector table).
5.  Relevant document chunks (`references`) are retrieved.
6.  The `user_prompt_function` formats the prompt including the original message and the retrieved `references`.
7.  The system prompt, chat history (if enabled), and the formatted user prompt are sent to the OpenAI API (`gpt-4`).
8.  The LLM generates a response based on the input and its training.
9.  The response is streamed back (or returned as a single object) to the user via the API.
10. The user message and the assistant's response are saved to the `llm.pdf_conversations` table via `pdf_conversation_storage`.
