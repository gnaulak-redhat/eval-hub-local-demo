# Eval Hub Local Test

## Prerequisites

- MLflow server running at `http://localhost:5000`
- OCI registry running at `localhost:5001`
- `crane` tool can be used to validate the OCI (optional)

## Start MLflow server

```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --host 127.0.0.1 \
  --port 5000
```

## Start OCI registry

```bash
podman run -d -p 5001:5000 --name eval-hub-oci-registry docker.io/library/registry:2
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

### Validate

Pull and extract a single tag:

```bash
crane pull localhost:5001/eval-results:$OCI_TAG eval-results.tar --insecure
mkdir -p eval-results-contents && tar -xf eval-results.tar -C eval-results-contents
```

Save manifest only:

```bash
crane manifest localhost:5001/eval-results:$OCI_TAG --insecure > manifest.json
```

Download all tags:

```bash
for tag in $(crane ls localhost:5001/eval-results --insecure); do
  crane pull localhost:5001/eval-results:$tag ${tag}.tar --insecure
  mkdir -p $tag && tar -xf ${tag}.tar -C $tag
done
```

Extracted tarballs typically contain: `manifest.json`, image config (e.g. `config.json` or sha256-named file), and layer tarballs.
