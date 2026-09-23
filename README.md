<div align="center">

# ✈️ TripMate AI

### A Multi-Agent Travel Planner built with LangGraph, MCP & FastAPI

*Describe your dream trip in plain English. A team of AI agents researches flights and hotels, drafts an itinerary, waits for **your approval**, and delivers a polished travel plan.*

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1.2-1C3C3C?logo=langchain&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.136-009688?logo=fastapi&logoColor=white)
![Groq](https://img.shields.io/badge/LLM-Groq%20gpt--oss--120b-F55036)
![MCP](https://img.shields.io/badge/Tools-MCP-6E56CF)
![PostgreSQL](https://img.shields.io/badge/Checkpoints-PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)

</div>

---

## 📖 Table of Contents

- [Features](#-features)
- [Agent Graph](#-agent-graph)
- [How a Request Flows](#-how-a-request-flows)
- [The Agents](#-the-agents)
- [System Architecture](#-system-architecture)
- [Using the App](#-using-the-app)
- [API Reference](#-api-reference)
- [Getting Started](#-getting-started)
- [Project Structure](#-project-structure)
- [Tech Stack](#-tech-stack)

---

## 🌟 Features

| | Feature | What it does |
|---|---|---|
| 🧠 | **Supervisor agent** | Reads your request, extracts trip constraints (origin, destination, duration, budget, style) and picks only the specialist agents it needs |
| 🛡️ | **Input guardrail** | Blocks requests that aren't about travel, or that ask for harmful content, before any tool runs |
| 🛫 | **Flight agent** | Pulls real airport and airline data through the **AviationStack MCP server** |
| 🏨 | **Hotel agent** | Runs a live web search for accommodation through the **Tavily MCP server** |
| 🗺️ | **Itinerary agent** | Combines everything into a practical, budget-aware day-by-day draft |
| 🙋 | **Human-in-the-loop** | The graph **pauses** so you can approve the draft or ask for changes |
| 💾 | **Persistent memory** | Every run is checkpointed to **PostgreSQL**, so a paused plan can be resumed later using its `thread_id` |
| 📄 | **Export** | Copy the final plan or download it as a PDF |

---

## 🕸️ Agent Graph

This is the LangGraph `StateGraph` defined in [`backend.py`](backend.py). Solid arrows are fixed edges and dotted arrows are conditional routes.

```mermaid
flowchart TD
    START([▶ START]) --> SUP

    SUP{{"🧠 Supervisor<br/><i>guardrail + routing</i>"}}

    SUP -. "❌ not travel-related" .-> BLOCK["🛡️ Guardrail Blocked"]
    SUP -. "flights needed" .-> FLIGHT["🛫 Flight Agent"]
    SUP -. "hotels only" .-> HOTEL["🏨 Hotel Agent"]
    SUP -. "no specialists needed" .-> ITIN

    FLIGHT -. "hotels selected" .-> HOTEL
    FLIGHT -. "otherwise" .-> ITIN
    HOTEL --> ITIN

    ITIN["🗺️ Itinerary Agent<br/><i>draft plan</i>"] --> HUMAN
    HUMAN[/"🙋 Human Approval<br/><b>interrupt()</b>"/] --> FINAL
    FINAL["✨ Final Agent<br/><i>polish or revise</i>"] --> END1([⏹ END])
    BLOCK --> END2([⏹ END])

    AV[(AviationStack MCP)] -.- FLIGHT
    TV[(Tavily MCP)] -.- HOTEL

    classDef sup fill:#6E56CF,stroke:#4c3a9e,color:#fff
    classDef agent fill:#0ea5e9,stroke:#0369a1,color:#fff
    classDef human fill:#f59e0b,stroke:#b45309,color:#fff
    classDef block fill:#ef4444,stroke:#991b1b,color:#fff
    classDef final fill:#10b981,stroke:#047857,color:#fff
    classDef tool fill:#1f2937,stroke:#111,color:#fff

    class SUP sup
    class FLIGHT,HOTEL,ITIN agent
    class HUMAN human
    class BLOCK block
    class FINAL final
    class AV,TV tool
```

**Routing rules**

- Specialist agents always run in a fixed order: `flight_agent` → `hotel_agent` → `itinerary_agent`. Any agent the supervisor didn't select is skipped.
- `itinerary_agent` **always** runs, because it combines whatever results the other agents produced.
- If the supervisor's JSON can't be parsed, the graph **falls back to the full workflow**. If the guardrail's JSON can't be parsed, the request is **allowed through**, so a formatting slip never breaks a real trip request.

---

## 🔄 How a Request Flows

```mermaid
sequenceDiagram
    autonumber
    actor U as 👤 User
    participant UI as 🌐 Web UI
    participant API as ⚡ FastAPI
    participant G as 🕸️ LangGraph
    participant DB as 💾 PostgreSQL
    participant MCP as 🔌 MCP Tools

    U->>UI: "Plan a 7-day Japan trip under 20000 SAR"
    UI->>API: POST /api/travel
    API->>G: invoke(user_query, thread_id)
    G->>G: Supervisor: guardrail + choose agents
    G->>MCP: AviationStack (airports, airlines)
    G->>MCP: Tavily search (hotels)
    G->>G: Itinerary agent drafts the plan
    G->>DB: checkpoint state
    G-->>API: ⏸ interrupt (draft itinerary)
    API-->>UI: requires_approval = true
    UI-->>U: Shows draft + Approve / Revise

    alt ✅ Approved
        U->>UI: Approve
    else ✏️ Revise
        U->>UI: "Cheaper hotels, add a free day"
    end
    UI->>API: POST /api/travel/approve
    API->>G: Command(resume={approved, feedback})
    G->>DB: load checkpoint
    G->>G: Final agent polishes / revises
    G-->>API: final_response
    API-->>UI: Final travel plan
    UI-->>U: 📄 Copy · Download PDF
```

---

## 🤖 The Agents

<details open>
<summary><b>🧠 Supervisor</b>: guardrail and planner</summary>

Makes two LLM calls:
1. **Guardrail**: returns `{"allowed": bool, "reason": str}`. Accepts destinations, flights, hotels, weather, budgets, visas, transport, food, packing and itineraries.
2. **Router**: returns `selected_agents`, the structured `trip_constraints` and its `reasoning`, all of which are shown in the UI's *Execution Plan* panel.
</details>

<details>
<summary><b>🛫 Flight Agent</b></summary>

Calls the `list_airports` and `list_airlines` tools on the AviationStack MCP server, then asks the LLM for the likely departure and arrival airports, the airlines on the route, typical flight time, an estimated fare range, peak-season warnings and booking advice.
</details>

<details>
<summary><b>🏨 Hotel Agent</b></summary>

Runs a live `tavily_search` for *"Best hotels for &lt;your request&gt;"*. If the search fails, it switches to general accommodation guidance and labels it as non-live advice.
</details>

<details>
<summary><b>🗺️ Itinerary Agent</b></summary>

Combines the user query, trip constraints, flight results and hotel results into a practical, budget-aware draft that's ready for review.
</details>

<details>
<summary><b>🙋 Human Approval</b></summary>

Calls LangGraph's `interrupt()`, which saves the state to Postgres and pauses the run. It resumes when `/api/travel/approve` is called with `{approved, feedback}`.
</details>

<details>
<summary><b>✨ Final Agent</b></summary>

Produces the finished plan with these sections: **Trip Summary · Flight Information · Hotel Suggestions · Day-by-Day Itinerary · Final Recommendations**. If you asked for a revision, it applies your feedback here.
</details>

---

## 🏗️ System Architecture

```mermaid
flowchart LR
    subgraph Client["🌐 Browser"]
        HTML["index.html<br/>script.js · style.css"]
    end

    subgraph Server["⚡ FastAPI · app.py"]
        R1["GET /"]
        R2["POST /api/travel"]
        R3["POST /api/travel/approve"]
        R4["GET /health"]
    end

    subgraph Core["🕸️ backend.py"]
        LG["LangGraph StateGraph"]
        LLM["ChatGroq<br/>openai/gpt-oss-120b"]
    end

    subgraph Tools["🔌 mcp_client.py"]
        T1["Tavily MCP<br/><i>streamable HTTP</i>"]
        T2["AviationStack MCP<br/><i>stdio via uvx</i>"]
    end

    PG[("💾 PostgreSQL<br/>PostgresSaver")]

    HTML <--> R1 & R2 & R3
    R2 & R3 --> LG
    LG <--> LLM
    LG <--> T1 & T2
    LG <--> PG
```

---

## 🖥️ Using the App

1. **Describe your trip.** Type a request or pick one of the quick prompts:
   > *Plan a complete 7 days Japan trip from Saudi Arabia including flights, hotels and sightseeing.*
2. **See the execution plan.** The UI shows whether the guardrail passed, the supervisor's reasoning, and a chip for each agent that was selected.
3. **Review the draft.** The draft itinerary appears together with an approval panel.
4. **Approve or revise.**
   - ✅ **Approve** to get a polished final plan.
   - ✏️ **Revise** with feedback such as *"reduce the hotel cost and add one free day"*. The final agent rewrites the plan to match.
5. **Export.** Copy the plan to your clipboard or download it as a PDF.

---

## 📡 API Reference

### `POST /api/travel`
Starts a new planning run. The run pauses at human approval.

```json
// Request
{ "message": "Plan a 5 day Dubai trip from Riyadh", "thread_id": null }
```

```json
// Response (abridged)
{
  "success": true,
  "thread_id": "user_3f9c...",
  "requires_approval": true,
  "answer": "## Draft itinerary ...",
  "selected_agents": ["flight_agent", "hotel_agent", "itinerary_agent"],
  "trip_constraints": { "destination": "Dubai", "origin": "Riyadh", "duration": "5 days", "...": "..." },
  "supervisor_reasoning": "...",
  "guardrail_allowed": true,
  "llm_calls": 5
}
```

### `POST /api/travel/approve`
Resumes a paused run. `feedback` is **required** when `approved` is `false`.

```json
{ "thread_id": "user_3f9c...", "approved": false, "feedback": "Make it cheaper" }
```

### `GET /health`
Returns the service status and a list of enabled features.

---

## 🚀 Getting Started

### Prerequisites
- Python **3.11+**
- A **PostgreSQL** database, local or hosted (for example Render or Neon)
- [`uv`](https://docs.astral.sh/uv/), which provides the `uvx` command used to launch the AviationStack MCP server
- API keys for: [Groq](https://console.groq.com) · [Tavily](https://tavily.com) · [AviationStack](https://aviationstack.com)

### 1. Clone and install

```bash
git clone https://github.com/MuteebMMN/TripeMate-MultiAgent.git
cd TripeMate-MultiAgent

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment variables

Copy the example file and fill in your keys:

```bash
cp .env.example .env
```

| Variable | Required | Description |
|---|:---:|---|
| `GROQ_API_KEY` | ✅ | LLM inference |
| `DATABASE_URL` | ✅ | PostgreSQL connection string. `sslmode=require` is added automatically |
| `TAVILY_API_KEY` | ✅ | Hotel web search |
| `AVIATIONSTACK_API_KEY` | ✅ | Airport and airline data |
| `DEFAULT_ORIGIN_IATA` | ➖ | Default departure airport code |
| `OPENWEATHER_API_KEY` | ➖ | Custom weather MCP server |
| `LANGSMITH_*` | ➖ | Optional LangSmith tracing |

### 3. Run

```bash
python app.py
```

Then open **http://127.0.0.1:8000** 🎉

### 🐳 Run with Docker

```bash
docker build -t tripmate-ai .
docker run -p 8000:8000 --env-file .env tripmate-ai
```

---

## 📁 Project Structure

```
TripeMate-MultiAgent/
├── app.py              # FastAPI server: routes, request models, HTML serving
├── backend.py          # LangGraph state, agents, routing, Postgres checkpointer
├── mcp_client.py       # MCP client setup (Tavily, AviationStack, Weather)
├── templates/
│   └── index.html      # Single-page UI
├── static/
│   ├── script.js       # API calls, approval flow, markdown rendering, PDF export
│   └── style.css       # Styling
├── requirements.txt
├── Dockerfile
└── .env.example
```

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | **LangGraph** `StateGraph`, `interrupt()`, `Command(resume=…)` |
| LLM | **Groq**: `openai/gpt-oss-120b` via `langchain-groq` |
| Tools | **Model Context Protocol** via `langchain-mcp-adapters` |
| Persistence | **PostgreSQL** + `langgraph-checkpoint-postgres` |
| API | **FastAPI** + Uvicorn |
| Frontend | HTML · CSS · Vanilla JS |
| Deployment | Docker |

---

<div align="center">

Built by **[Muteeb Nasir](https://github.com/MuteebMMN)** · Have a good trip! 🌍

</div>
