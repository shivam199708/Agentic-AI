# Agentic AI Research Agent

A FastAPI-based agentic AI research system that uses tool-using agents to conduct comprehensive research workflows. The project is currently in early development stages.

## Project Status

🚀 **Just Getting Started** - Currently setting up the foundational architecture with Docker containerization.

## Overview

This project aims to build a FastAPI web application that orchestrates intelligent research agents using LLMs. The initial setup focuses on establishing a robust Docker-based development environment.

## Key Components (In Development)

* Multi-step agent workflows for research planning and execution
* Integration with external research tools (APIs, search engines, etc.)
* Task management and progress tracking
* Web-based UI for launching research tasks

---

## Quick Start with Docker

### Prerequisites

* **Docker** (Desktop on Windows/macOS, or engine on Linux)

### Setup

1. **Create a `.env` file** in the project root with your API credentials:

   ```bash
   OPENAI_API_KEY=your-open-api-key
   TAVILY_API_KEY=your-tavily-api-key
   ```

2. **Build the Docker image:**

   ```bash
   docker build -t agentic-ai .
   ```

3. **Run the container:**

   ```bash
   docker run --rm -it -p 8000:8000 -p 5432:5432 \
     --name agentic-ai \
     --env-file .env \
     agentic-ai
   ```

### Access the Application

* **Web UI:** [http://localhost:8000/](http://localhost:8000/)
* **API Docs:** [http://localhost:8000/docs](http://localhost:8000/docs)

### Stop the Container

Press `Ctrl+C` in the terminal where the container is running, or in another terminal:

```bash
docker stop agentic-ai
```

---

## Project Structure

```
agentic-ai/
├─ main.py                      # FastAPI application entry point
├─ src/
│  ├─ planning_agent.py         # Planning logic for research workflows
│  ├─ agents.py                 # Individual agent implementations
│  └─ research_tools.py         # External tool integrations
├─ templates/
│  └─ index.html                # Web UI
├─ static/                      # Static assets (CSS, JS, images)
├─ docker/
│  └─ entrypoint.sh             # Docker container startup script
├─ requirements.txt             # Python dependencies
├─ Dockerfile                   # Container configuration
└─ README.md                    # This file
```

---

## Environment Variables

Create a `.env` file in the project root:

```bash
# Required
OPENAI_API_KEY=your-open-api-key
TAVILY_API_KEY=your-tavily-api-key

# Optional (defaults provided)
# POSTGRES_USER=app
# POSTGRES_PASSWORD=local
# POSTGRES_DB=appdb
```

---

## Prerequisites

* **Docker** (Desktop on Windows/macOS, or engine on Linux).


* API keys stored in a `.env` file:

  ```
  OPENAI_API_KEY=your-open-api-key
  TAVILY_API_KEY=your-tavily-api-key
  ```

* Python deps are installed by Docker from `requirements.txt`:

  * `fastapi`, `uvicorn`, `sqlalchemy`, `python-dotenv`, `jinja2`, `requests`, `wikipedia`, etc.
  * Plus any libs used by your `aisuite` client.

---

## Environment variables

The app **reads only `DATABASE_URL`** at startup.

* The container’s entrypoint sets a sane default for local dev:

  ```
  postgresql://app:local@127.0.0.1:5432/appdb
  ```
* To use Tavily:

  * Provide `TAVILY_API_KEY` (via `.env` or `-e`).

Optional (if you want to override defaults done by the entrypoint):

* `POSTGRES_USER` (default `app`)
* `POSTGRES_PASSWORD` (default `local`)
* `POSTGRES_DB` (default `appdb`)

---

## Build & Run (local/dev)

### 1) Build

```bash
docker build -t fastapi-postgres-service .
```

### 2) Run (foreground)

```bash
docker run --rm -it  -p 8000:8000  -p 5432:5432  --name fpsvc  --env-file .env  fastapi-postgres-service
```

You should see logs like:

```
🚀 Starting Postgres cluster 17/main...
✅ Postgres is ready
CREATE ROLE
CREATE DATABASE
🔗 DATABASE_URL=postgresql://app:local@127.0.0.1:5432/appdb
INFO:     Uvicorn running on http://0.0.0.0:8000
```

### 3) Open the app

* UI: [http://localhost:8000/](http://localhost:8000/)
* Docs: [http://localhost:8000/docs](http://localhost:8000/docs)

---

## Troubleshooting

**Container fails to start**

* Ensure Docker is running and you have sufficient disk space
* Check that ports 8000 and 5432 are not already in use
* Run with verbose logging: `docker run ... --name agentic-ai 2>&1 | head -50`

**API errors related to API keys**

* Verify your `.env` file is in the project root and has correct formatting
* Ensure you're passing it correctly: `--env-file .env`
* For OpenAI: Make sure your API key has the necessary permissions

**Database connection issues**

* The container initializes the database automatically
* If issues persist, remove and rebuild: `docker stop agentic-ai && docker rm agentic-ai && docker build -t agentic-ai .`

**View container logs**

```bash
docker logs -f agentic-ai
```

---

## Next Steps

- [ ] Complete core agent implementations
- [ ] Add comprehensive testing suite
- [ ] Optimize research workflows
- [ ] Deploy to production environment

---

## Troubleshooting

**I open [http://localhost:8000](http://localhost:8000) and see nothing / errors**

* Confirm `templates/index.html` exists inside the container:

  ```bash
  docker exec -it fpsvc bash -lc "ls -l /app/templates && ls -l /app/static || true"
  ```
* Watch logs while you load the page:

  ```bash
  docker logs -f fpsvc
  ```

**Container asks for a Postgres password on startup**

* The entrypoint uses **UNIX socket + peer auth** for admin tasks (no password).
  Ensure you’re not calling `psql -h 127.0.0.1 -U postgres` in the script—use:

  ```bash
  su -s /bin/bash postgres -c "psql -c '...'"
  ```

**`DATABASE_URL not set` error**

* The entrypoint exports a default DSN. If you overrode it, ensure it’s valid:

  ```
  postgresql://<user>:<password>@<host>:<port>/<database>
  ```

**Tables disappear on restart**

* In your `main.py` you call `Base.metadata.drop_all(...)` on startup.
  Comment it out or guard with an env flag:

  ```python
  if os.getenv("RESET_DB_ON_STARTUP") == "1":
      Base.metadata.drop_all(bind=engine)
  ```

**Tavily / arXiv / Wikipedia errors**

* Provide `TAVILY_API_KEY` and ensure network access, provide in the root dir and `.env` file as follows:
```
# OpenAI API Key
OPENAI_API_KEY=your-open-api-key
TAVILY_API_KEY=your-tavily-api-key
```

* Wikipedia rate limits sometimes; try later or handle exceptions gracefully.

---

## Development tips

* **Hot reload** (optional): For dev, you can run Uvicorn with `--reload` if you mount your code:

  ```bash
  docker run --rm -it -p 8000:8000 -p 5432:5432 \
    -v "$PWD":/app \
    --name fpsvc fastapi-postgres-service \
    bash -lc "pg_ctlcluster \$(psql -V | awk '{print \$3}' | cut -d. -f1) main start && uvicorn main:app --host 0.0.0.0 --port 8000 --reload"
  ```

* **Connect to DB from host:**

  ```bash
  psql "postgresql://app:local@localhost:5432/appdb"
  ```

---
