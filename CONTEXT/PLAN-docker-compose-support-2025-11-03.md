# Docker Compose Support Project Plan (GH-101)

## Context & Goals

### Conversation & Decision Log
- 2025-11-03: User requested a comprehensive, execution-ready plan saved under `CONTEXT/` with actionable checklists, full technical context, code references, external source links, and validation criteria for GH issue #101.
- Agreement: Deliver plan as markdown (`PLAN-docker-compose-support-2025-11-03.md`) so future contributors can execute without reopening this discussion.

### Issue Summary (GH-101)
- Goal: Add Docker Compose support that runs the `opencode-web` frontend and the OpenCode server in containers while allowing host-mounted repositories so agents can operate with root privileges in isolation.
- Acceptance Highlights: Provide multi-stage Dockerfiles, compose orchestration for dev/prod, volume strategy, environment variable configuration, inter-service networking, health checks, and documentation.

### Guiding Principles & Constraints
- Prefer Bun tooling (`package.json` scripts, TanStack Start server) for installs/builds to stay aligned with repo standards.
- Containers must enable mounting arbitrary host repos into a writable workspace inside the server container without exposing additional host resources.
- Provide development (hot reload, bind mounts) and production (optimized images, read-only artifacts) experiences with minimal duplication.
- Maintain security posture: expose only required ports, warn about elevated permissions, and gate logging so secrets are not printed outside development contexts.
- Document every knob (env vars, volume paths, ports) so `docker compose` users can override without editing YAML.

## Architecture Overview

### Current State Snapshot
- `package.json`: defines Bun-based workflows (`bun run dev`, `bun run build`, `bun run lint`, etc.) that the Dockerfiles must reuse to avoid divergence.
- `server.ts`: Bun-powered production server that serves the built client, proxies `/api/events` to the OpenCode server, and enforces CORS/SSE behavior; needs environment parity when containerized.
- `src/lib/opencode-config.ts`: centralizes how `OPENCODE_SERVER_URL` / `VITE_OPENCODE_SERVER_URL` are resolved; compose stack must set these env vars consistently.
- `src/lib/opencode-http-api.ts`: enumerates REST endpoints consumed by the UI (sessions, files, events, MCP status, etc.); networking in Compose must ensure `frontend` reaches these endpoints over the internal network.

### Target Containerized Topology
```
[Host]
  └─ docker compose project network (opencode_net)
       ├─ frontend service (Bun + Vite build output served via TanStack Start)
       │    • Exposes port 3000 -> host configurable
       │    • Talks to server service via http://server:8080
       └─ server service (OpenCode agent runtime based on sst/opencode)
            • Mounts host repo(s) into /workspace
            • Provides REST + SSE endpoints on port 8080
```
- Optional bind mount for `./` into frontend container during dev for hot reload.
- Compose profiles: `dev` (bind mounts, `bun run dev`), `prod` (multi-stage build artifacts, lighter runtime).

## Technical Specifications

### Service Definitions
| Service | Dockerfile/Image | Responsibilities | Ports | Depends On |
| --- | --- | --- | --- | --- |
| `frontend` | `docker/frontend/Dockerfile` (multi-stage: deps → build → runtime) | Install dependencies with Bun, run `bun run build`, serve TanStack Start output via `server.ts` in production or `bun run dev` in dev | 3000 internal; published to `${FRONTEND_PORT:-3000}` | `server` availability for API/SSE traffic |
| `server` | `docker/server/Dockerfile` (based on `ghcr.io/sst/opencode` or clone build) | Provide OpenCode agent backend, accept mounted `/workspace`, manage MCP + shell commands | 8080 internal; published to `${SERVER_PORT:-8080}` | Host volumes mounted before start |

### Networking & Ports
| Name | Host Default | Container | Notes |
| --- | --- | --- | --- |
| `frontend` | 3000 | 3000 | Reuse `PORT` env consumed by `server.ts`; allow override via `.env`.
| `server` | 8080 | 8080 | Matches API base consumed by `src/lib/opencode-http-api.ts`.
| Network | n/a | `opencode_net` (bridge) | Declare explicit network to isolate services while enabling name-based DNS (frontend → `http://server:8080`).

### Environment & Configuration Matrix
| Variable | Purpose | Source File Reference | Default | Profile Notes |
| --- | --- | --- | --- | --- |
| `OPENCODE_SERVER_URL` | Backend base URL for both SSR & client | `src/lib/opencode-config.ts` | `http://server:8080` (inside Compose) | Export to frontend container; set to `http://localhost:${SERVER_PORT}` for host access.
| `VITE_OPENCODE_SERVER_URL` | Client-side override for fetches | `src/lib/opencode-config.ts` | Same as above | Must match exposed host URL so browser can reach backend.
| `PORT` | Frontend listening port | `server.ts` | 3000 | Map to `${FRONTEND_PORT}` when running `bun server.ts` in container.
| `SERVER_PORT` | Backend listening port | OpenCode server entry | 8080 | Keep consistent with Compose mapping.
| `LOCAL_CODE_PATH` | Host path to mount into `/workspace` | Compose env | `../..` (example) | Document expectation for absolute host paths when launching Compose.
| `WORKSPACE_PATH` | Container path for code | Compose env | `/workspace` | Use consistent path across server image + docs.
| `NODE_ENV` | Mode flag | Bun runtime | `production` for main Compose, `development` for dev profile | Drives logging and build flow.

### API & Integration Points
- `src/lib/opencode-http-api.ts`: Compose must ensure the following endpoints remain reachable when proxied through `server.ts`: `/agent`, `/session/*`, `/file/*`, `/event` (SSE), `/mcp`, `/command`, etc.
- `server.ts`: `/api/events` proxy expects `getOpencodeServerUrl()` to resolve to the server container; configure envs accordingly and ensure network-level DNS resolution (`server`).
- SSE connections require `idleTimeout: 0`; keep this default in the container by not overriding server bootstrap logic.

### Volume & File Sync Requirements
- Mount host repositories as read-write volumes into the server container (default `/workspace`).
- Provide optional mount for SSH keys or `.gitconfig` via Compose override if needed (document but leave commented).
- Use cached/consistent options on macOS/Windows (e.g., `:cached` or `:delegated`) when possible.
- Frontend container: in dev profile, mount repo root to enable hot reload; in production, copy source during build to keep runtime image small.

### Health & Observability
- Add Compose health checks:
  - Frontend: `curl -f http://localhost:3000/health` (create lightweight endpoint or reuse `/` once build ready).
  - Server: `curl -f http://localhost:8080/config` or `/health` if available; otherwise create script calling `/config`.
- Use `depends_on` with `condition: service_healthy` to gate frontend start until server is ready.
- Log volume mount warnings when host path missing; fail fast to avoid silent misconfiguration.

## Code References

### Existing Files
| Path | Relevance |
| --- | --- |
| `package.json` | Source of Bun scripts that Dockerfiles must run (`bun run build`, `bun run dev`, `bun run lint`). |
| `server.ts` | Runtime entry for serving built frontend and proxying events; container entrypoint for frontend image. |
| `src/lib/opencode-config.ts` | Documents env variable expectations for server URL resolution. |
| `src/lib/opencode-http-api.ts` | Enumerates all frontend → backend endpoints that must remain routable through Compose networking. |
| `script/vite-build.ts` | Custom build pipeline invoked by `bun run build`; required in Docker build stage. |
| `CONTEXT/IMPLEMENTATION-SUMMARY-2025-11-03.md` | Existing context record to cross-link once Docker work begins. |

### New or Updated Files
| Path | Action |
| --- | --- |
| `docker/frontend/Dockerfile` | New multi-stage frontend image definition (deps → build → runtime). |
| `docker/server/Dockerfile` | New backend image based on `sst/opencode`, installing agent dependencies and preparing `/workspace`. |
| `docker/docker-compose.yml` | Base Compose stack targeting production-like workflow. |
| `docker/docker-compose.dev.yml` | Overrides enabling bind mounts, hot reload, and developer-centric defaults. |
| `docker/.dockerignore` | Prevents copying node_modules, build artifacts, and `CONTEXT/` into Docker contexts. |
| `docker/.env.example` | Document all Compose-consumed env vars with safe defaults. |
| `docs/DOCKER.md` (or `README.md` section) | Instructions for building, running, and troubleshooting the Compose stack. |

### External References
| Name | Git URL | Usage |
| --- | --- | --- |
| SST / OpenCode backend | https://github.com/sst/opencode | Source for server base image or Dockerfile FROM target. |
| SST monorepo examples | https://github.com/sst/sst | Multi-stage Bun build patterns. |
| Chakra UI sandbox Dockerfile | https://github.com/chakra-ui/chakra-ui | Reference multi-stage Bun builds. |
| Forem compose example | https://github.com/forem/forem | Development compose profile conventions and volume mounts. |
| RDFLib compose example | https://github.com/RDFLib/rdflib | Volume mounting strategy reference. |
| Bun official Docker usage | https://github.com/oven-sh/bun | Installing Bun inside containers. |

## Implementation Order & Tasks

### Milestone Ordering Summary
| Order | Milestone | Depends On | Exit Criteria |
| --- | --- | --- | --- |
| 1 | Repository & requirement audit | — | Confirm Bun scripts, env needs, and host prerequisites documented. |
| 2 | Frontend Dockerfile + dev tooling | 1 | Buildable multi-stage image, optional dev entrypoint validated. |
| 3 | Server Dockerfile + workspace prep | 1 | Backend image runs OpenCode server with mount-ready `/workspace`. |
| 4 | Compose orchestration (base + dev profiles) | 2,3 | `docker compose config` succeeds; services reach each other via `server` hostname. |
| 5 | Configuration, docs, and env samples | 4 | `.env.example` + docs cover every knob; onboarding tested. |
| 6 | Validation & release preparation | 2-5 | Tests, linting, Compose smoke tests pass; plan ready for PR. |

### Milestone 1 – Repository & Requirement Audit
 - [x] Verify Bun/Docker versions required; capture in docs (e.g., Bun `1.3.x`, Docker Engine `>=24`).
 - [x] Inventory scripts from `package.json` that containers must run (`bun run build`, `bun run dev`, `bun run lint`).
 - [x] Confirm server expectations around `OPENCODE_SERVER_URL` in `src/lib/opencode-config.ts` to inform Compose env defaults.
 - [x] Identify host directories that need mounting (local OpenCode repos, SSH credentials) and decide on safe defaults.

### Milestone 2 – Frontend Dockerfile & Dev Experience
 - [x] Create `docker/frontend/Dockerfile` with stages: `base-deps` (install Bun + deps), `builder` (`bun run build`), `runtime` (`bun server.ts`).
 - [x] Add `docker/.dockerignore` entries (`node_modules`, `dist`, `CONTEXT`, `**/.git`).
 - [x] Implement optional dev stage that runs `bun run dev --host 0.0.0.0` when profile `dev` active.
 - [x] Parameterize `PORT`, `NODE_ENV`, and server URL envs through Docker build args / runtime env.

### Milestone 3 – Server Dockerfile & Workspace Prep
 - [x] Implement `docker/server/Dockerfile` FROM `ubuntu` or `ghcr.io/sst/opencode` image, install any required packages (git, build-essential, ssh).
 - [x] Clone or COPY server sources, run `bun install`, and expose port 8080.
 - [x] Create `/workspace` directory with correct ownership (root or configurable user) and document expected mount semantics.
 - [x] Expose env vars for authentication, provider configs, and align with `src/lib/opencode-http-api.ts` endpoints.

### Milestone 4 – Compose Orchestration (Base + Dev Profiles)
 - [x] Author `docker/docker-compose.yml` declaring `frontend` & `server`, shared network, healthchecks, and volume definitions.
 - [x] Add `docker/docker-compose.dev.yml` that overrides `frontend` command, mounts repo root, and enables live reload; ensure server volume uses `${LOCAL_CODE_PATH}`.
 - [x] Configure `depends_on` with `condition: service_healthy` so frontend waits for server readiness.
 - [x] Wire environment variables from `.env` into services (ports, URLs, workspace path, optional feature flags).

### Milestone 5 – Configuration, Documentation, and Samples
 - [x] Create `docker/.env.example` covering all vars listed in the matrix, with inline comments.
 - [x] Document usage in `docs/DOCKER.md` (or README section): instructions for building, running, and troubleshooting the Compose stack.
 - [x] Reference host repo mounting instructions, including Windows/macOS path nuances and security considerations.
  - [x] Update existing context docs (e.g., `CONTEXT/IMPLEMENTATION-SUMMARY-2025-11-03.md`) with a short entry pointing to the Compose workflow once implemented.

### Milestone 6 – Validation & Release Prep
- [ ] Run Bun quality gates within containers (`bun run lint`, `bun x tsc --noEmit`, `bun run test` if applicable).
- [ ] Execute `docker compose -f docker/docker-compose.yml up --build` (prod) and verify UI + backend connectivity end-to-end.
- [ ] Execute `docker compose -f docker/docker-compose.dev.yml --profile dev up` for hot reload validation with mounted volume edits.
- [ ] Add documentation screenshots/log excerpts if needed and prepare release notes / CHANGELOG entry referencing GH-101.

## Validation Criteria

| Validation Scope | How to Verify |
| --- | --- |
| Frontend build repeatability | Inside frontend image: `bun install && bun run build` succeeds using repo scripts; `PORT` overrides behave as expected. |
| Backend readiness | `curl -f http://localhost:${SERVER_PORT}/config` returns JSON when stack is up; SSE endpoint `/event` streams when UI opens a session. |
| Compose integrity | `docker compose -f docker/docker-compose.yml config` and `docker compose -f docker/docker-compose.dev.yml config` pass without warnings; health checks gate startup order. |
| Volume mounts | Editing a file in `${LOCAL_CODE_PATH}` reflects inside server container at `/workspace` and actions in UI (e.g., list files) show updated content. |
| Quality gates | Run within host or container:
```bash
bun run lint
bun x tsc --noEmit
bun run test
```
|
| Documentation completeness | New/updated docs explain prerequisites, commands, profiles, env vars, troubleshooting, and link to GH-101 for traceability. |

### End-to-End Smoke Test Script
```bash
cd docker
cp .env.example .env # adjust ports/paths as needed
docker compose up --build -d
docker compose logs -f server
open http://localhost:${FRONTEND_PORT:-3000}
```
- Confirm login/session creation works, SSE updates stream, and file operations succeed inside mounted workspace.
- Tear down with `docker compose down -v` and ensure no residual containers or volumes remain.

### Exit Definition
A milestone is complete only when its validation steps pass, documentation is updated, and artifacts (Dockerfiles, compose files, env samples) align with references listed above.
