# wiremock-playground

A hands-on playground for learning [WireMock](https://wiremock.org/) — a tool that lets you mock HTTP APIs locally so you never have to depend on staging or production services during development.

---

## Table of Contents

- [What is WireMock?](#what-is-wiremock)
- [Why use it?](#why-use-it)
- [Core concepts](#core-concepts)
- [Approach 1 — Docker container (embedded in your project)](#approach-1--docker-container-embedded-in-your-project)
- [Approach 2 — Standalone binary with web UI](#approach-2--standalone-binary-with-web-ui)
- [Stub mapping basics](#stub-mapping-basics)
- [Recording mode — snapshot a real API](#recording-mode--snapshot-a-real-api)
- [Useful resources](#useful-resources)

---

## What is WireMock?

WireMock is an open-source HTTP mock server. It intercepts HTTP requests made by your application and returns pre-configured (stubbed) responses — all without touching any real backend service.

You can:

- Define stubs manually as JSON files that live alongside your code.
- Record real API traffic once and replay it forever.
- Simulate edge cases (slow responses, errors, partial failures) that are hard to trigger in a real environment.

---

## Why use it?

| Problem | How WireMock helps |
|---|---|
| Stage/prod APIs are flaky or rate-limited | Run a local copy that never goes down |
| Onboarding takes hours waiting for environment access | New developers clone the repo and start immediately |
| A third-party API changes and breaks your build | Your stub stays frozen at the last known-good response |
| You need to test a 500 error or a timeout | Trivial to configure in a stub; impossible to trigger reliably in prod |
| CI pipelines call real APIs | Replace them with deterministic stubs — no network, no credentials |

The key insight is that WireMock lets you **lock in a snapshot of a working API** and develop against it. When the real API changes, you update the snapshot intentionally rather than having it break you by surprise.

---

## Core concepts

| Concept | Description |
|---|---|
| **Stub mapping** | A JSON file that pairs a request pattern with a canned response |
| **Mappings directory** | Folder (usually `mappings/`) where stub JSON files are stored |
| **`__files` directory** | Folder for response body files referenced by stubs |
| **Admin API** | REST API at `/__admin` for managing stubs at runtime |
| **Recording** | Proxy mode that records real API calls as stub files automatically |
| **Scenarios** | State machine to simulate multi-step flows (e.g., first call returns empty list, second returns item after creation) |

---

## Approach 1 — Docker container (embedded in your project)

This is the recommended approach for team projects. The WireMock container lives right next to your application in a `docker-compose.yml`, so every developer gets the same mock server just by running `docker compose up`.

### Directory layout

```
my-project/
├── docker-compose.yml
└── wiremock/
    ├── mappings/          # ← stub JSON files go here
    │   └── get-user.json
    └── __files/           # ← large response bodies go here
        └── user-response.json
```

### `docker-compose.yml`

```yaml
services:
  app:
    build: .
    environment:
      API_BASE_URL: http://wiremock:8080   # point your app at the mock
    depends_on:
      - wiremock

  wiremock:
    image: wiremock/wiremock:3x             # official image
    ports:
      - "8080:8080"
    volumes:
      - ./wiremock/mappings:/home/wiremock/mappings
      - ./wiremock/__files:/home/wiremock/__files
    command: --verbose
```

### Run it

```bash
docker compose up
# WireMock is now available at http://localhost:8080
# Admin UI is at http://localhost:8080/__admin
```

### Add a stub on the fly (no restart needed)

```bash
curl -X POST http://localhost:8080/__admin/mappings \
  -H 'Content-Type: application/json' \
  -d '{
    "request":  { "method": "GET", "url": "/api/users/1" },
    "response": { "status": 200, "jsonBody": { "id": 1, "name": "Alice" } }
  }'
```

> **Tip:** Stubs added via the API are ephemeral. Write them to `wiremock/mappings/` as JSON files to persist them across restarts.

---

## Approach 2 — Standalone binary with web UI

WireMock ships as a self-contained JAR (Java required) or as a Docker image that bundles a community-built web UI. This is handy for quick exploration, demoing an API, or when you want a visual interface to manage stubs without writing JSON by hand.

### Option A — JAR (requires Java 11+)

```bash
# Download the latest standalone JAR
curl -L -o wiremock.jar \
  https://repo1.maven.org/maven2/org/wiremock/wiremock-standalone/3.10.0/wiremock-standalone-3.10.0.jar

# Run it
java -jar wiremock.jar --port 8080 --verbose

# Open the admin API in your browser
open http://localhost:8080/__admin
```

The admin API at `/__admin` exposes all stub management endpoints as JSON. There is no built-in graphical UI in the official JAR, but see Option B below.

### Option B — Docker image with graphical UI (`holomekc/wiremock-gui`)

The community image `holomekc/wiremock-gui` bundles a full web UI on top of the standard WireMock server:

```bash
docker run -it --rm \
  -p 8080:8080 \
  -v $(pwd)/mappings:/home/wiremock/mappings \
  -v $(pwd)/__files:/home/wiremock/__files \
  --name wiremock \
  holomekc/wiremock-gui:latest
```

Then open **http://localhost:8080/__admin/webapp** in your browser.

The UI lets you:

- Browse, create, edit, and delete stub mappings visually.
- Inspect the **request journal** (every request the server has received).
- Start and stop **recording** against a real API.
- Manage **scenarios** and reset state.
- Upload and edit **body files**.

---

## Stub mapping basics

A stub is a JSON file with two top-level keys: `request` (what to match) and `response` (what to return).

### `wiremock/mappings/get-products.json`

```json
{
  "request": {
    "method": "GET",
    "url": "/api/products"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": [
      { "id": 1, "name": "Widget", "price": 9.99 },
      { "id": 2, "name": "Gadget", "price": 24.99 }
    ]
  }
}
```

### Simulating a slow response (latency)

```json
{
  "request": { "method": "GET", "url": "/api/products" },
  "response": {
    "status": 200,
    "jsonBody": [],
    "fixedDelayMilliseconds": 3000
  }
}
```

### Simulating an error

```json
{
  "request": { "method": "POST", "url": "/api/orders" },
  "response": {
    "status": 503,
    "jsonBody": { "error": "Service temporarily unavailable" }
  }
}
```

### Matching on query parameters or headers

```json
{
  "request": {
    "method": "GET",
    "urlPath": "/api/users",
    "queryParameters": {
      "role": { "equalTo": "admin" }
    }
  },
  "response": {
    "status": 200,
    "jsonBody": [{ "id": 99, "name": "Admin User" }]
  }
}
```

---

## Recording mode — snapshot a real API

Recording is the fastest way to create stubs. WireMock acts as a proxy: you point it at the real API, run your application normally, and WireMock captures every request/response pair as a stub file.

```bash
# Start recording against a real API
curl -X POST http://localhost:8080/__admin/recordings/start \
  -H 'Content-Type: application/json' \
  -d '{ "targetBaseUrl": "https://api.example.com" }'

# ... exercise your application as normal ...

# Stop recording — stubs are written to the mappings directory
curl -X POST http://localhost:8080/__admin/recordings/stop
```

After stopping, the captured stubs appear in `wiremock/mappings/`. Commit them to source control and your team has a permanent, offline snapshot of the API exactly as it was when recording happened.

---

## Useful resources

- [WireMock documentation](https://wiremock.org/docs/)
- [Official Docker image (`wiremock/wiremock`)](https://hub.docker.com/r/wiremock/wiremock)
- [Community GUI image (`holomekc/wiremock-gui`)](https://hub.docker.com/r/holomekc/wiremock-gui)
- [Stub mapping reference](https://wiremock.org/docs/stubbing/)
- [Request matching reference](https://wiremock.org/docs/request-matching/)
- [Recording and playback](https://wiremock.org/docs/record-playback/)
- [Simulating faults](https://wiremock.org/docs/simulating-faults/)
