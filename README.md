# RAG Blueprint Helm Chart

A Helm chart that deploys an NVIDIA RAG Blueprint on a RunAI cluster with **lightweight NIM-compatible services**. Designed to run entirely on CPU (no real GPUs required), making it ideal for reference and demo environments with fake or virtual GPUs.

Deploying via Helm ensures RunAI detects the application under **AI Applications** in the UI.

## What Gets Deployed

| Component | Image | Purpose | Resources |
|-----------|-------|---------|-----------|
| **nim-llm** | `python:3.11-slim` | LLM service with OpenAI-compatible `/v1/chat/completions` | 100m–500m CPU, 128–256Mi |
| **nemoretriever-embedding-ms** | `python:3.11-slim` | Embedding service (`/v1/embeddings`) | 100m–500m CPU, 128–256Mi |
| **nemoretriever-ranking-ms** | `python:3.11-slim` | Reranking service (`/v1/ranking`) | 100m–500m CPU, 128–256Mi |
| **rag-server** | `nvcr.io/nvidia/blueprint/rag-server:2.4.0` | RAG orchestrator (connects LLM + embeddings + vector DB) | 200m–2 CPU, 512Mi–2Gi |
| **rag-frontend** | `nvcr.io/nvidia/blueprint/rag-frontend:2.4.0` | Web UI (NodePort on port 3000) | 50m–500m CPU, 64–256Mi |
| **milvus** | `milvusdb/milvus:v2.4.17` | Vector database (CPU mode, FLAT index) | 200m–2 CPU, 512Mi–4Gi |
| **etcd** | `quay.io/coreos/etcd:v3.5.18` | Metadata store for Milvus | 100m–500m CPU, 128–512Mi |
| **rag-redis-master** | `redis:7.4` | Caching layer | 50m–250m CPU, 64–256Mi |
| **rag-minio** | `minio/minio:RELEASE.2024-11-07T00-52-20Z` | Object storage | 50m–500m CPU, 128–512Mi |

**Total: 9 pods, all CPU-only, ~1 CPU / ~4Gi memory footprint.**

## Architecture

```
┌─────────────────┐     ┌──────────────┐
│  rag-frontend   │────▶│  rag-server  │
│  (port 3000)    │     │  (port 8081) │
└─────────────────┘     └──────┬───────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐   ┌────────────┐   ┌────────────┐
        │ nim-llm  │   │ embedding  │   │  ranking   │
        │  service │   │  service   │   │  service   │
        └──────────┘   └────────────┘   └────────────┘

        ┌──────────┐   ┌──────┐   ┌───────┐
        │  milvus  │◀──│ etcd │   │ redis │
        └──────────┘   └──────┘   └───────┘
              │
        ┌──────────┐
        │  minio   │
        └──────────┘
```

## Prerequisites

- `kubectl` configured and pointing to the target cluster
- `helm` v3 installed
- A RunAI project namespace (e.g., `runai-team-a`) must already exist
- An NGC image pull secret must exist in the namespace (default: `dockerregistry-ngcapikey-nvcrio`)

## Quick Start — Deploy to us-demo-west

```bash
# 1. Make sure kubectl is pointing to us-demo-west
kubectl config use-context us-demo-west

# 2. Clone and deploy
git clone https://github.com/chadchappy/rag-demo.git
cd rag-demo
helm install rag-demo . --namespace runai-team-a

# 3. Verify all pods are running
kubectl get pods -n runai-team-a
```

## Configuration

Edit `values.yaml` or pass overrides at install time:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `namespace` | `runai-team-a` | Target RunAI project namespace |
| `imagePullSecret` | `dockerregistry-ngcapikey-nvcrio` | NGC image pull secret name |

### Deploy to a different project

```bash
helm install rag-demo . --namespace runai-user-b \
  --set namespace=runai-user-b
```

## Manage the Release

```bash
# Check status
helm list -n runai-team-a
kubectl get pods -n runai-team-a

# Upgrade after changes
helm upgrade rag-demo . --namespace runai-team-a

# Uninstall
helm uninstall rag-demo --namespace runai-team-a
```

## How the NIM Services Work

The three NIM services (LLM, embedding, reranking) are lightweight Python HTTP servers that implement the same API contracts as NVIDIA NIM containers:

- **LLM**: Responds to `/v1/chat/completions` with a canned response (supports streaming)
- **Embedding**: Responds to `/v1/embeddings` with random 1024-dim vectors
- **Reranking**: Responds to `/v1/ranking` with random relevance scores

This allows the full RAG pipeline (rag-server + frontend) to function end-to-end without real GPU inference.

## Notes

- The rag-server image (`rag-server:2.4.0`) is pulled from NGC and requires the image pull secret
- Milvus runs in **CPU mode** with FLAT indexing (no GPU search)
- The frontend is exposed as a **NodePort** service on port 3000
- RunAI will detect this deployment as an **AI Application** because it is deployed as a Helm release

