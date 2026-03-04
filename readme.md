# Eval Hub Local Demo

Showcase how eval-hub-server can be used to run evaluation locally. This example hook it up with lighteval framework

## Prerequisites

- MLflow server running at `http://localhost:5000`
- OCI registry running at `localhost:5001`
  - `crane` tool can be used to validate the OCI (optional)
- LLM deployed `http://localhost:8001` 

## Start MLflow server

```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --host localhost \
  --port 5000
```


## Start OCI registry

```bash
podman run -d -p 5001:5000 \
    --name eval-hub-oci-registry \
    -e REGISTRY_STORAGE_DELETE_ENABLED=true \
    docker.io/library/registry:2

```

## Start LLM model

Using llama.cpp
```bash
mkdir -p models
cd models
curl -L -o qwen2.5-1.5b-instruct.gguf \
  https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf
```
Start the server
```bash
llama-server -m qwen2.5-1.5b-instruct-q4_k_m.gguf -c 2048 --port 8001
```

## Install eval-hub-server

```bash
uv sync
source .venv/bin/activate
```
> **Note:** Currently `eval-hub-sdk[adapter, server]` are both sourced from wheel files available locally via `pyproject.toml` bypass. When ready, they will be available from PyPI directly.

## Start eval-hub-server

With MLflow env vars (eval-hub-server from venv):

```bash
MLFLOW_TRACKING_URI=http://localhost:5000 \
SERVICE_URL=http://localhost:8080 \
  eval-hub-server --local --configdir ./config
```


## Evaluation job


Start evaluation job
```bash
curl --request POST \
  --url http://localhost:8080/api/v1/evaluations/jobs \
  --header 'content-type: application/json' \
  --data '{
  "model": {
    "url": "http://localhost:8001",
    "name": "qwen2.5-1.5b-instruct-q4_k_m.gguf"
  },
  "benchmarks": [
    {
      "id": "hellaswag",
      "provider_id": "c_provider1",
      "parameters": {
        "num_examples": 10
      }
    }
  ],
  "experiment": {
    "name": "hellaswag-exp1",
    "tags": [
      {
        "key": "model",
        "value": "qwen2.5-1.5b-instruct-q4_k_m.gguf"
      },
      {
        "key": "benchmark",
        "value": "hellaswag"
      }
    ]
  },
  "exports": {
    "oci": {
      "coordinates": {
        "oci_host": "localhost:5001",
        "oci_repository": "eval-results",
        "oci_tag": "hellaswag-exp1"
      }
    }
  }
}'
```

## Inspect MLFlow

check experiments
```bash
curl -s "http://localhost:5000/api/2.0/mlflow/experiments/get-by-name?experiment_name=hellaswag-exp1" | jq .
```
check metrics
```bash
curl -s -X POST "http://localhost:5000/api/2.0/mlflow/runs/search" \
  -H "Content-Type:  application/json" \
  -d '{"experiment_ids": ["1"]}' | jq '.runs[] | {run_id: .info.run_id, status: .info.status, metrics: .data.metrics, params: .data.params}'
```

## Inspect OCI registry

Set the tag (default `hellaswag-exp1`):

```bash
export OCI_TAG="hellaswag-exp1"
```

List and inspect:

```bash
crane ls localhost:5001/eval-results --insecure

# Inspect a specific tag's manifest
crane manifest localhost:5001/eval-results:$OCI_TAG --insecure

# Inspect config/layers
crane config localhost:5001/eval-results:$OCI_TAG --insecure

# List tags and show manifest for each
for tag in $(crane ls localhost:5001/eval-results --insecure); do
  echo "=== $tag ==="
  crane manifest localhost:5001/eval-results:$tag --insecure | jq .
done
```

### Validate OCI registry uploaded files

Pull and extract a single tag:

```bash
crane pull localhost:5001/eval-results:$OCI_TAG eval-results.tar --insecure
mkdir -p eval-results-contents && tar -xf eval-results.tar -C eval-results-contents

cd eval-results-contents
for f in *.tar.gz; do tar -xzf "$f"; done

# will show you the oci uploaded files.
cat *.json
```

clean up:
```bash
crane delete localhost:5001/eval-results:$OCI_TAG --insecure  
```
