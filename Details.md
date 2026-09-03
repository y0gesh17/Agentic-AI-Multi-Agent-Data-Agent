# Agentic AI — Data Agent (Detailed Flow)

A multi-agent LangGraph + LangChain system that takes a natural-language request, classifies it as **SQL** or **ETL**, and runs a specialist subgraph. This document follows the real call path in this repo: which file starts, how state moves, and how tools/LLMs are wired.

YouTube walkthrough: https://youtu.be/7yOmi4IX-Rs

---

## Table of contents

1. [What this project does](#1-what-this-project-does)
2. [How a run starts (entry points)](#2-how-a-run-starts-entry-points)
3. [End-to-end flow](#3-end-to-end-flow)
4. [LangChain vs LangGraph in this codebase](#4-langchain-vs-langgraph-in-this-codebase)
5. [State schemas (Pydantic)](#5-state-schemas-pydantic)
6. [LLM selection](#6-llm-selection)
7. [Supervisor: Data Agent](#7-supervisor-data-agent)
8. [SQL Analyst agent](#8-sql-analyst-agent)
9. [ETL Analyst agent (tools + ReAct loop)](#9-etl-analyst-agent-tools--react-loop)
10. [Utilities](#10-utilities)
11. [Database bootstrap](#11-database-bootstrap)
12. [Project structure](#12-project-structure)
13. [Setup and run](#13-setup-and-run)
14. [Example invocations](#14-example-invocations)
15. [Import-time side effects and caveats](#15-import-time-side-effects-and-caveats)

---

## 1. What this project does

The user never talks to SQL or Pandas directly. They send English (or similar) to one compiled graph: `data_agent`.

That graph:

1. Reads the last user message.
2. Asks an LLM to classify the intent as `"sql"` or `"etl"` (structured output).
3. **Delegates** by invoking another compiled graph:
   - `sql_analyst` — NL → Postgres SELECT → safety judge → execute → NL answer.
   - `etl_analyst` — LLM + LangChain tools: extract from HTTP APIs, or transform files with generated Pandas code.

Sample ride-sharing data lives under `data/` (users, vehicles, rides, payments, ratings). `feed_db.py` loads those CSVs into PostgreSQL so the SQL agent has a schema to query.

---

## 2. How a run starts (entry points)

### Primary: `main.py`

This is the intended application entry.

```python
from agents.data_agent import data_agent
from langchain_core.messages import HumanMessage

if __name__ == "__main__":
    response = data_agent.invoke(
        {
            "messages": [HumanMessage(content="...")],
            "route_response": "",
        }
    )
    print(response)
```

What happens when you run `python main.py`:

1. Python imports `agents.data_agent`.
2. That import **builds and compiles** three graphs (data, and via imports, SQL + ETL), binds LLMs, and currently also **writes graph PNGs** (see caveats).
3. `data_agent.invoke(...)` runs the supervisor graph once with initial state matching `DataAgentSchema`.
4. The printed `response` is the **final graph state** (messages plus `route_response`), not a single string.

### Alternate entries (same graphs, no supervisor)

| Command | What it runs |
|---------|----------------|
| `python main.py` | Supervisor `data_agent` with a hardcoded ETL example (PokeAPI extract). |
| `python agents/data_agent.py` | Same supervisor graph; also writes `data_agent_graph.png`. |
| `python agents/sql_analyst.py` | SQL subgraph only (hardcoded payment-methods question); writes `sql_analyst_graph.png`. |
| `python agents/etl_analyst.py` | ETL subgraph only (PokeAPI extract); writes `etl_analyst_graph.png`. |
| `python feed_db.py` | Create tables + COPY CSVs into Postgres. Not part of the agent loop. |

There is no HTTP server, Streamlit app, or CLI parser. Invocation is Python `invoke` with a dict of state fields.

---

## 3. End-to-end flow

```
python main.py
        │
        ▼
┌───────────────────────────────────────┐
│  agents/data_agent.py                 │
│  compiled graph: data_agent           │
│  state: DataAgentSchema               │
│                                       │
│  START → router_node                  │
│            │                          │
│            ├─ route_response=="sql" → sql_node  → END
│            └─ route_response=="etl" → etl_node  → END
└───────────────────────────────────────┘
            │                    │
            ▼                    ▼
   sql_analyst.invoke()   etl_analyst.invoke()
   (nested graph)         (nested graph)
```

**Router** uses `llm.with_structured_output(RouterSchema)` so the model must return `{ "answer": "sql" | "etl", "comments": "..." }`. Only `answer` is stored on state as `route_response`.

**sql_node** does not pass the chat history as LangChain messages into the SQL graph. It copies the last message text into `user_question` and starts a fresh `AgentSchema`.

**etl_node** wraps the last message in a new `HumanMessage` and invokes `etl_analyst` with `{ "messages": [...] }` only.

After a sub-agent returns, the supervisor **appends the entire sub-agent result object** onto `state.messages` (not necessarily a single `AIMessage`). Downstream code that expects only LangChain message types should treat that as a quirk of this implementation.

---

## 4. LangChain vs LangGraph in this codebase

### LangChain (models, messages, tools, structured output)

Used as the **LLM SDK**, not as LangChain Agents (`AgentExecutor` is not used).

| API | Where | Role |
|-----|--------|------|
| `ChatOpenAI` / `ChatAnthropic` | `utils/llm_pick.py` | Chat models. |
| `load_dotenv()` | `llm_pick.py`, `feed_db.py` | Loads `.env` into `os.environ`. |
| `HumanMessage`, `AIMessage`, `ToolMessage` | agents | Standard chat / tool protocol. |
| `llm.with_structured_output(PydanticModel)` | router + SQL judge | Forces JSON-shaped answers (`RouterSchema`, `JudgeSchema`). |
| `@tool` from `langchain.tools` | `etl_analyst.py` | Turns Python functions into LLM-callable tools with schemas from type hints + docstrings. |
| `llm.bind_tools(tools)` | `etl_analyst.py` | Model may emit `tool_calls` instead of (or before) a final text reply. |
| `tool.invoke(args)` | `tool_node` | Executes the chosen tool with parsed arguments. |

**Structured output** is how routing and SQL safety stay machine-readable. The judge does not regex the SQL; it asks another LLM to fill `JudgeSchema.answer` as `"Yes"` or `"No"`.

**Tools** are only used on the ETL path. The SQL path is a **fixed pipeline of nodes**, not a tool-calling loop.

### LangGraph (graphs, nodes, edges, compile)

LangGraph stores **typed state** and runs **nodes** that read/write that state.

Patterns used here:

1. **`StateGraph(Schema)`** — schema is a Pydantic model. Fields are the graph state.
2. **`Annotated[list, add]` on `messages`** — list reducer: new lists are **concatenated**, not replaced. That is why nodes do `state.messages = state.messages + [...]`.
3. **`add_node(name, fn)`** — each node is a Python function `(state) -> state`.
4. **`add_edge(START, "first_node")`** — entry.
5. **`add_conditional_edges(source, router_fn, mapping)`** — `router_fn` returns a **string key**; mapping sends execution to a node or `END`.
6. **`compile()`** — produces a `CompiledStateGraph` with `.invoke(input_dict)` and `.get_graph().draw_mermaid_png()`.
7. **Nested graphs** — supervisor nodes call `other_graph.invoke(...)`. That is subgraph-as-function, not `add_node(compiled_graph)` as a native subgraph.

There is **no checkpointer**, no `interrupt`, no human-in-the-loop, and no streaming API in this repo. Each `invoke` is one-shot.

---

## 5. State schemas (Pydantic)

File: `Models/schema.py`

### `DataAgentSchema` (supervisor)

- `messages` — conversation / results (reducer `add`).
- `route_response` — `"sql"` or `"etl"` after the router.

### `RouterSchema` (not graph state; structured LLM output)

- `answer`: `"sql"` | `"etl"`
- `comments`: why

### `AgentSchema` (SQL subgraph)

| Field | Meaning |
|-------|---------|
| `messages` | Curated question + final answer messages |
| `user_question` | Raw NL from supervisor |
| `curated_ques` | LLM-rewritten question |
| `prompt_query_context` | Full prompt including live DB schema text |
| `generated_sql_query` | Model’s SQL string |
| `is_safe` | `"Yes"` / `"No"` |
| `comments` | Judge rationale |
| `sql_query_execution_result` | `str(cursor.fetchall())` or similar |
| `final_answer` | User-facing prose |

### `JudgeSchema`

Same shape as router: `answer` Yes/No + `comments`. Used only inside `is_safe_sql`.

### `ETLAgentSchema`

- `messages` only. Tool calls live on the last `AIMessage.tool_calls`. Tool results are appended as `ToolMessage`s. The LLM is invoked again with the full history (ReAct).

---

## 6. LLM selection

File: `utils/llm_pick.py`

`pick_llm(level)` returns a chat model. Temperature is `0` for OpenAI paths.

| `level` | Model (as in code) | Used for |
|---------|--------------------|----------|
| `"low"` | `gpt-5.6-luna` | SQL: curate question, final NL answer |
| `"medium"` | `gpt-5.6-terra` | SQL: generate SQL, safety judge |
| `"high"` | `gpt-5.6-sol` | Defined, unused by agents currently |
| `"claude"` | `claude-sonnet-5` (`ChatAnthropic`) | Data-agent router; ETL LLM + tool binding; Pandas codegen inside `transform_load_tool` |

API keys come from the environment (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) via dotenv when this module is imported.

---

## 7. Supervisor: Data Agent

File: `agents/data_agent.py`

Compiled object: `data_agent`.

### Graph

```
START → router_node → (conditional route_edge)
                         ├─ "sql" → sql_node → (no outgoing edge → finish)
                         └─ "etl" → etl_node → (no outgoing edge → finish)
```

`route_edge` raises `ValueError` if `route_response` is anything other than `sql` or `etl`.

### Nodes

**`router_node`**

- Takes `state.messages[-1].content`.
- `llm_router.invoke(message).model_dump()` → `answer`.
- Sets `state.route_response`.
- Does not add a routing explanation to `messages`.

**`sql_node`**

- Builds a complete `AgentSchema` dict with empty strings and `is_safe: "No"` (overwritten later).
- `sql_analyst.invoke(input_schema)`.
- Appends the SQL graph’s full return value to `messages`.

**`etl_node`**

- `etl_analyst.invoke({ "messages": [HumanMessage(content=message)] })`.
- Appends that return value to `messages`.

Unused imports in this file (`ETLTools`, duplicate `sql_analyst` import, `ChatAnthropic`, `tool`) are leftover; they do not change runtime behavior.

---

## 8. SQL Analyst agent

File: `agents/sql_analyst.py`

Compiled object: `sql_analyst`.

This is a **linear DAG with one safety branch**, not a tool-calling agent.

```
START
  → curate_ques
  → prompt_query_context
  → generate_sql
  → is_safe_sql
       ├─ is_safe == "yes" → execute_sql → represent_final_answer → END
       └─ otherwise        → canceled_sql → END
```

### Node-by-node

1. **`curate_ques`**  
   `pick_llm("low")` rewrites `user_question`. Stores `curated_ques`. Appends a `HumanMessage` with that rewrite.

2. **`prompt_query_context`**  
   Reads Postgres from env: `host`, `port`, `user`, `password`, `database`.  
   `DatabaseUtil.schema_details("public")` lists tables, columns, types, and 5 sample rows.  
   Builds a long instruction: “output SQL only, default LIMIT 10, Postgres.”  
   Stores that string; **does not call an LLM here**.

3. **`generate_sql`**  
   `pick_llm("medium")` on that prompt. Raw `.content` is treated as executable SQL (no parser that strips markdown fences).

4. **`is_safe_sql`**  
   `with_structured_output(JudgeSchema)`. Prompt forbids INSERT/UPDATE/DELETE/DROP/ALTER/TRUNCATE/CREATE. Sets `is_safe` and `comments`.

5. **`execute_sql`**  
   `DatabaseUtil.execute_sql(query)` → `fetchall()`, `commit()`, stringify. On error returns `None`.

6. **`represent_final_answer`**  
   `pick_llm("low")` turns result + curated question into user-facing text. Appends `AIMessage`.

7. **`canceled_sql`**  
   No DB call. `final_answer` explains the judge comments. Appends `AIMessage`.

Safety is **LLM-based**, not a SQL parser. A jailbroken or mistaken judge can still allow destructive SQL. `execute_sql` also `commit()`s, which is unnecessary for pure SELECT.

---

## 9. ETL Analyst agent (tools + ReAct loop)

File: `agents/etl_analyst.py`

Compiled object: `etl_analyst`.

This is the only agent that uses **LangChain tools**.

### Tools (`@tool`)

Both wrap `utils.etl_tools.ETLTools`.

**`extract_load_tool(url, output_folder, format)`**

- HTTP GET JSON.
- `pd.json_normalize(data['results'])` — assumes a PokeAPI-style `{ "results": [...] }` payload.
- Writes `extracted_data.{format}` under project-root-joined `output_folder`.
- Formats: `csv`, `json` (records, lines), `parquet`.

**`transform_load_tool(input_file_path, output_folder, output_format, user_question)`**

- Reads first 3 rows via `transform_load_context` (csv/json/parquet).
- `pick_llm("claude")` is asked for **Pandas-only code** (no explanation).
- Strips markdown fences / leading `python`.
- `ETLTools.execute_code` runs `exec(code)` in-process (full Python, not a sandbox).
- Returns a status string including the code and exec result.

The toolkit list is `tools = [extract_load_tool, transform_load_tool]`. The bound model is `llm_bind = pick_llm("claude").bind_tools(tools)`.

### Graph (ReAct)

```
START → llm_node → is_tool_call?
                      ├─ tool_calls present → tool_node → llm_node (loop)
                      └─ no tool_calls      → END
```

**`llm_node`**

- Builds a prompt that includes `{messages}` (chat history).
- `llm_bind.invoke(prompt)` — if the model wants work done, the AIMessage has `tool_calls`.
- Appends that AIMessage to state.

**`tool_node`**

- Reads `state.messages[-1].tool_calls`.
- Looks up `tool.name`, `tool.invoke(tool_call['args'])`.
- Appends one `ToolMessage` per call (`tool_call_id` preserved so the next LLM turn can bind results).

**`is_tool_call`**

- If the last message has `tool_calls`, go to `tool_node`; else `END`.

Typical extract run:

1. User: extract PokeAPI → save CSV in `data/extract`.
2. LLM emits `extract_load_tool` with url/folder/format.
3. Tool writes the file, returns success text.
4. LLM sees `ToolMessage`, replies in natural language, **no new tool_calls** → graph ends.

Typical transform run:

1. LLM emits `transform_load_tool`.
2. Inside the tool, a **second** Claude call generates Pandas; `exec` runs it.
3. Outer loop continues until the analyst LLM stops calling tools.

---

## 10. Utilities

### `utils/etl_tools.py` — `ETLTools`

| Method | Behavior |
|--------|----------|
| `extract_load` | `requests.get`, normalize `results`, write file. Resolves `output_folder` relative to **project root** (parent of `utils/`). |
| `transform_load_context` | Load file, return `str(df.head(3))`. |
| `execute_code` | `exec(code)` in the current interpreter; returns success/error string. |

### `utils/database.py` — `DatabaseUtil`

| Method | Behavior |
|--------|----------|
| `__init__` | `psycopg2.connect(**db_config)`. |
| `schema_details(schema_name)` | `information_schema` tables/columns + `SELECT * ... LIMIT 5`. Closes connection in `finally`. |
| `execute_sql` | `cursor.execute`, `fetchall`, `commit`, close connection. |

**Important:** this module has **top-level** code that connects with hardcoded credentials, dumps schema to `test_schema_details.txt`, and runs on **every import**. Importing `sql_analyst` (which imports `DatabaseUtil`) can fail or write files even if you only wanted ETL.

Env keys used by agents: `host`, `port`, `user`, `password`, `database` (see `.env.example`).

---

## 11. Database bootstrap

File: `feed_db.py` (run separately, before SQL questions).

1. `load_dotenv()`.
2. Connect with the same env vars (`database` key in `psycopg2` config).
3. `CREATE TABLE IF NOT EXISTS` for `users`, `vehicles`, `rides`, `payments`, `ratings` plus indexes and FKs.
4. `TRUNCATE ... CASCADE` then `COPY` from `data/*.csv` in FK order: users → vehicles → rides → payments → ratings.
5. Print row counts, commit, close.

This is classic ETL **outside** the LangGraph ETL agent. The agent does not call `feed_db.py`.

Ride-share domain (for SQL examples):

- `users` — riders/drivers
- `vehicles` — owned by `driver_id` → users
- `rides` — rider, driver, geo, fare, status
- `payments` — per ride
- `ratings` — 1–5 with comments

---

## 12. Project structure

```
AI_Data_Agent-main/
├── main.py                 # App entry: data_agent.invoke(...)
├── feed_db.py              # Postgres schema + CSV load
├── Models/schema.py        # All Pydantic graph / structured-output models
├── agents/
│   ├── data_agent.py       # Supervisor graph (router + dispatch)
│   ├── sql_analyst.py      # SQL pipeline graph
│   └── etl_analyst.py      # Tool-calling ETL graph
├── utils/
│   ├── llm_pick.py         # pick_llm("low"|"medium"|"high"|"claude")
│   ├── database.py         # Postgres schema + execute
│   └── etl_tools.py        # HTTP extract, file peek, exec Pandas
├── data/
│   ├── users.csv, vehicles.csv, rides.csv, payments.csv, ratings.csv
│   └── extract/            # Default extract_load output dir
├── .env.example
├── pyproject.toml          # package data-agent, Python >= 3.12
└── requirements.txt
```

Python **3.12+**. Core deps: `langchain`, `langgraph`, `langchain-openai`, `langchain-anthropic`, `pandas`, `psycopg2-binary`, `pydantic`, `pyarrow`, `dotenv`.

---

## 13. Setup and run

1. Create a venv, install: `uv pip install -r requirements.txt` or `pip install -e .`
2. Copy `.env.example` to `.env` and set:
   - `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`
   - `host`, `port`, `user`, `password`, `database`
3. Start PostgreSQL; create the database named in `.env`.
4. `python feed_db.py` to load sample data.
5. `python main.py` (or change the `HumanMessage` in `main.py` for SQL vs ETL).

---

## 14. Example invocations

Supervisor (same as `main.py`):

```python
from agents.data_agent import data_agent
from langchain_core.messages import HumanMessage

out = data_agent.invoke({
    "messages": [HumanMessage(content="What payment methods exist in the database?")],
    "route_response": "",
})
```

Expected path: router → `sql` → `sql_node` → curate → schema prompt → SQL → judge → execute → NL answer.

ETL extract:

```text
Extract https://pokeapi.co/api/v2/pokemon into data/extract as csv
```

ETL transform (after a file exists):

```text
Transform data/extract/extracted_data.csv, keep only bulbasaur, save CSV under data/transform
```

---

## 15. Import-time side effects and caveats

1. **Graph PNG generation** at import: `data_agent.py` (and the `__main__` blocks of the other agents) call IPython `Image(...draw_mermaid_png())` and write `*_graph.png`. Importing the supervisor from `main.py` can require IPython and network for mermaid rendering.

2. **`utils/database.py` module-level connection** runs on import of `DatabaseUtil`. Fixing that (guard with `if __name__ == "__main__"`) is recommended so ETL-only runs do not need Postgres.

3. **SQL generation** may return markdown-wrapped SQL; there is no strip step like the ETL Pandas cleaner.

4. **`exec` in `execute_code`** is not sandboxed. The transform tool can run arbitrary Python the model emits.

5. **Extract path** assumes JSON with a `results` key.

6. **Router** has no “chitchat” class; only `sql` or `etl`. Ambiguous questions still get forced into one bucket.

7. **LLM names** in `llm_pick.py` must match whatever your OpenAI/Anthropic accounts actually expose.

---

## How the three graphs relate (mental model)

- **LangGraph** = control flow (which function runs next, branching, loops).
- **LangChain chat models** = intelligence inside nodes.
- **Structured output** = typed decisions (route, safe/unsafe).
- **Tools + bind_tools + ToolMessage** = ETL ReAct loop only.
- **Nested `invoke`** = supervisor delegates to specialists without merging their state schemas into one giant graph.

That is the full runtime story: `main.py` → compiled `data_agent` → router LLM → either the SQL node pipeline or the ETL tool loop → printed final state.
