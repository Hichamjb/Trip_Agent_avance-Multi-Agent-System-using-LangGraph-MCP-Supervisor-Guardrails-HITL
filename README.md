# 🌍 Trip Agent Advanced — Multi-Agent Travel Planning System

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.136%2B-009688.svg)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.2.2-orange.svg)](https://github.com/langchain-ai/langgraph)
[![MCP](https://img.shields.io/badge/MCP-1.28.1-purple.svg)](https://modelcontextprotocol.io/)
[![Groq](https://img.shields.io/badge/LLM-Groq-red.svg)](https://groq.com/)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-336791.svg)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-ready-blue.svg)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> An AI-powered multi-agent travel planning system built with **LangGraph**, **MCP**, **Groq**, **FastAPI**, **PostgreSQL**, and **Human-in-the-Loop (HITL)** approval.

---

## 📌 Overview

**Trip Agent Advanced** is a multi-agent travel planning application that transforms a natural-language travel request into a structured travel plan.

The system uses a **Supervisor Agent** to analyze the request, validate whether it belongs to the travel domain, extract trip constraints, and dynamically select the specialist agents required to complete the task.

The current workflow includes:

* 🧠 Supervisor Agent
* 🛡️ LLM-based Input Guardrail
* ✈️ Flight Agent
* 🏨 Hotel Agent
* 🌦️ Weather Agent
* 💰 Budget Agent
* 🗓️ Itinerary Agent
* 👤 Human-in-the-Loop approval
* ✨ Final Response Agent

The application is exposed through a **FastAPI** web API and an interactive browser interface. LangGraph state is persisted using a **PostgreSQL checkpointer**, allowing a travel-planning thread to pause at the human approval step and resume later.

---

# ✨ Features

## 🤖 Multi-Agent Travel Planning

The system decomposes travel planning into specialized tasks:

| Agent               | Responsibility                                                                   |
| ------------------- | -------------------------------------------------------------------------------- |
| 🧠 Supervisor       | Validates the request, extracts constraints, selects agents and controls routing |
| ✈️ Flight Agent     | Provides flight-related information using AviationStack MCP data                 |
| 🏨 Hotel Agent      | Searches the web for hotel information through Tavily MCP                        |
| 🌦️ Weather Agent   | Retrieves current weather and forecast through OpenWeather MCP                   |
| 💰 Budget Agent     | Evaluates trip affordability and identifies budget risks                         |
| 🗓️ Itinerary Agent | Combines specialist results into a complete draft itinerary                      |
| 👤 Human Approval   | Pauses the workflow for user review                                              |
| ✨ Final Agent       | Produces the final polished travel response                                      |

---

## 🧠 Supervisor + Guardrail

The Supervisor performs two important tasks before travel research begins.

### 1. Travel-domain guardrail

The system uses the LLM to determine whether the request is related to travel planning or travel information.

Supported topics include:

* destinations
* flights
* hotels
* weather
* budgets
* visas
* transportation
* sightseeing
* food
* packing
* itineraries

Clearly unrelated, harmful, or illegal requests can be blocked.

### 2. Dynamic agent selection

The supervisor extracts structured travel constraints such as:

```text
destination
origin
duration
budget
travel_style
special_preferences
```

It then selects the required agents.

Example:

```json
{
  "selected_agents": [
    "flight_agent",
    "hotel_agent",
    "weather_agent",
    "budget_agent",
    "itinerary_agent"
  ],
  "trip_constraints": {
    "destination": "Rome",
    "origin": "Casablanca",
    "duration": "5 days",
    "budget": "€1200",
    "travel_style": "cultural",
    "special_preferences": [
      "history",
      "food"
    ]
  }
}
```

If supervisor parsing fails, the application falls back to the complete travel workflow rather than stopping the request.

---

# 🏗️ Architecture

```text
                         ┌─────────────────────┐
                         │       User          │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      FastAPI        │
                         │    Web Interface    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                     ┌──────────────────────────┐
                     │   Supervisor + Guardrail │
                     │                          │
                     │ • Travel validation      │
                     │ • Constraint extraction  │
                     │ • Agent selection        │
                     └────────────┬─────────────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
                ▼                 ▼                 ▼
        ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
        │ Flight      │   │ Hotel       │   │ Weather     │
        │ Agent       │   │ Agent       │   │ Agent       │
        └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
               │                 │                 │
               ▼                 ▼                 ▼
          AviationStack       Tavily          OpenWeather
             MCP               MCP               MCP
               │                 │                 │
               └─────────────────┼─────────────────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │ Budget Agent  │
                         └───────┬───────┘
                                 │
                                 ▼
                       ┌───────────────────┐
                       │ Itinerary Agent   │
                       │ Draft Generation  │
                       └─────────┬─────────┘
                                 │
                                 ▼
                       ┌───────────────────┐
                       │ Human Approval    │
                       │    interrupt()    │
                       └─────────┬─────────┘
                                 │
                       ┌─────────┴─────────┐
                       │                   │
                  Approved             Revision
                       │                   │
                       └─────────┬─────────┘
                                 ▼
                       ┌───────────────────┐
                       │   Final Agent     │
                       │ Final Travel Plan │
                       └─────────┬─────────┘
                                 │
                                 ▼
                              User
```

The actual LangGraph graph contains the supervisor, specialist agents, itinerary generation, HITL interruption, and final response stages.

---

# 🔄 LangGraph Workflow

The graph starts with the Supervisor:

```text
START
  │
  ▼
Supervisor
  │
  ├── Guardrail blocked ──► END
  │
  ├── Flight Agent
  │
  ├── Hotel Agent
  │
  ├── Weather Agent
  │
  ├── Budget Agent
  │
  └── Itinerary Agent
          │
          ▼
   Human Approval
          │
          ▼
     Final Agent
          │
          ▼
         END
```

The supervisor determines which specialist agents are relevant. The selected agents are then executed according to the configured agent order before the itinerary is generated.

---

# 🔌 MCP Architecture

The project uses `langchain-mcp-adapters` and `MultiServerMCPClient` to communicate with multiple MCP servers.

## MCP Servers

### 🔎 Tavily MCP

Transport:

```text
Streamable HTTP
```

Endpoint:

```text
https://mcp.tavily.com/mcp/
```

Used primarily by the Hotel Agent for web research.

The application calls the Tavily MCP search tool to retrieve hotel and accommodation information.

---

### ✈️ AviationStack MCP

Transport:

```text
stdio
```

The project launches:

```bash
uvx aviationstack-mcp
```

The Flight Agent uses MCP tools including:

```text
list_airports
list_airlines
```

The retrieved airport and airline information is then passed to the Groq LLM to generate flight-related guidance.

> The current implementation should not be described as a complete real-time flight-price booking engine. The Flight Agent generates recommendations from AviationStack MCP information and LLM reasoning.

---

### 🌦️ OpenWeather MCP

The repository contains its own MCP server:

```text
custom_weather_mcp_server.py
```

It uses:

```python
FastMCP("Weather MCP Server")
```

and exposes:

```text
get_current_weather
get_forecast
```

The server communicates with OpenWeather's API.

The Weather Agent calls both tools:

```text
get_current_weather
get_forecast
```

and combines their results for the travel workflow.

---

# ✈️ Flight Agent

The Flight Agent is responsible for flight-related planning.

Its workflow is approximately:

```text
User Request
     ↓
AviationStack MCP
     ↓
Airport Information
     +
Airline Information
     ↓
Groq LLM
     ↓
Flight Recommendations
```

The agent asks the model to provide:

1. Likely departure airport
2. Likely arrival airport
3. Airlines serving the route
4. Typical flight duration
5. Estimated airfare range
6. Peak-season pricing warning
7. Booking advice

---

# 🏨 Hotel Agent

The Hotel Agent performs web research using Tavily MCP.

It constructs a search query such as:

```text
Best hotels for [user travel request]
```

and sends it to the Tavily MCP server.

The returned research is passed into the travel workflow.

If the MCP search fails, the application returns a fallback message asking the final planner to provide general accommodation guidance and clearly label it as non-live advice.

---

# 🌦️ Weather Agent

The Weather Agent first extracts the destination from the user's request.

It then requests:

```text
Current Weather
Forecast
```

through the local OpenWeather MCP server.

The current weather response contains information such as:

```text
city
temperature
feels-like temperature
humidity
weather condition
wind speed
```

The forecast tool returns the first five three-hour forecast entries.

---

# 💰 Budget Agent

The Budget Agent evaluates whether the planned trip is realistic for the user's budget.

It receives:

```text
User Query
Trip Constraints
Flight Results
Hotel Results
Weather Results
```

and produces:

1. Estimated cost categories
2. Budget risk areas
3. Money-saving suggestions
4. Overall feasibility

If exact live prices are unavailable, the prompt instructs the model to clearly label estimates as approximate.

---

# 🗓️ Itinerary Agent

The Itinerary Agent integrates the available specialist results.

It receives:

```text
Trip Constraints
Flight Results
Hotel Results
Weather Results
Budget Results
```

and generates a practical, budget-aware draft itinerary ready for human review.

---

# 👤 Human-in-the-Loop

The project implements a real LangGraph `interrupt()` checkpoint.

Before producing the final response, the graph pauses and sends the user:

```text
Do you approve this itinerary?
```

The interrupt payload contains:

* Draft itinerary
* Approval request
* Selected agents
* Supervisor reasoning
* Expected approval response

The user can:

### ✅ Approve

```json
{
  "approved": true,
  "feedback": ""
}
```

The workflow resumes and the Final Agent generates the final travel response.

### ✏️ Request a revision

```json
{
  "approved": false,
  "feedback": "Reduce the hotel cost and add more free activities."
}
```

The workflow resumes with the feedback, and the Final Agent incorporates the requested changes.

---

# 💾 PostgreSQL Checkpointing

LangGraph state is persisted using:

```python
PostgresSaver
```

The application connects to PostgreSQL using:

```text
DATABASE_URL
```

and automatically adds:

```text
sslmode=require
```

when it is not already present.

The checkpointer is initialized with:

```python
checkpointer = PostgresSaver(_conn)
checkpointer.setup()
```

and then attached when compiling the LangGraph workflow.

This persistence is important for HITL because the graph must resume the same travel-planning thread after the user submits approval or revision feedback.

---

# 🌐 FastAPI Application

The web application is implemented with **FastAPI**, not Flask.

Main file:

```text
app.py
```

The API runs on:

```text
http://127.0.0.1:8000
```

by default.

---

# 🔗 API Endpoints

## `GET /`

Returns the TripMate AI web interface.

---

## `POST /api/travel`

Starts or resumes a travel-planning thread.

### Request

```json
{
  "message": "Plan a 7-day trip to Japan from Morocco with a €2000 budget.",
  "thread_id": null
}
```

`thread_id` is optional for a new request.

### Response

The API returns information including:

```json
{
  "success": true,
  "thread_id": "...",
  "answer": "...",
  "requires_approval": true,
  "approval_request": "...",
  "flight_results": "...",
  "hotel_results": "...",
  "weather_results": "...",
  "budget_results": "...",
  "itinerary": "...",
  "selected_agents": [],
  "trip_constraints": {},
  "supervisor_reasoning": "...",
  "guardrail_allowed": true
}
```

The exact response is produced by `_serialize_result()` in `backend.py`.

---

## `POST /api/travel/approve`

Resumes a paused travel-planning thread.

### Request

```json
{
  "thread_id": "user_...",
  "approved": true,
  "feedback": ""
}
```

For a revision:

```json
{
  "thread_id": "user_...",
  "approved": false,
  "feedback": "Find cheaper hotels and add more free activities."
}
```

The API requires feedback when the draft is rejected for revision.

---

## `GET /health`

Returns a basic application health response.

Example:

```json
{
  "status": "ok",
  "message": "TripMate AI API is running",
  "features": [
    "supervisor_agent",
    "input_guardrail",
    "human_in_the_loop"
  ]
}
```

---

# 🖥️ Web Interface

The frontend is implemented with:

* HTML
* CSS
* Vanilla JavaScript
* Marked.js
* html2pdf.js

The interface provides:

* Travel request input
* Quick prompts
* Supervisor execution plan
* Selected-agent display
* Guardrail status
* Draft itinerary display
* Human approval controls
* Revision feedback
* Copy-to-clipboard
* PDF export

The frontend stores the current LangGraph thread ID in browser `localStorage`, allowing the browser to continue using the same conversation thread.

---

# 📄 PDF Export

The web interface can export the generated travel plan as:

```text
ai-travel-plan.pdf
```

using `html2pdf.js` in the browser.

---

# 📁 Project Structure

The repository currently contains the following main files:

```text
Trip_Agent_avance/
│
├── app.py
│   └── FastAPI application and API endpoints
│
├── backend.py
│   └── LangGraph workflow, agents, state and PostgreSQL checkpointing
│
├── mcp_client.py
│   └── MCP client configuration and tool helpers
│
├── custom_weather_mcp_server.py
│   └── Local OpenWeather MCP server
│
├── requirements.txt
│   └── Python dependencies
│
├── Dockerfile
│   └── Container configuration
│
├── templates/
│   └── index.html
│       └── Web interface
│
├── static/
│   ├── style.css
│   │   └── Application styling
│   │
│   └── script.js
│       └── Frontend logic and API calls
│
├── .gitignore
│
├── .dockerignore
│
├── LICENSE
│
└── README.md
```

The repository root currently contains `app.py`, `backend.py`, `mcp_client.py`, `custom_weather_mcp_server.py`, `requirements.txt`, `Dockerfile`, `LICENSE`, and the frontend directories.

---

# 🧰 Tech Stack

| Layer                 | Technology                            |
| --------------------- | ------------------------------------- |
| Programming Language  | Python 3.11+                          |
| Web Framework         | FastAPI                               |
| ASGI Server           | Uvicorn                               |
| LLM                   | Groq                                  |
| Model                 | Llama 3.3 70B Versatile               |
| Agent Orchestration   | LangGraph                             |
| Tool Protocol         | Model Context Protocol (MCP)          |
| MCP Adapter           | LangChain MCP Adapters                |
| Web Research          | Tavily MCP                            |
| Flight Data           | AviationStack MCP                     |
| Weather               | OpenWeather API via custom MCP server |
| Database              | PostgreSQL                            |
| LangGraph Persistence | PostgresSaver                         |
| Frontend              | HTML / CSS / JavaScript               |
| Markdown Rendering    | Marked.js                             |
| PDF Export            | html2pdf.js                           |
| Containerization      | Docker                                |

The dependency versions currently pinned in the repository include LangGraph 1.2.2, LangChain 1.3.2, LangChain-Groq 1.1.3, FastAPI 0.136.3, `langchain-mcp-adapters` 0.3.0, MCP 1.28.1, and PostgreSQL checkpoint support.

---

# ⚙️ Configuration

The repository does not currently include a `.env.example` file, so create a `.env` file manually in the project root.

Required environment variables:

```env
GROQ_API_KEY=your_groq_api_key

DATABASE_URL=your_postgresql_connection_string

TAVILY_API_KEY=your_tavily_api_key

AVIATION_STACK_API_KEY=your_aviationstack_api_key

OPENWEATHER_API_KEY=your_openweather_api_key
```

The AviationStack key can also be provided using:

```env
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
```

The MCP client accepts both names.

---

# 🚀 Installation

## 1. Clone the repository

```bash
git clone https://github.com/Hichamjb/Trip_Agent_avance-Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL.git

cd Trip_Agent_avance-Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL
```

---

## 2. Create a virtual environment

### Linux / macOS

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

---

## 3. Install Python dependencies

```bash
pip install -r requirements.txt
```

---

## 4. Install `uv`

The AviationStack MCP server is launched through `uvx`.

Install `uv` following the official Astral documentation, then verify:

```bash
uvx --version
```

The application explicitly checks for `uvx` when loading the AviationStack MCP server.

---

## 5. Create `.env`

Create:

```text
.env
```

in the project root:

```env
GROQ_API_KEY=your_groq_api_key
DATABASE_URL=postgresql://username:password@host:5432/database
TAVILY_API_KEY=your_tavily_api_key
AVIATION_STACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
```

---

# 🗄️ PostgreSQL Setup

The application requires PostgreSQL because LangGraph uses `PostgresSaver` for persistent graph state.

Set:

```env
DATABASE_URL=postgresql://username:password@host:5432/database
```

The application automatically adds:

```text
sslmode=require
```

when necessary.

It also initializes the LangGraph checkpoint tables with:

```python
checkpointer.setup()
```

---

# ▶️ Run the Application

Start the FastAPI application:

```bash
python app.py
```

The application starts Uvicorn on:

```text
http://127.0.0.1:8000
```

Open the address in your browser.

Alternatively:

```bash
uvicorn app:app --reload --host 127.0.0.1 --port 8000
```

---

# 🧪 Testing MCP Connections

The MCP client includes a helper for testing the configured MCP servers.

You can run:

```python
import asyncio
from mcp_client import get_all_tools

asyncio.run(get_all_tools())
```

It checks:

```text
tavily
aviationstack
weather
```

and reports the available tools for each server.

---

# 🐳 Docker

The repository contains a `Dockerfile`.

Build:

```bash
docker build -t trip-agent .
```

Run:

```bash
docker run --rm \
  -p 8000:8000 \
  --env-file .env \
  trip-agent
```

Then open:

```text
http://localhost:8000
```

The Docker image is based on Python 3.11 and starts:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

> The repository does not currently provide a `docker-compose.yml`, so PostgreSQL must be provided separately.

---

# 🔐 Security Considerations

The project contains an LLM-based input guardrail, but it should not be considered a complete security boundary.

For production use, consider adding:

* Authentication
* Authorization
* API rate limiting
* Secret management
* HTTPS
* Stronger prompt-injection defenses
* Input/output schema validation
* MCP server access control
* PostgreSQL security configuration
* Request logging and monitoring
* API quota management

Never commit:

```text
.env
```

or API keys to Git.

---

# ⚠️ Current Limitations

## Flight Data

The Flight Agent currently retrieves airport and airline information through AviationStack MCP and uses the LLM to generate flight guidance.

It should **not** be described as a complete flight booking system or guaranteed real-time fare-search engine.

---

## Hotel Search

Hotel information is obtained through Tavily web search.

Results therefore depend on:

* Tavily availability
* Search quality
* Web content
* Current indexed information

The system does not perform hotel reservations.

---

## Weather

Weather is connected to OpenWeather through the project's MCP server.

Weather information is therefore dependent on the OpenWeather API and the configured API key.

---

## Budget

The Budget Agent performs an LLM-based feasibility analysis.

It does not execute a deterministic accounting system or payment operation.

---

## No Booking or Payment

The project does not execute:

* Flight booking
* Hotel booking
* Payment
* Cancellation
* Ticket issuance

The HITL step reviews the generated itinerary; it is not a payment authorization system.

---

## LLM Dependency

The current implementation uses:

```text
Groq
Llama 3.3 70B Versatile
```

The backend requires `GROQ_API_KEY`.

---

# 🔮 Future Improvements

Possible extensions include:

### ✈️ Advanced Flight Search

Add dedicated tools for:

```text
flight search
price comparison
departure dates
arrival dates
connections
baggage
airline filters
```

### 🏨 Dedicated Hotel API

Replace or complement web search with a structured hotel API.

### 🗺️ Maps and Places

Add:

* Google Maps
* OpenStreetMap
* Places APIs
* routing
* distance calculation

### 🧠 Long-Term Memory

Store traveler preferences such as:

```text
preferred airlines
hotel preferences
budget preferences
travel style
favorite activities
```

### 📊 Observability

Add:

* LangSmith
* structured logging
* agent execution traces
* latency monitoring
* MCP tool-call monitoring
* token/cost tracking

### 🔒 Stronger Guardrails

Introduce:

* structured validation
* prompt-injection classifiers
* output verification
* tool authorization policies
* destination/result validation

### ⚡ Parallel Agent Execution

Independent agents such as:

```text
Flight
Hotel
Weather
```

could be executed concurrently to reduce total latency.

### 💳 Transactional Booking

A future production system could add booking APIs behind explicit HITL authorization.

---

# 🎯 Example Request

Example:

```text
I want to travel from Casablanca to Rome for 5 days.
My budget is €1200.
I like history, museums and Italian food.
Please consider the weather and recommend suitable hotels.
```

The system can:

```text
1. Validate the travel request
        ↓
2. Extract destination, origin, duration and budget
        ↓
3. Select relevant specialist agents
        ↓
4. Retrieve flight information
        ↓
5. Search hotel information
        ↓
6. Retrieve current weather and forecast
        ↓
7. Analyze budget feasibility
        ↓
8. Generate a draft itinerary
        ↓
9. Pause for human approval
        ↓
10. Apply approval/revision feedback
        ↓
11. Generate the final travel response
```

---

# 📋 Final Response Format

The Final Agent is instructed to organize the final response into sections such as:

```text
1. Trip Summary
2. Flight Information
3. Hotel Suggestions
4. Weather Information
5. Day-by-Day Itinerary
6. Estimated Budget
7. Final Recommendations
```

It also incorporates human feedback when a revision is requested.

---

# 📄 License

This project is distributed under the MIT License.

See:

```text
LICENSE
```

for details.

---

# 👨‍💻 Author

**Hicham Jabbad**

GitHub:

https://github.com/Hichamjb

---

# ⭐ Project Summary

Trip Agent Advanced demonstrates how **LangGraph + MCP + LLM agents + PostgreSQL persistence + Human-in-the-Loop** can be combined into a travel-planning workflow.

The core architecture is:

```text
                 User
                   │
                   ▼
             FastAPI API
                   │
                   ▼
          Supervisor + Guardrail
                   │
          ┌────────┼─────────┐
          │        │         │
          ▼        ▼         ▼
       Flight    Hotel    Weather
       Agent     Agent     Agent
          │        │         │
          └────────┼─────────┘
                   │
                   ▼
              Budget Agent
                   │
                   ▼
            Itinerary Agent
                   │
                   ▼
          ┌─────────────────┐
          │ Human Approval  │
          │   interrupt()   │
          └────────┬────────┘
                   │
                   ▼
             Final Agent
                   │
                   ▼
              Final Plan
```

The project demonstrates:

**Multi-Agent Orchestration → MCP Tool Integration → Dynamic Routing → Persistent State → Human Review → Final AI Response**
