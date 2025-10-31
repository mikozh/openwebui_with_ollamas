# Open WebUI + Dual Ollama (Embeddings service + LLM service)

Spin up **Open WebUI** with **two Ollama services** on the same Docker network:

* **`ollama-embeddings`** — dedicated to embeddings/RAG
* **`ollama-llm`** — dedicated to chat/inference
* **`openwebui`** — points to the LLM service by default and can reach both

```
┌──────────────────────┐         ┌───────────────────────┐
│      openwebui       │ ─────▶  │     ollama-llm        │
│ OLLAMA_BASE_URL →    │         │  :11434 (internal)    │
│ http://ollama-llm    │         │  runs your LLM model  │
└─────────┬────────────┘         └───────────┬───────────┘
          │                                  │
          │        same Docker network "ai"  │
          ▼                                  ▼
┌──────────────────────────┐       (both visible to Open WebUI)
│   ollama-embeddings      │
│   :11434 (internal)      │
│   runs your embedding    │
│   model                  │
└──────────────────────────┘
```

---

## Contents

* [Prerequisites](#prerequisites)
* [Files](#files)
* [Quick start](#quick-start)
* [Access & basic usage](#access--basic-usage)
* [Switching endpoints in Open WebUI](#switching-endpoints-in-open-webui)
* [Changing models](#changing-models)
* [Verify models are loaded](#verify-models-are-loaded)
* [Using embeddings (examples)](#using-embeddings-examples)
* [Calling the LLM (examples)](#calling-the-llm-examples)
* [GPU notes](#gpu-notes)
* [Expose Ollama ports to host (optional)](#expose-ollama-ports-to-host-optional)
* [Maintenance & upgrades](#maintenance--upgrades)
* [Troubleshooting](#troubleshooting)
* [Security notes](#security-notes)
* [License](#license)

---

## Prerequisites

* **Docker** (Desktop or Engine) and **docker compose v2**
* Sufficient disk space for model weights/caches (varies by models and quantization)
* **NVIDIA GPU** with recent drivers + NVIDIA Container Toolkit

---

## Files

* `docker-compose.yml` — this stack:

  * starts **Open WebUI** on host **`3000`**
  * starts **two Ollama** services (internal-only)
  * pre-pulls one **embeddings** model and one **LLM** (via the `entrypoint` lines)
  * includes health checks and persistent volumes
  * includes GPU reservations under `deploy:` for both Ollama services

> Open WebUI is preconfigured to talk to `http://ollama-llm:11434` via `OLLAMA_BASE_URL`. You can switch the target inside the UI at any time.

---

## Quick start

1. Save the provided `docker-compose.yml` next to this README.
2. Launch:

   ```bash
   docker compose up -d
   ```
3. Check status:

   ```bash
   docker compose ps
   docker compose logs -f openwebui
   ```
4. Open **[http://localhost:3000](http://localhost:3000)**.

---

## Access & basic usage

* **Open WebUI**: [http://localhost:3000](http://localhost:3000)
  Default backend (via env): `http://ollama-llm:11434`

Both **`ollama-llm`** and **`ollama-embeddings`** are on the same internal network and discoverable by service name.

---

## Switching endpoints in Open WebUI

In the Open WebUI interface:

1. Go to **Settings → Connections** (wording may vary).
2. Set the Base URL(s):

   * **Chat (default):** `http://ollama-llm:11434`
   * **Embeddings:** `http://ollama-embeddings:11434`
3. Save. You can switch anytime.

> Some builds allow a separate **Embeddings URL**. If available, point it at `ollama-embeddings` and select your embedding model there.

---

## Changing models

You can change which models are pre-pulled or used at runtime without altering the overall architecture.

### A) Edit the compose file (pre-pull on startup)

In each service’s `entrypoint`, replace the model tag after `ollama pull` with the one you want:

```sh
# inside ollama-embeddings service entrypoint
ollama pull <your-embedding-model>

# inside ollama-llm service entrypoint
ollama pull <your-llm-model>
```

### B) Pull models at runtime (no compose edit)

```bash
# Pull a different LLM into the LLM service
docker compose exec ollama-llm ollama pull <your-llm-model>

# Pull a different embedding model into the embeddings service
docker compose exec ollama-embeddings ollama pull <your-embedding-model>
```

Then select the desired model in Open WebUI (or specify it in your API calls).

---

## Verify models are loaded

```bash
docker compose exec ollama-llm         ollama list
docker compose exec ollama-embeddings  ollama list
```

You should see your chosen LLM/embedding models listed in each service.

---

## Using embeddings (examples)

**Inside the embeddings container:**

```bash
docker compose exec ollama-embeddings bash -lc \
  'curl -s http://localhost:11434/api/embeddings \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"<your-embedding-model>\",\"prompt\":\"hello world\"}" | jq .'
```

**From another service on the same network (`ai`):**

```http
POST http://ollama-embeddings:11434/api/embeddings
Content-Type: application/json

{
  "model": "<your-embedding-model>",
  "prompt": "Your text here"
}
```

---

## Calling the LLM (examples)

**Simple text generation (inside the LLM container):**

```bash
docker compose exec ollama-llm bash -lc \
  'curl -s http://localhost:11434/api/generate \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"<your-llm-model>\",\"prompt\":\"Write a haiku about autumn\"}" | jq .'
```

**Multimodal input (only if your model supports images):**

```bash
# $IMG64 should contain a base64-encoded image (no data: prefix).
docker compose exec ollama-llm bash -lc \
  'IMG64=$(base64 -w0 /path/to/image.jpg); \
    curl -s http://localhost:11434/api/generate \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"<your-llm-model>\",\"prompt\":\"Describe this image\",\"images\":[\"$IMG64\"]}" | jq .'
```

---

## GPU notes

Your compose includes a `deploy.resources.reservations.devices` GPU stanza for both Ollama services.
This **applies in Docker Swarm mode**. In plain `docker compose up` (non-Swarm), the `deploy:` block is ignored.

If you’re **not** on Swarm, enable GPUs with one of these:

**A) Compose v2 shorthand**

```yaml
services:
  ollama-llm:
    # ...
    gpus: all
  ollama-embeddings:
    # ...
    gpus: all
```

**B) CLI flag**

```bash
docker compose run --rm --gpus all ollama-llm nvidia-smi
```

Make sure the **NVIDIA Container Toolkit** is installed and `nvidia-smi` works on the host.

---

## Expose Ollama ports to host (optional)

By default, both Ollama services are **internal-only**.
Expose if you need direct host access:

```yaml
ollama-embeddings:
  ports:
    - "11435:11434"
ollama-llm:
  ports:
    - "11436:11434"
```

Then call from your host:

* `http://localhost:11435` (embeddings)
* `http://localhost:11436` (LLM)

---

## Maintenance & upgrades

* Update images:

  ```bash
  docker compose pull
  docker compose up -d
  ```
* Change/pull models at any time (see [Changing models](#changing-models)).
* Back up volumes:

  * `openwebui-data`
  * `ollama-embeddings-data`
  * `ollama-llm-data`

---

## Troubleshooting

* **Open WebUI can’t see models**
  Wait for both services to turn **healthy**. Check:

  ```bash
  docker compose logs -f ollama-llm
  docker compose logs -f ollama-embeddings
  ```

  Confirm Open WebUI’s base URL is the intended service name.

* **GPU not being used**
  If not on Swarm, the `deploy:` GPU stanza is ignored. Use `gpus: all` or the CLI flag as above.

* **Port 3000 already in use**
  Change the Open WebUI mapping:

  ```yaml
  openwebui:
    ports:
      - "3001:8080"
  ```

* **Model too large / memory errors**
  Choose a smaller or quantized variant; ensure adequate RAM/VRAM and disk space.

* **Slow first response**
  First run compiles/loads the model; subsequent runs are faster.

---

## Security notes

* **Auth is disabled** in the compose (`WEBUI_AUTH=False`).
  If the host/port is reachable by others, enable authentication:

  ```yaml
  environment:
    - WEBUI_AUTH=True
  ```

  Then configure users in Open WebUI.

* Keep Ollama services internal unless you need host access. If exposing, prefer localhost bindings and/or protect with a reverse proxy + TLS.
