# AI-PDF-Chatbot-RAG-OpenAI-LangChain-FAISS
The application extracts and chunks PDF content, creates a vector knowledge base, retrieves relevant information through semantic search, and generates contextual answers using OpenAI. Implemented PostgreSQL authentication, role-based Admin/User access, Streamlit UI, and Docker-based database infrastructure.

# AI PDF Chatbot with Role-Based Authentication

## Overview

This project is a Streamlit-based PDF question-answering application
with:

* PostgreSQL-backed user authentication
* Password hashing and verification
* Role-based access control (`admin` / `user`)
* Separate admin and user application entry points
* PDF document upload and text extraction
* Text chunking with LangChain
* OpenAI embeddings
* FAISS vector similarity search
* LLM-based question answering
* Docker Compose configuration for PostgreSQL

The application is organized around a login/role-selection layer and
separate document-processing experiences for administrators and regular
users.

> **Security note:** The supplied source files contain hard-coded API
> keys and database credentials. These values should be considered
> compromised and rotated. Production deployments should load secrets
> from environment variables or a secrets manager.

---

## Architecture

```text
                         +----------------------+
                         |      Streamlit       |
                         |      main.py         |
                         |  Login / Role Check  |
                         +----------+-----------+
                                    |
                     +--------------+--------------+
                     |                             |
                  admin                         user
                     |                             |
          +----------v---------+        +----------v---------+
          |      admin.py      |        |      user.py       |
          | PDF upload         |        | Existing FAISS     |
          | Text extraction    |        | vector store       |
          | Chunking           |        | Question input     |
          | Embeddings         |        | Similarity search  |
          | FAISS creation     |        | LLM answer         |
          +----------+---------+        +----------+---------+
                     |                             |
                     +-------------+---------------+
                                   |
                          +--------v--------+
                          |  OpenAI / LLM   |
                          | Embeddings + QA |
                          +-----------------+

             +--------------------------------------+
             |              PostgreSQL               |
             | auth_app                              |
             | users / roles / user_role             |
             +------------------+-------------------+
                                |
                         authentication_db.py
```

---

## Project Components

---

File                                Responsibility

---

`main.py`                           Main Streamlit login page,
authentication, role lookup, and
routing

`authentication_db.py`              PostgreSQL connection, credential
lookup, password verification, role
lookup, and user creation

`admin.py`                          Administrator-side PDF upload,
extraction, chunking, embedding
generation, FAISS creation/loading,
and Q&A

`user.py`                           User-side Q&A against an existing
FAISS vector store

`registration_form.py`              Streamlit registration form for
creating users

`docker-compose.yml`                PostgreSQL 16 container and
persistent database volume

`password.yml`                      Legacy/configuration-based
Streamlit Authenticator credentials

`authentication.py`                 Legacy Streamlit Authenticator
login implementation

`app.py`                            Earlier/alternate PDF chatbot
implementation

`app_deepseek.py`                   Alternate chatbot implementation
using DeepSeek for answer
generation

`authentication_db_old.py`          Earlier database authentication
implementation
--------------

---

# 1. `main.py`

`main.py` acts as the primary application entry point.

### Responsibilities

1. Imports authentication/database helpers.
2. Establishes a PostgreSQL connection through `get_db_connection()`.
3. Displays a Streamlit login form.
4. Authenticates the supplied username/password.
5. Retrieves the user's role.
6. Stores successful login information in `st.session_state`.
7. Starts the appropriate Streamlit application based on the role.
8. Provides a logout action.
9. Provides a link to the registration application.

### Authentication flow

```text
User enters username/password
            |
            v
     authenticate_user()
            |
            v
     PostgreSQL users table
            |
            v
     Password verification
            |
            +---- failure ---> Invalid credentials
            |
           success
            |
            v
       verify_user()
            |
            v
       Role lookup
            |
       +----+----+
       |         |
     admin      user
       |         |
       v         v
   admin.py    user.py
```

### Session state

Successful authentication sets:

```python
st.session_state["logged_in"]
st.session_state["username"]
st.session_state["name"]
```

Logout clears the Streamlit session state.

### Current routing implementation

The supplied implementation launches:

```text
streamlit run admin.py
```

or:

```text
streamlit run user.py
```

using `subprocess.run()`.

This is functional as a prototype, but it is not an ideal production
architecture because a Streamlit application normally should not spawn
additional Streamlit servers for each authenticated user. A cleaner
architecture would route users within one Streamlit process using
session state and role-specific UI.

---

# 2. `authentication_db.py`

This module contains the database and authentication logic used by
`main.py`.

## Database connection

`get_db_connection()` creates a PostgreSQL connection using `psycopg2`.

Current configuration expects:

```text
Database: auth_app
User: postgres
Host: localhost
Port: 5432
Password: configured database password
```

The Docker Compose configuration creates the same database name and
PostgreSQL service.

### Recommended configuration

Credentials should not be embedded in Python source code.

Use environment variables instead:

```python
import os
import psycopg2

conn = psycopg2.connect(
    dbname=os.environ["POSTGRES_DB"],
    user=os.environ["POSTGRES_USER"],
    password=os.environ["POSTGRES_PASSWORD"],
    host=os.environ.get("POSTGRES_HOST", "localhost"),
    port=os.environ.get("POSTGRES_PORT", "5432"),
)
```

---

## `get_users(username)`

Queries the `users` table for a specific username.

Conceptually:

```sql
SELECT username, password
FROM users
WHERE username = %s;
```

The username is passed as a parameter, which avoids constructing the SQL
statement through string concatenation.

Returns:

```text
(username, password_hash)
```

or `None` when the user is not found.

---

## `get_user_role(username)`

Resolves the user's role through the relationship between:

* `users`
* `user_role`
* `roles`

The query follows this relationship:

```text
users
  |
  | user_id
  v
user_role
  |
  | role_id
  v
roles
```

The SQL retrieves:

```sql
SELECT r.role_type
FROM roles r
JOIN user_role ur ON r.id = ur.role_id
JOIN users u ON u.id = ur.user_id
WHERE u.username = %s;
```

The returned role is expected to be something such as:

```text
admin
```

or:

```text
user
```

---

## `verify_user(username, entered_password)`

This function:

1. Retrieves the stored password hash.
2. Uses Werkzeug's `check_password_hash()`.
3. Compares the supplied password against the stored hash.
4. Retrieves the user's role after successful password validation.
5. Returns the role.

Example result:

```python
"admin"
```

or:

```python
"user"
```

Invalid credentials return `None`.

---

## `add_user(username, password)`

Creates a password hash and inserts a new user into PostgreSQL.

The module currently uses:

```python
stauth.Hasher([password]).generate()[0]
```

while the registration form uses Werkzeug's:

```python
generate_password_hash()
```

This should be standardized to one password-hashing implementation
across the application. Since `verify_user()` uses Werkzeug's
`check_password_hash()`, using Werkzeug consistently is the simplest
approach.

---

## `authenticate_user(username, password)`

This is the credential-validation function used by `main.py`.

It:

1. Gets the user record.
2. Verifies the password hash.
3. Returns a small user object on success.

Example:

```python
{
    "username": "example",
    "name": "example"
}
```

Otherwise it returns `None`.

---

# 3. `admin.py`

`admin.py` implements the document ingestion and question-answering
workflow.

## High-level workflow

```text
PDF upload
    |
    v
PdfReader
    |
    v
Extract page text
    |
    v
RecursiveCharacterTextSplitter
    |
    v
Text chunks
    |
    v
OpenAI Embeddings
    |
    v
FAISS vector store
    |
    v
Similarity search
    |
    v
Top matching chunks
    |
    v
LLM QA chain
    |
    v
Answer displayed in Streamlit
```

---

## PDF ingestion

The Streamlit sidebar exposes a PDF uploader:

```python
st.file_uploader(
    "Upload a PDF File",
    type="pdf"
)
```

When a file is supplied, `PyPDF2.PdfReader` reads the PDF and extracts
text page by page.

The extracted page text is concatenated into one string.

### Limitation

`PdfReader.extract_text()` works primarily with text-based PDFs.
Scanned/image-only PDFs may require OCR before meaningful text can be
extracted.

---

## Text chunking

The extracted text is divided using:

```python
RecursiveCharacterTextSplitter(
    separators=["\n"],
    chunk_size=1000,
    chunk_overlap=150,
    length_function=len
)
```

### Parameters

Parameter             Value Purpose

---

`chunk_size`           1000 Approximate maximum chunk length
`chunk_overlap`         150 Repeated context between adjacent chunks
separator           newline Preferred split boundary

Chunk overlap helps preserve context when an answer spans two
neighboring chunks.

---

# 4. Embeddings and FAISS

The application creates OpenAI embeddings using `OpenAIEmbeddings`.

Each text chunk is converted into a numerical vector representing its
semantic meaning.

FAISS is then used as the local vector database.

### Initial creation

If the configured FAISS path does not exist:

```python
FAISS.from_texts(chunks, embeddings)
```

creates the vector store.

It is then persisted with:

```python
vector_store.save_local(VECTOR_STORE_PATH)
```

### Existing vector store

If the directory already exists:

```python
FAISS.load_local(
    VECTOR_STORE_PATH,
    embeddings,
    allow_dangerous_deserialization=True
)
```

loads the existing index.

### Important implementation consideration

The current design does not explicitly associate a vector index with a
document identifier/version. Uploading another PDF while an existing
FAISS directory exists can therefore result in the existing index being
reused rather than rebuilding it for the newly uploaded document.

For a production implementation, the application should manage indexes
using document IDs or a persistent vector-store strategy.

---

# 5. Question Answering

When a user enters a question:

```python
vector_store.similarity_search(user_question, k=3)
```

retrieves the three most semantically similar chunks.

The retrieved chunks are passed to a LangChain QA chain.

The current implementation uses:

```text
ChatOpenAI
model: gpt-4o-mini
temperature: 0
max_tokens: 50
```

and:

```python
load_qa_chain(llm, chain_type="stuff")
```

The `"stuff"` chain combines the retrieved documents and supplies them
to the model together with the question.

### RAG pattern

The application is therefore implementing a basic Retrieval-Augmented
Generation (RAG) architecture:

```text
Question
   |
   v
Embedding / similarity retrieval
   |
   v
Top 3 document chunks
   |
   v
LLM prompt/context
   |
   v
Generated answer
```

---

# 6. `user.py`

`user.py` is the regular-user chatbot interface.

Unlike `admin.py`, it does not upload or create the reference PDF index.
Instead, it expects a previously generated FAISS vector store.

## Workflow

```text
User opens application
        |
        v
Load OpenAI embeddings
        |
        v
Load existing FAISS index
        |
        v
User enters question
        |
        v
Similarity search (k=3)
        |
        v
ChatOpenAI / gpt-4o-mini
        |
        v
Answer
```

This creates a logical separation between:

* **Administrator:** manages/builds the reference knowledge base.
* **Regular user:** queries the existing knowledge base.

---

# 7. `registration_form.py`

This module provides the registration interface.

The form accepts:

* Username
* Password

The password is hashed using Werkzeug:

```python
generate_password_hash(new_password)
```

The application checks whether the username already exists before
inserting the new record.

A UUID is generated for the new user:

```python
uuid.uuid4()
```

The record is inserted into the `users` table and the transaction is
committed.

### Current implementation issue

The form checks:

```python
if st.form_submit_button:
```

instead of testing the value returned by the submit button.

A more reliable pattern is:

```python
submit_button = st.form_submit_button("Register")

if submit_button:
    ...
```

The password hash is also currently calculated before the submit action.
Hashing should occur only after the user submits the form and after
basic validation.

---

# 8. PostgreSQL and Docker

`docker-compose.yml` defines a PostgreSQL 16 service.

```yaml
services:
  db:
    image: postgres:16
    container_name: auth-postgres
```

The database is:

```text
auth_app
```

and the PostgreSQL user is:

```text
postgres
```

The database password is supplied through:

```text
${DB_PASSWORD}
```

Port `5432` is exposed.

A named Docker volume:

```text
auth_pg_data
```

persists database data outside the lifecycle of the PostgreSQL
container.

## Start PostgreSQL

Create a `.env` file:

```env
DB_PASSWORD=change-me
```

Then start the database:

```bash
docker compose up -d
```

Verify the container:

```bash
docker compose ps
```

Stop the environment:

```bash
docker compose down
```

The named volume is retained unless explicitly removed.

---

# 9. Database Model

Based on the authentication queries, the current application expects at
least the following logical tables:

```text
users
roles
user_role
```

## `users`

Expected fields include:

```text
id
username
password
```

The older implementation also referenced:

```text
name
email
```

but the current `authentication_db.py` primarily uses `id`, `username`,
and `password`.

## `roles`

Expected fields:

```text
id
role_type
```

## `user_role`

Expected relationship fields:

```text
user_id
role_id
```

### Relationship

```text
users.id
   |
   | 1:N
   v
user_role.user_id

user_role.role_id
   |
   | N:1
   v
roles.id
```

The exact DDL/schema creation scripts are not included in the supplied
project files, so the complete database schema cannot be documented
beyond these fields and relationships supported by the source code.

---

# 10. Legacy Authentication Implementations

The project contains multiple earlier authentication approaches.

## `authentication.py`

This version uses `streamlit_authenticator.Authenticate` with
credentials loaded from `password.yml`.

The configuration contains:

```yaml
credentials:
  usernames:
    admin:
      ...
    user:
      ...
```

It also configures a cookie with a 30-day expiry.

This appears to be an earlier/static credential-based implementation
rather than the current PostgreSQL-backed approach.

## `authentication_db_old.py`

This version loads users directly from PostgreSQL and converts them into
the credential structure expected by `streamlit_authenticator`.

It also contains a registration form.

Compared with the current implementation, it stores more user
attributes:

```text
username
name
email
password
```

The current architecture appears to have moved authentication
responsibility toward `authentication_db.py` and `main.py`.

---

# 11. Alternate Chatbot Implementations

## `app.py`

This is an earlier chatbot implementation that:

* Uploads a PDF
* Extracts text
* Splits text into chunks
* Creates OpenAI embeddings
* Builds/loads FAISS
* Performs similarity search
* Uses `gpt-4o-mini`
* Returns the generated answer

It is functionally similar to the document-processing logic in
`admin.py`.

## `app_deepseek.py`

This implementation follows a similar RAG pipeline but uses:

```python
ChatDeepSeek
```

with:

```text
deepseek-chat
```

as the answer-generation model.

It retains OpenAI embeddings while changing the LLM used for the final
answer.

---

# Requirements

The project uses the following Python dependencies:

```text
streamlit
streamlit-authenticator
psycopg2-binary
Werkzeug
PyPDF2
langchain
langchain-community
langchain-openai
faiss-cpu
PyYAML
python-dotenv
langchain-deepseek
```

These dependencies are also provided in [`requirements.txt`](requirements.txt).

## Installing Dependencies

Create and activate a Python virtual environment, then run:

```bash
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Install the project dependencies:

```bash
pip install -r requirements.txt
```

The project uses `python-dotenv` to load the `OPENAI_API_KEY` and other configuration values from the `.env` file rather than hard-coding secrets in Python source files.

# 13. Running the Application

## Step 1 --- Start PostgreSQL

```bash
docker compose up -d
```

## Step 2 --- Configure secrets

Set environment variables for database credentials and the LLM provider
API key.

Do not commit secrets to GitHub.

Example:

```env
DB_PASSWORD=your-database-password
OPENAI_API_KEY=your-api-key
```

## Step 3 --- Start the main application

```bash
streamlit run main.py
```

The application should present the login page.

## Step 4 --- Register a user

The supplied implementation links to a separate registration application
on:

```text
http://localhost:8505
```

Run the registration application separately if retaining the current
design:

```bash
streamlit run registration_form.py --server.port 8505
```

## Step 5 --- Log in

Use a registered account.

The application resolves the role from PostgreSQL.

## Step 6 --- Admin workflow

An administrator can access the document-processing interface, where the
PDF is:

```text
Uploaded
→ Extracted
→ Chunked
→ Embedded
→ Indexed in FAISS
```

## Step 7 --- User workflow

A regular user accesses the existing FAISS index and asks questions
against the indexed knowledge base.

---

# 14. Security Considerations

The current source code should **not** be deployed as-is.

## API keys

API keys are present directly in source files.

They must be:

1. Revoked/rotated.
2. Removed from source code.
3. Stored in environment variables or a secrets manager.
4. Excluded from Git history.

Recommended:

```python
OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
```

## Database credentials

Database credentials are also present in the Python source.

Use environment variables instead.

## `password.yml`

The supplied YAML contains plaintext example passwords. It should not be
committed with real credentials.

Use hashed passwords and secret management.

## FAISS deserialization

The code uses:

```python
allow_dangerous_deserialization=True
```

This should only be used with trusted FAISS files. Do not load arbitrary
indexes supplied by untrusted users.

## Authentication architecture

The current application launches separate Streamlit processes through
`subprocess.run()`. This should be replaced by in-process role-based
routing for a production deployment.

## Input validation

Registration should validate:

* Empty username
* Empty password
* Password length
* Duplicate username
* Database errors

## Database resources

Database connections should ideally use context managers or connection
pooling so that connections are reliably closed when exceptions occur.

---

# 15. Recommended Production Improvements

### Authentication

* Centralize authentication in one module.
* Use one password hashing algorithm.
* Add account/session expiration.
* Add login attempt throttling.
* Add stronger password policy.
* Avoid exposing authentication details through error messages.

### Database

* Add explicit schema migrations.
* Add unique constraint on `users.username`.
* Add foreign-key constraints for `user_role`.
* Use a connection pool.
* Use environment-based configuration.

### RAG pipeline

* Associate every FAISS index with a document/version.
* Support multiple documents.
* Store document metadata.
* Rebuild indexes when source documents change.
* Consider a persistent vector database for larger deployments.
* Increase answer token limits when longer answers are required.
* Add source/citation information to generated answers.

### PDF processing

* Add OCR for scanned documents.
* Handle `extract_text()` returning `None`.
* Validate file size.
* Validate PDF content.
* Handle corrupted PDFs gracefully.

### Streamlit architecture

Instead of:

```text
main.py
  |
  +--> subprocess --> admin.py
  |
  +--> subprocess --> user.py
```

prefer:

```text
main.py
  |
  +--> authenticate
  |
  +--> role = admin --> render_admin()
  |
  +--> role = user  --> render_user()
```

This avoids spawning additional Streamlit servers and makes session
management significantly easier.

---

# 16. Troubleshooting

## PostgreSQL connection failure

Check:

```bash
docker compose ps
```

Verify PostgreSQL is listening on port `5432`.

Also verify:

```text
Database name
Username
Password
Host
Port
```

match the application's configuration.

## FAISS index not found

The user application expects an existing FAISS index.

Run the administrator document-ingestion flow first.

Verify the configured:

```text
VECTOR_STORE_PATH
```

points to the same directory used when the index was created.

## PDF produces no useful text

The PDF may be scanned or image-based.

Use OCR before passing it to `PyPDF2`.

## Authentication succeeds but role is missing

Verify the `user_role` record exists and points to a valid `roles.id`.

Expected relationship:

```text
users.id -> user_role.user_id
roles.id -> user_role.role_id
```

## LLM/API failure

Verify the API key is valid and available to the process through its
environment.

Do not place a new key directly into source code.

---

# 17. End-to-End Data Flow

```text
                    USER
                     |
                     v
              +-------------+
              |   main.py   |
              +------+------+
                     |
              username/password
                     |
                     v
          +----------------------+
          | authentication_db.py |
          +----------+-----------+
                     |
                     v
                PostgreSQL
                     |
              +------+------+
              |             |
          credentials      role
              |             |
              +------+------+
                     |
                     v
              Session State
                     |
              +------+------+
              |             |
            admin          user
              |             |
              v             v
          admin.py       user.py
              |             |
          PDF upload      Question
              |             |
          PDF extraction     |
              |             |
          Text chunking      |
              |             |
          Embeddings         |
              |             |
             FAISS <---------+
              |
       Similarity retrieval
              |
              v
         Relevant chunks
              |
              v
             LLM
              |
              v
           Answer
```

---

# 18. Current Project Status

The supplied code represents a functional prototype of a role-based RAG
chatbot with PostgreSQL authentication and FAISS-based document
retrieval.

The main architectural concepts are:

* Streamlit for the UI
* PostgreSQL for authentication data
* Role-based authorization
* PyPDF2 for PDF extraction
* LangChain for text processing and QA orchestration
* OpenAI embeddings for semantic representation
* FAISS for local vector similarity search
* OpenAI/DeepSeek-compatible LLM integrations for answer generation
* Docker Compose for PostgreSQL infrastructure

The project would benefit from consolidating the legacy
authentication/chatbot implementations, removing secrets from source
control, adding database schema/migration management, and replacing
subprocess-based Streamlit routing with a single-process role-aware
application.

---

## License

No license information was provided in the supplied project files. Add
an appropriate `LICENSE` file before publishing the repository if the
project is intended for public distribution.
