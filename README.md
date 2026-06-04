# Dify Agent Safety Review

This repository is a security-focused secondary development of
[Dify](https://github.com/langgenius/dify). It keeps the upstream Dify codebase,
then adds a publish-time agent safety gate and a concrete backend SSRF fix found
by auditing Dify's remote-file fetch path.

The original upstream README is preserved as [README.dify.md](./README.dify.md).

## What This Adds

### 1. Agent Safety Review Gate

Before an agent workflow or RAG pipeline is published, Dify now runs a structured
safety review over the workflow graph.

The review checks:

- schema-aware fields for Dify node types instead of broad string scanning;
- prompt-injection patterns in real prompt/config fields;
- risky external HTTP actions;
- URL safety issues such as private ranges, encoded IPs, redirect targets, DNS
  rebinding, disallowed schemes, and metadata-service endpoints;
- domain allowlists through `agent_safety_review.allowed_domains`;
- whether dangerous graph paths pass through an approval node before external
  actions.

Main entry points:

- [api/services/agent_safety_review_service.py](./api/services/agent_safety_review_service.py)
- [api/services/workflow_service.py](./api/services/workflow_service.py)
- [api/services/rag_pipeline/rag_pipeline.py](./api/services/rag_pipeline/rag_pipeline.py)
- [api/controllers/console/app/workflow.py](./api/controllers/console/app/workflow.py)

The implementation notes are in
[docs/agent-safety-review-plugin.md](./docs/agent-safety-review-plugin.md).

### 2. Remote File Redirect SSRF Fix

While reviewing Dify's own code, I traced user-controlled remote file URLs
through:

- `factories.file_factory.remote.get_remote_file_info()`
- `core.helper.download.download_with_size_limit()`
- `core.file.remote_fetcher.make_request()`
- `core.helper.ssrf_proxy.make_request()`

The risky behavior was that remote-file GET/HEAD calls could pass
`follow_redirects=True` straight to the network client. A public URL could
redirect to a private network target, metadata-service address, encoded loopback
IP, or DNS-rebinding host before Dify performed local redirect-target
validation.

This repository fixes that in
[api/core/file/remote_fetcher.py](./api/core/file/remote_fetcher.py):

- remote-file GET/HEAD redirects are followed manually one hop at a time;
- HTTPX automatic redirects are disabled for this path;
- each initial and redirect URL must be HTTP(S);
- decimal, hex, and octal-style IPv4 host forms are parsed;
- DNS is resolved on every hop;
- private, loopback, link-local, reserved, multicast, unspecified, and other
  non-global addresses are blocked;
- unresolved hosts and excessive redirect chains are blocked;
- safe relative redirects still work.

Regression tests live in
[api/tests/unit_tests/core/file/test_remote_fetcher.py](./api/tests/unit_tests/core/file/test_remote_fetcher.py).

## Why It Is AI-Native

This is not a thin wrapper around Dify. It changes the agent publishing path
itself. The system treats an agent workflow as a graph, reasons about whether
risky tool/action paths pass through human approval, and blocks unsafe changes
before they go live.

That makes it useful as an interview project because it combines:

- real open-source code reading;
- production-style backend integration;
- graph-aware agent safety;
- SSRF/security hardening;
- tests and CI as visible proof.

## Validation

Run from the `api` directory:

```bash
uv run --with pytest-mock pytest -o addopts='' \
  tests/unit_tests/services/test_agent_safety_review_service.py \
  tests/unit_tests/core/file/test_remote_fetcher.py \
  tests/unit_tests/services/rag_pipeline/test_rag_pipeline_service.py \
  tests/unit_tests/controllers/console/app/test_workflow.py \
  tests/unit_tests/controllers/console/datasets/rag_pipeline/test_rag_pipeline_workflow.py \
  -q
```

Current result:

```text
175 passed
```

Additional remote-file path regression suite:

```bash
uv run --with pytest-mock pytest -o addopts='' \
  tests/unit_tests/core/file/test_remote_fetcher.py \
  tests/unit_tests/core/helper/test_download.py \
  tests/unit_tests/factories/test_file_factory.py \
  tests/unit_tests/controllers/console/test_remote_files.py \
  tests/unit_tests/controllers/web/test_remote_files.py \
  -q
```

Current result:

```text
104 passed
```

Static checks used:

```bash
uv run --with ruff ruff check \
  services/agent_safety_review_service.py \
  core/file/remote_fetcher.py \
  services/workflow_service.py \
  services/rag_pipeline/rag_pipeline.py \
  controllers/console/app/workflow.py \
  controllers/console/datasets/rag_pipeline/rag_pipeline_workflow.py \
  tests/unit_tests/services/test_agent_safety_review_service.py \
  tests/unit_tests/core/file/test_remote_fetcher.py \
  tests/unit_tests/services/rag_pipeline/test_rag_pipeline_service.py \
  tests/unit_tests/controllers/console/app/test_workflow.py \
  tests/unit_tests/controllers/console/datasets/rag_pipeline/test_rag_pipeline_workflow.py
```

Current result:

```text
All checks passed
```

GitHub Actions coverage is defined in
[.github/workflows/agent-safety-review.yml](./.github/workflows/agent-safety-review.yml).

## Relationship To Upstream

This project is based on Dify and keeps the original codebase for compatibility.
It is intended as a standalone security-oriented fork for demonstration,
interview, and further experimentation.

The original Dify project belongs to LangGenius and its contributors.
