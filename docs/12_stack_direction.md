# Colab Infinity — Stack Direction

**Status:** Direction

**Date:** 2026-08-31

**Architectural source of truth:** `Anakyklos/architecture/rfcs/0003-ephemeral-burst-compute.md`

## Decision

The new Colab Infinity / Megingjörð architecture adopts a split stack:

- **Go** for the local control plane;
- **SQLite** for durable local lease state;
- **Python** for ephemeral compute workers and AI/data/scientific workloads;
- **provider adapters** behind a stable `ComputeLease` boundary;
- **JSON manifests + versioned artifacts** for workload input/output contracts.

The previous Google-Colab-specific stack — Playwright/Selenium browser automation, Ngrok, FastAPI as a mandatory remote-server layer, Google Drive as authoritative runtime state, and account/session rotation — is Legacy and does not define the new product architecture.

## Why Go owns the control plane

The dominant problem of the new product is durable lifecycle orchestration rather than ML application logic.

The control plane must handle:

- `ComputeLease` creation and state transitions;
- provider discovery/selection;
- provisioning and teardown;
- process and remote-job lifecycle;
- timeouts and cancellation;
- resource, duration and cost budgets;
- privacy/policy checks;
- artifact movement and integrity verification;
- retry/reconciliation where explicitly allowed;
- durable local state and recovery;
- CLI and future IPC/capability integration.

This problem class matches the Anakyklos Go Technology Palette: concurrent I/O-oriented infrastructure, process management, cancellation, persistence and provider adapters with simple native distribution.

The preferred host artifact is a small native executable with minimal dependencies. Avoid agent frameworks, Redis, message brokers, Kubernetes, mandatory containers or other infrastructure until a demonstrated requirement exists.

## Local persistence

Use **SQLite** as the default durable store for control-plane state.

Reference state may include:

- `leases`;
- `lease_events`;
- provider observations/status;
- workload manifests/references;
- artifact metadata and hashes;
- resource/cost observations where relevant;
- reconciliation/recovery metadata.

SQLite is authoritative for Colab Infinity lease state. A remote provider, notebook filesystem or Google Drive must not become the source of truth for the lifecycle.

## Python owns the ephemeral worker layer

Python remains the preferred worker language when the value comes from:

- PyTorch / Transformers / model ecosystems;
- numerical/scientific processing;
- data tooling;
- GPU-enabled experimentation;
- rapid workload-specific scripting;
- AI/ML benchmarks and evaluation.

Python is intentionally placed **inside the ephemeral resource**, not required as a permanently resident host runtime.

A lease may therefore create a worker with heavyweight Python dependencies, execute a bounded workload, emit outputs/evidence and disappear. The normal local Anakyklos installation should not pay the runtime/memory/storage cost of those optional compute dependencies while idle.

Python is not mandatory for every workload. A provider may execute Go, Rust, native binaries, shell commands or another explicitly packaged workload when the contract requires it. Python is the default for the AI/data/scientific worker territory, not a universal guest ABI.

## Provider interface

Provider-specific mechanics must stay behind a narrow adapter boundary.

Reference conceptual interface:

```text
Provider
  Probe
  Provision
  Execute / Dispatch
  Collect
  Cancel
  Teardown
```

Exact method names and synchronous/asynchronous semantics remain implementation details until the MVP is specified.

Potential adapters include:

- Google Colab;
- local discrete GPU;
- rented/spot GPU provider;
- temporary cloud CPU/GPU instance;
- future authorized compute backend.

A provider change must not redefine the `ComputeLease` model.

## Workload contract

Workloads should be provider-independent whenever practical.

A workload package should describe at minimum:

- workload/version identifier;
- required resource class;
- timeout/duration limit;
- privacy class;
- declared inputs;
- execution entrypoint or runner contract;
- declared outputs;
- verification/integrity expectations.

Prefer structured JSON manifests for contracts and files/artifacts for large payloads.

Example conceptual manifest:

```json
{
  "workload": "model-benchmark",
  "version": 1,
  "timeout_seconds": 1200,
  "privacy_class": "internal",
  "inputs": ["dataset.jsonl", "config.json"],
  "outputs": ["metrics.json", "result.json"]
}
```

The same logical workload should be runnable on different authorized providers when their resource capabilities satisfy the contract.

## Artifact model

Prefer artifact exchange over treating the remote worker as a persistent filesystem or service.

Typical inputs:

- workload manifest;
- source/archive;
- dataset or minimized fixture;
- model/config references.

Typical outputs:

- result JSON;
- benchmark metrics;
- logs/evidence;
- generated artifacts/archive.

Use cryptographic hashes such as SHA-256 for integrity/provenance where material. Artifact stores/transports may vary by provider; they are not authoritative owners of product state.

## Google Colab adapter direction

Google Colab remains a useful first provider experiment, but its adapter must use authorized mechanisms and must not depend on bypassing platform limits.

The MVP may be semi-manual if Colab does not provide a suitable supported provisioning API. Architectural correctness is preferred over browser automation that attempts to make a volatile free notebook look like a permanent server.

A valid early flow may be:

```text
Go control plane
  → prepare lease/workload package
  → authorized transfer mechanism
  → user/authorized process starts Colab notebook
  → Python worker executes workload
  → result artifacts returned
  → control plane records completion/teardown
```

Google Drive may be used by a Colab adapter as a temporary artifact transport when appropriate. It must not be the authoritative lease database.

## Removed from the core

The following are **not** architectural requirements of the new control plane:

### Playwright / Selenium

Browser automation existed to sustain the former continuous-account/session model. It is not part of the new baseline and must not be reintroduced to circumvent provider access or quota policies.

### Ngrok

Remote public tunneling is not a core requirement. A provider adapter may use a provider-supported network mechanism if a real workload requires it, but the stable architecture does not depend on exposing every worker as an HTTP server.

### FastAPI

FastAPI may be useful inside a specific Python workload, but workers are not required to expose an OpenAI-compatible or generic HTTP API. Artifact/job execution is the preferred baseline for bounded compute.

### Google Drive as runtime state

Drive may transport artifacts for a provider-specific flow. Durable lifecycle state belongs locally in SQLite.

### Account rotation / anti-detection mechanisms

Provider quotas and acceptable-use rules are constraints to respect. Credential/cookie rotation, anti-detection behavior or multiple-account strategies whose purpose is to bypass resource/access limits are outside the product direction.

## Security boundary

Remote compute amplifies resources, not authority.

Workers receive only the minimum inputs, credentials and permissions required by the workload. They must not inherit unrestricted access to local files, Anakyklos databases, production secrets or unrelated capabilities.

If future execution of untrusted packaged workloads requires a stronger portable sandbox boundary, evaluate WebAssembly/WASI before inventing a custom plugin ABI. Rust may be justified for a concrete security-sensitive native enforcement component. Neither is required for the first MVP.

## Technology exclusions for the MVP

Do not introduce by default:

- Zig merely for lower theoretical footprint;
- Rust without a concrete security/safety boundary;
- Mojo without a profiled accelerated-compute bottleneck;
- containers as a mandatory universal abstraction;
- a distributed scheduler;
- Kubernetes;
- Redis/Postgres/message brokers;
- a permanent Python daemon on the host;
- a mandatory HTTP server in every worker.

## Reference architecture

```text
                Megingjörð
             Go control plane
                    │
               ComputeLease
                    │
             Provider interface
          ┌─────────┼─────────┐
          │         │         │
        Colab     Local     Cloud
          │         │         │
          └──── ephemeral worker ────┐
                    │                 │
             Python when AI/data     │ other packaged workloads
                    │                 │
                    └──────┬──────────┘
                           │
                    result/evidence
                           │
                    local SQLite
```

## MVP stack

For the first bounded end-to-end implementation:

```text
Host control plane: Go
Durable state: SQLite
Contract format: JSON
Artifact integrity: SHA-256 where material
First provider: Google Colab experiment
Worker: Python
Heavy AI/data dependencies: worker-only and workload-specific
```

The MVP should prove `request → provision/connect → execute → collect → teardown → reconcile durable lease state` before multi-provider scheduling or advanced autonomy is introduced.
