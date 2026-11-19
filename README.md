# UdaPlay: AI Research Agent for Video Games

UdaPlay answers questions about video games. It checks its **own small database** first and **searches the web** only when that isn't enough. Every answer says **where it came from**.

---

## The idea in one picture

```
 Your question
      │
      ▼
 ① retrieve_game ──── search our 15-game database (ChromaDB)
      │
      ▼
 ② evaluate_retrieval ── an LLM "judge" asks: do these results answer the question?
      │
      ├── YES ──────────────────────────────┐
      │                                     ▼
      └── NO ─► ③ game_web_search (OpenAI web search) ─► ④ Final answer + Source + Confidence
```

Think of it like a junior analyst. First you check the company spreadsheet. If the answer isn't there, you Google it, and you write down the link.

---

## Folder layout

```
project/starter/
├── games/                           15 JSON files, one per game (our "internal data")
├── lib/                             helper code from the course (LLM, Agent, StateMachine, tools...)
├── Udaplay_01_starter_project.ipynb Part 1: build the database
├── Udaplay_02_starter_project.ipynb Part 2: build and run the agent
├── requirements.txt
└── .env.example                     template for your API keys
```

---

## Step 0: Set up your keys

1. In `project/starter/`, copy `.env.example` to a file named exactly **`.env`** and fill it in:

```
OPENAI_API_KEY="voc-..."                 # from the Udacity Cloud Resources panel
OPENAI_BASE_URL="https://openai.vocareum.com/v1"
TAVILY_API_KEY="tvly-..."               # optional: not used, see below
```

**Web search uses OpenAI's built-in web search tool with the same `voc-` key**, so you don't need a Tavily key. The project rubric accepts any web search API ("e.g., Tavily").

`.env` is in `.gitignore`, so it is never committed.

---

## Step 1: Part 1 notebook (build the database)

Open `Udaplay_01_starter_project.ipynb` **from the `project/starter` folder** and run all cells.

| Cell | What it does, in plain words |
|---|---|
| Load `.env` | Reads your API keys and checks all 3 exist |
| `PersistentClient(path="chromadb")` | Creates a database saved on disk in `chromadb/` |
| Embedding function | Turns text into numbers so the computer can compare *meaning*, not just exact words |
| Create collection `udaplay` | A "table" inside the database |
| Add documents | Each game file becomes one text: `[Platform] Name (Year) - Genre, Publisher, Description` |
| Test search | Asks e.g. *"Who published GTA: San Andreas?"* and shows that the right game comes back first |

> **Distance** = how far the result is from the question. **Smaller is better.**

Don't delete the `chromadb/` folder. Part 2 needs it.

---

## Step 2: Part 2 notebook (the agent)

Open `Udaplay_02_starter_project.ipynb` (same folder) and run all cells.

### The 3 tools

| Tool | Job | How |
|---|---|---|
| `retrieve_game(query)` | Find the 3 most similar games | `collection.query(...)` on the Part 1 database |
| `evaluate_retrieval(question, retrieved_docs)` | Decide if those games answer the question | A second LLM returns an `EvaluationReport`: `useful` (True/False), `confidence` (high/medium/low), `description` (why) |
| `game_web_search(question)` | Fallback when the judge says *not useful* | OpenAI Responses API with the `web_search_preview` tool; returns an answer plus the source **URLs** |

Each tool prints a line like `[STEP 1 - retrieve_game] ...`, so you can **watch the agent think** in the output.

### The agent (state machine)

The `Agent` class in `lib/agents.py` runs as a small **state machine**, a fixed set of steps with rules for moving between them:

```
entry → message_prep → llm_processor ──(wants a tool?)── yes → tool_executor ─┐
                            ▲                                                 │
                            └─────────────────────────────────────────────────┘
                                         no → termination (final answer)
```

The system instructions tell the LLM to always run **retrieve → evaluate → web search only if needed**, then answer in this format:

```
**Answer:** ...
**Source:** Internal database: <game>   OR   Web: <URL>
**Confidence:** high / medium / low
```

### Memory (conversation state)

Each question you ask with the same `session_id` is added to one shared message history. So you can ask:

1. *"Who published Grand Theft Auto: San Andreas?"*
2. *"What platform was it released on?"* → the agent knows "it" = GTA: San Andreas.

A new `session_id` starts with an empty history.

### Example queries in the notebook

| # | Question | Expected path |
|---|---|---|
| 1 | When was Pokémon Gold and Silver released? | Database ✅ (1999, Game Boy Color) |
| 2 | Which one was the first 3D platformer Mario game? | Database ✅ (Super Mario 64, 1996) |
| 3 | Was Mortal Kombat X released for PlayStation 5? | Not in database → **web search** + URL |
| 4 | Who published Grand Theft Auto: San Andreas? | Database ✅ (Rockstar Games) |
| 5 | What platform was it released on, and in which year? | Memory follow-up → PS2, 2004 |
| 6 | What is Rockstar Games working on right now? | Current news → **web search** + URL |
| - | Count saved questions per session | `demo` = 6, a new session = 0 |

LLM wording and web results change between runs. That's expected and fine.

---

## Before you submit (checklist)

- [ ] Run **every cell** in Part 1, then Part 2, and **save** both notebooks with outputs
- [ ] Check at least one answer came from the database and one from the web (with a URL)
- [ ] Keep the `lib/` folder
- [ ] **Delete `.env`** (or blank out the values) after your final run
