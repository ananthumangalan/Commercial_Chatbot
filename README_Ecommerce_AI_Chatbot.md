# E-Commerce AI Chatbot

## Project Overview

This project is a **Phase 1 Proof of Concept (POC) for an AI-powered e-commerce chatbot**.

The chatbot handles two major types of customer queries:

1. **FAQ queries**
   - What is your refund policy?
   - What payment methods are accepted?
   - How can I track my order?

2. **Product queries**
   - Show me Nike shoes below Rs. 3000.
   - Show me top 2 Nike shoes with rating higher than 4.5.
   - Are there any Puma shoes on sale?

The application uses a **semantic router** to understand the user's intent and send the query to the correct pipeline.

---

## High-Level Architecture

```text
                         USER
                          |
                          v
                  Streamlit Chat UI
                       main.py
                          |
                          v
                  Semantic Router
              FastEmbedEncoder + SemanticRouter
                    /             \
                   /               \
                FAQ                 SQL
                 |                   |
                 v                   v
              faq.py               sql.py
                 |                   |
             ChromaDB             SQLite
                 |                   |
          Relevant FAQ          Product Data
                 |                   |
                 v                   v
             Groq LLM            Groq LLM
                   \              /
                    \            /
                     FINAL RESPONSE
```

---

## Tech Stack

- **Python**
- **Streamlit** - chatbot user interface
- **Semantic Router** - intent classification
- **FastEmbedEncoder** - generates embeddings for routing
- **ChromaDB** - vector database for FAQ retrieval
- **SQLite** - product database
- **Pandas** - handling SQL results and CSV data
- **Groq API** - access to the LLM
- **Current Groq Model** - `openai/gpt-oss-120b`
- **python-dotenv** - loading environment variables

---

## Project Structure

```text
project/
|
|-- app/
|   |-- main.py
|   |-- router.py
|   |-- faq.py
|   |-- sql.py
|   |-- db.sqlite
|   |-- .env
|   |
|   `-- resources/
|       |-- faq_data.csv
|       `-- ecommerce_data_final.csv
|
|-- README.md
`-- requirements.txt
```

### Important Files

- `main.py` - Streamlit frontend and project controller
- `router.py` - classifies the user query into FAQ or SQL intent
- `faq.py` - FAQ ingestion, retrieval, RAG, and LLM answer generation
- `sql.py` - natural-language-to-SQL pipeline and response generation
- `db.sqlite` - product database
- `faq_data.csv` - FAQ question-answer data
- `ecommerce_data_final.csv` - product dataset/resource
- `.env` - stores Groq configuration

---

# How the Project Works

## 1. User Enters a Query

The user interacts with the application through the Streamlit chat interface.

Example:

```text
What is your refund policy?
```

or:

```text
Show me Nike shoes below Rs. 3000
```

The query is passed to the `ask()` function in `main.py`.

---

## 2. Semantic Routing

The query is sent to the semantic router in `router.py`.

The router uses:

```text
FastEmbedEncoder
        +
SemanticRouter
```

The encoder converts the user query into an embedding.

The semantic router compares the meaning of the query with example utterances defined for two routes:

```text
faq
sql
```

### FAQ Route Examples

- Return policy
- Refund questions
- Payment methods
- Order tracking
- Card discounts

### SQL Route Examples

- Product price
- Brand search
- Discount search
- Product rating
- Product availability

The router returns either `faq` or `sql`.

---

# FAQ Pipeline

The FAQ pipeline is implemented in `faq.py`.

Its main purpose is to answer policy and general e-commerce questions using a **RAG pipeline**.

## FAQ Flow

```text
faq_data.csv
      |
      v
ingest_faq_data()
      |
      v
   ChromaDB
      |
      v
User FAQ Question
      |
      v
get_relevant_qa()
      |
      v
Top Relevant FAQ Answers
      |
      v
Build Context
      |
      v
generate_answer()
      |
      v
Groq LLM
      |
      v
Final Answer
```

## Step 1: FAQ Data Ingestion

The function:

```python
ingest_faq_data(path)
```

performs these steps:

1. Checks whether the FAQ collection already exists in ChromaDB.
2. Reads `faq_data.csv` using Pandas.
3. Stores FAQ questions as documents.
4. Stores the corresponding answers as metadata.
5. Creates unique IDs.
6. Adds the data to the ChromaDB collection.

Conceptually:

```text
Question -> searchable document
Answer   -> metadata
```

## Step 2: FAQ Retrieval

The function:

```python
get_relevant_qa(query)
```

searches the ChromaDB collection using semantic similarity and retrieves the top relevant FAQ records.

Example:

```text
User:
"When will I get my money back?"

            ↓

Semantic similarity search

            ↓

Relevant FAQ:
"How long does it take to process a refund?"
```

## Step 3: Build Context

The answers retrieved from ChromaDB are combined into a context string.

That context is passed to the LLM together with the original user question.

## Step 4: Generate the FAQ Answer

The function:

```python
generate_answer(query, context)
```

sends the question and retrieved context to the Groq LLM.

The prompt tells the model to:

- answer only using the provided context
- avoid making up unsupported answers
- say `"I don't know"` if the answer is not present

## Step 5: `faq_chain()`

The function:

```python
faq_chain(query)
```

connects the complete FAQ workflow:

```text
get_relevant_qa()
        ↓
build context
        ↓
generate_answer()
        ↓
return final answer
```

### FAQ Pipeline Summary

```text
ingest_faq_data()
→ stores FAQ data

get_relevant_qa()
→ retrieves relevant FAQ data

generate_answer()
→ generates answer using retrieved context

faq_chain()
→ connects retrieval and generation
```

---

# SQL / Product Pipeline

The product-search pipeline is implemented in `sql.py`.

Its job is to convert a user's natural-language product question into SQL, execute the SQL on the SQLite database, and return a readable response.

## SQL Flow

```text
User Product Question
        |
        v
generate_sql_query()
        |
        v
Groq LLM
        |
        v
Generated SQL
        |
        v
run_query()
        |
        v
SQLite Database
        |
        v
Product Rows
        |
        v
data_comprehension()
        |
        v
Groq LLM
        |
        v
Final Readable Answer
```

## SQL Prompt

The `sql_prompt` tells the LLM how the product database is structured.

The `product` table contains fields such as:

- `product_link`
- `title`
- `brand`
- `price`
- `discount`
- `avg_rating`
- `total_ratings`

Its purpose is:

```text
Natural-language question
        ↓
SQL query
```

## `generate_sql_query()`

The function:

```python
generate_sql_query(question)
```

sends:

1. the database schema and SQL instructions as the system prompt
2. the actual user question as the user message

The LLM generates SQL and returns it inside `<SQL> ... </SQL>` tags.

## `run_query()`

The function:

```python
run_query(query)
```

performs these steps:

1. Checks that the query begins with `SELECT`.
2. Connects to `db.sqlite`.
3. Executes the SQL query.
4. Loads the result into a Pandas DataFrame.
5. Returns the DataFrame.

## Comprehension Prompt

The `comprehension_prompt` is used **after** the database query has been executed.

```text
SQL Prompt
User Question -> SQL

Comprehension Prompt
Database Result -> Natural-Language Answer
```

The comprehension prompt tells the LLM to format product information using details such as:

- product title
- price
- discount
- rating
- product link

## `data_comprehension()`

The function:

```python
data_comprehension(question, context)
```

receives the original user question and the database result, then asks the LLM to convert the raw data into a readable customer response.

## `sql_chain()`

The function:

```python
sql_chain(question)
```

connects the full SQL pipeline:

```text
User Question
      ↓
generate_sql_query()
      ↓
Extract SQL from <SQL> tags
      ↓
run_query()
      ↓
SQLite
      ↓
Convert result to records
      ↓
data_comprehension()
      ↓
Return final answer
```

### SQL Pipeline Summary

```text
generate_sql_query()
→ natural language to SQL

run_query()
→ SQL to database result

data_comprehension()
→ database result to readable answer

sql_chain()
→ connects the complete SQL workflow
```

---

# Main Application

The Streamlit application is controlled by `main.py`.

## Responsibilities of `main.py`

```text
Prepare FAQ data
      ↓
Create Streamlit UI
      ↓
Receive user query
      ↓
Send query to semantic router
      ↓
FAQ or SQL
      ↓
Call appropriate chain
      ↓
Receive response
      ↓
Display response
      ↓
Store chat history
```

The central controller function is:

```python
def ask(query):
    route = router(query).name

    if route == "faq":
        return faq_chain(query)

    elif route == "sql":
        return sql_chain(query)
```

This function acts as the bridge between the semantic router and the two chatbot pipelines.

---

# Streamlit Chat History

Streamlit's:

```python
st.session_state
```

is used to store user and assistant messages during the current session.

Each chat turn follows this pattern:

```text
Receive user message
      ↓
Display user message
      ↓
Store user message
      ↓
Process query
      ↓
Display assistant response
      ↓
Store assistant response
```

The current implementation stores chat history for the user interface. Previous messages are not currently passed to the LLM as conversational memory.

---

# Environment Configuration

Create a `.env` file inside the `app` folder:

```env
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=openai/gpt-oss-120b
```

**Important:** Never commit your real API key to GitHub.

A recommended `.gitignore` entry is:

```text
.env
```

---

# Running the Project

## 1. Install Dependencies

```bash
pip install -r requirements.txt
```

The current router uses FastEmbed, so ensure the semantic router and FastEmbed dependencies are installed:

```bash
python -m pip install semantic-router
python -m pip install fastembed
```

## 2. Move to the App Folder

```bash
cd app
```

## 3. Run Streamlit

```bash
streamlit run main.py
```

The app should open in the browser at a local URL such as:

```text
http://localhost:8501
```

---

# Example Queries

## FAQ Queries

```text
What is your refund policy?
```

```text
How long does it take to process a refund?
```

```text
What payment methods are accepted?
```

## Product Queries

```text
Show me Nike shoes below Rs. 3000
```

```text
Show me top 2 Nike shoes with rating higher than 4.5
```

```text
Are there any Puma shoes on sale?
```

---

# Key Concepts Used in the Project

## Semantic Routing

```text
User Query
    ↓
FastEmbedEncoder
    ↓
Embedding
    ↓
SemanticRouter
    ↓
FAQ / SQL
```

## RAG

```text
Retrieval
→ retrieve FAQ knowledge from ChromaDB

Augmentation
→ add retrieved answers to the prompt

Generation
→ LLM generates an answer using that context
```

## Text-to-SQL

```text
Natural Language
      ↓
LLM
      ↓
SQL
      ↓
SQLite
```

## LLM Grounding

```text
FAQ Knowledge
→ ChromaDB

Product Knowledge
→ SQLite

LLM
→ understands, generates SQL, and communicates results
```

---

# Final End-to-End Flow

```text
                            USER
                             |
                             v
                       Streamlit UI
                             |
                             v
                 FastEmbed Semantic Router
                      /              \
                     /                \
                   FAQ                 SQL
                    |                   |
                    v                   v
               FAQ RAG             Text-to-SQL
                    |                   |
                ChromaDB              LLM
                    |                   |
             Retrieved FAQs           SQL
                    |                   |
                    v                 SQLite
                   LLM                  |
                    |              Product Data
                    |                   |
                    |                  LLM
                    |                   |
                     \                 /
                      \               /
                        FINAL RESPONSE
```

---

# What I Learned From This Project

This project helped me understand how to combine multiple AI and backend components into one chatbot application:

- building a Streamlit chatbot interface
- using embeddings for semantic understanding
- performing semantic intent routing with FastEmbed
- storing and retrieving FAQ knowledge using ChromaDB
- implementing a RAG pipeline
- designing prompts for grounded LLM responses
- generating SQL from natural-language questions
- executing queries against SQLite
- converting raw database results into natural-language responses
- working with Groq-hosted LLMs
- managing configuration using `.env`
- debugging a multi-component GenAI application step by step

---

# Project Status

The current Phase 1 chatbot successfully supports:

- FAQ semantic routing
- FAQ retrieval and RAG-based answering
- product-query semantic routing
- natural-language-to-SQL generation
- SQLite product search
- LLM-based product response formatting
- Streamlit chat interface
- session-based display of chat history

This architecture can be adapted to other domains such as real estate, banking, healthcare, HR, travel, and customer support by changing the routes, knowledge sources, prompts, and databases.
