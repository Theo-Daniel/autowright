# Autowright

**A self-hosted, multi-stage AI software-development workflow built with Temporal, OpenHands, Ollama, FastAPI, and Next.js.**

Autowright takes a repository URL and a plain-English development request, uses role-based planning for more complex work, asks an OpenHands coding agent to implement the change, and runs a separate AI-assisted review of the resulting Git diff. Temporal coordinates the long-running stages and records execution history.

> **Project status:** Engineering prototype / development environment. The source implements the workflow below; end-to-end reliability, automated deployment, and production security have not been independently established. The developer agent creates a local branch and commit; the current implementation does **not** automatically push or open a pull request.

## What it does

1. **Classify:** Decide whether a request needs a detailed planning cycle or a shorter path.
2. **Analyze:** Produce a project brief (complex tasks only).
3. **Plan:** Generate requirements and acceptance criteria (complex tasks only).
4. **Design:** Propose architecture and implementation changes (complex tasks only).
5. **Break down:** Produce implementation stories (complex tasks only).
6. **Implement:** Run OpenHands against the selected repository; instruct it to inspect code, make changes, run checks, self-review, and commit.
7. **Review:** Ask a separate Ollama-backed QA worker to evaluate the Git diff against the task and, when available, its planning artifacts.

Simple tasks skip the four extended planning stages. The planning roles are **BMAD-inspired** prompts; the implementation is not a claim of official BMAD certification or compatibility with a particular external BMAD release.

## Architecture

```text
                         Next.js 15 web interface
                         (submit + task status)
                                   |
                              REST / WebSocket
                                   |
                              FastAPI API
                                   |
                          Temporal workflow
                                   |
             +---------------------+-------------------+
             |                     |                   |
      planning-queue           agent-queue          qa-queue
       Planning worker        Developer worker     QA worker
       Ollama prompts          OpenHands + Ollama   Ollama review
       Classification,         Clone -> branch ->  Git diff + PRD
       brief, PRD,             edit -> test ->      -> verdict
       design, stories         local commit
             |                     |                   |
             +---------------------+-------------------+
                                   |
                           Temporal result/history
```

The Docker Compose development stack includes **PostgreSQL** for Temporal persistence, **Temporal Server/UI**, the **FastAPI service**, and distinct **planning**, **developer**, and **QA** workers. **Ollama runs on the host machine**, rather than in this Compose stack.

### Where the implementation lives

| Area | Files | Responsibility |
| --- | --- | --- |
| Web application | [`web/app/`](web/app/), [`web/components/`](web/components/) | Requirement submission, task feed, API availability indicator |
| REST and WebSocket API | [`api/main.py`](api/main.py), [`api/routes/`](api/routes/) | Starts workflows; exposes status and polling-based WebSocket updates |
| Orchestration | [`agents/agent/worker.py`](agents/agent/worker.py) | Registers `DevPipelineWorkflow` on `agent-queue`, controls developer activity |
| Planning worker | [`agents/planning/worker.py`](agents/planning/worker.py), [`agents/planning/prompts/`](agents/planning/prompts/) | Classification, analyst, PM, architect, and scrum-master activities |
| OpenHands execution | [`agents/agent/worker.py`](agents/agent/worker.py) | Creates sandbox task instructions and collects development results |
| QA worker | [`agents/qa/worker.py`](agents/qa/worker.py), [`agents/qa/prompts/qa.md`](agents/qa/prompts/qa.md) | Reviews Git diff and returns structured feedback |
| Infrastructure | [`docker-compose.yaml`](docker-compose.yaml), [`k8s/`](k8s/), [`scripts/`](scripts/) | Local containers and experimental Minikube deployment |

## Tech stack

- **Python 3.12**, FastAPI and the Temporal Python SDK
- **Next.js 15**, React 19, TypeScript and Tailwind CSS
- **OpenHands CodeActAgent** for repository-aware coding tasks
- **Ollama** with locally hosted coding models
- **Temporal** for workflow orchestration, activity retry policies and execution history
- **Docker Compose**, PostgreSQL and Kubernetes/Minikube deployment manifests

## Run locally

### Requirements

- Docker Desktop / Docker Compose
- Node.js 20+ and npm (for the standalone web frontend)
- [Ollama](https://ollama.com/) installed and running on the host
- A machine capable of running a local coding model such as `qwen2.5-coder:14b`
- Git access to the target repository

### 1. Clone and prepare the model

```bash
git clone https://github.com/Theo-Daniel/autowright.git
cd autowright

ollama pull qwen2.5-coder:14b
```

Start Ollama if it is not already running:

```bash
ollama serve
```

The Compose configuration uses the host Ollama endpoint (`host.docker.internal:11434`) and configures the planning, development and QA workers to use the `qwen2.5-coder:14b` model. You can change the model values in `docker-compose.yaml` to match an installed model.

### 2. Start the backend stack

From the repository root:

```bash
docker compose up --build
```

This starts Temporal, PostgreSQL, the API, and the three workers. On a first run, image builds and model setup can be resource-intensive.

### 3. Start the web interface

In another terminal:

```bash
cd web
npm install
npm run dev
```

Open <http://localhost:3000>, enter a Git repository URL and a development requirement, then submit the task.

### 4. Check the services

| Component | Local address |
| --- | --- |
| Web interface | <http://localhost:3000> |
| API health | <http://localhost:8000/health> |
| API documentation | <http://localhost:8000/docs> |
| Temporal Web UI | <http://localhost:8080> |
| Temporal gRPC (host-mapped) | `localhost:7234` |

The frontend's availability indicator calls the API health endpoint; it is **not** a comprehensive health check of every worker or the local language model.

### Submit directly through the API

```bash
curl -X POST http://localhost:8000/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "repo_url": "https://github.com/you/example-repo",
    "requirement": "Add a GET /health endpoint"
  }'
```

The API returns a task identifier and Temporal workflow identifier. Use the web task feed or Temporal UI to inspect progression and results. Try the system first with a small, disposable test repository that you own or are authorised to modify.

## Workflow design decisions

**Specialised queues.** Planning tasks run on `planning-queue`, OpenHands work on `agent-queue`, and QA runs on `qa-queue`. This separates LLM-only planning from resource-intensive code execution.

**Durable orchestration.** Temporal persists workflow state and handles activity retries. Long-running or externally visible actions such as Git commits still require careful idempotency and recovery design; the system should not be described as universally “crash-proof.”

**Conditional planning.** A classifier selects a faster development path for simple changes and the extended analyst → PM → architect → stories sequence for more complex requirements.

**Independent review.** The QA worker examines a Git diff and available requirements, returning a verdict and feedback. This is **advisory review**, not a guaranteed merge-blocking or repair gate.

**Local inference.** Planning, coding and QA can use Ollama-hosted local models. The workflow still interacts with external Git repositories when directed to clone them, so local inference does not imply zero network activity.

## Kubernetes / Minikube

The repository includes an additional Minikube-oriented deployment path:

```bash
./scripts/setup-minikube.sh
./scripts/build-images.sh
./scripts/deploy-all.sh
```

The Kubernetes manifests use the existing `openclaw` namespace/image identifiers and differ from the Docker Compose setup, including model configuration. Review the manifests and host resource requirements before using this path. Kubernetes deployment has **not** been verified by this README update.

## Configuration

Selected development settings appear in [`docker-compose.yaml`](docker-compose.yaml):

| Variable | Current local configuration | Meaning |
| --- | --- | --- |
| `TEMPORAL_HOST` | `temporal:7233` | Worker/API connection inside the Docker network |
| `OLLAMA_HOST` | `http://host.docker.internal:11434` | Host Ollama endpoint |
| `PLANNING_MODEL` | `qwen2.5-coder:14b` | Planning worker model |
| `MODEL` | `ollama/qwen2.5-coder:14b` | OpenHands model identifier |
| `QA_MODEL` | `qwen2.5-coder:14b` | QA worker model |
| `RUNTIME_TYPE` | `docker` | OpenHands execution mode for local Compose |
| `WORKSPACE_BASE` | `/workspace` | Shared working area for developer and QA workers |

The included `.env.example` contains older OpenClaw-era settings and does **not** automatically configure all Docker Compose services. Treat `docker-compose.yaml` as the source of truth for the current local stack.

## Current limitations and security notes

- **No automatic Git push/PR:** The developer prompt creates a branch and local commit; remote publication is not implemented as a verified end-to-end feature.
- **QA is not enforced:** The workflow returns the QA verdict but does not automatically repair rejected changes or prevent a local commit. The QA worker currently treats an unparseable model response as an `APPROVED` fallback; this should be changed to fail closed before relying on the verdict.
- **Limited generic test selection:** The coding agent currently attempts `python -m pytest`, which does not cover test suites in every language or framework.
- **No dedicated project test suite/CI detected:** The repository has deployment scripts but no committed automated tests or GitHub Actions test workflow in the reviewed tree. Demonstration and testing claims should be independently validated.
- **Development-only trust boundary:** The developer container mounts the Docker socket and executes code from selected repositories. Use only trusted repositories in an isolated development environment; do not expose this setup directly as a public service.
- **Legacy branding:** Some application labels and manifests still use `OpenClaw`. These are retained in the current source and should be renamed consistently in a separate code change.

## Further reading

- [`ARTICLE.md`](ARTICLE.md) — the longer write-up describing design decisions and the original development approach
- [`CLAUDE.md`](CLAUDE.md) — developer-oriented notes and commands
- [Temporal documentation](https://docs.temporal.io/)
- [OpenHands](https://github.com/All-Hands-AI/OpenHands)
- [Ollama](https://ollama.com/)
- [BMAD Method](https://docs.bmad-method.org/)

## License

[MIT](LICENSE)
