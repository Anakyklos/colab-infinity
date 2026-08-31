# Colab Infinity — Ephemeral Compute Direction

**Status:** Direction

**Date:** 2026-08-31

**Architectural source of truth:** `Anakyklos/architecture/rfcs/0003-ephemeral-burst-compute.md`

## Identity

- Product: **Colab Infinity**
- Codename: **Megingjörð**
- Class: **Mythica**
- Role: **Ephemeral / Burst Compute Plane**

Colab Infinity gives Anakyklos temporary access to stronger compute when a bounded workload exceeds the practical local baseline.

It is not intended to be an always-on free server, permanent inference backend, universal remote executor or authoritative state owner.

## Product thesis

```text
normal local operation
      ↓
workload exceeds practical local capacity
      ↓
authorized ComputeLease
      ↓
temporary stronger compute
      ↓
result / evidence returned
      ↓
worker destroyed
      ↓
normal local operation
```

The resource amplification is temporary. Authority is not amplified.

## Provider independence

Google Colab is the first provider/backend candidate because the project began there. It does not define the product boundary.

Future authorized providers may include local discrete GPUs, Google Colab, spot/rented GPUs, temporary cloud CPU/GPU instances, and other bounded compute providers.

The stable contract should describe the compute requirement and lease lifecycle rather than expose provider-specific session mechanics to consumers.

## Core abstraction

```text
ComputeLease
  id
  provider
  resource_class
  started_at
  expires_at
  max_cost
  privacy_class
  permissions
  input_refs
  output_contract
  state
```

Reference lifecycle:

```text
requested → provisioning → ready → active → draining → completed → destroyed
```

Every lease is bounded by duration, resource, cost and authority constraints.

## Workload examples

- Cadinho model/tool/candidate benchmarks;
- bounded LLM inference;
- scientific/data workloads;
- compute-heavy builds/tests/benchmarks when remote execution is semantically safe;
- parallel processing;
- future ML experimentation compatible with provider and privacy constraints.

Remote execution is not preferred when local execution is cheaper, safer or sufficiently fast.

## Ephemeral-worker invariant

Remote workers are disposable.

They must not become authoritative owners of Katherine memory, LifeOS facts, Tecer health/wellness data, Ouroboros Mission state, Cadinho authoritative lineage/evidence state, Capability Registry state or long-lived production secrets.

Durable inputs are referenced/copied in. Declared outputs/evidence are copied out. The worker can then disappear without corrupting Anakyklos state.

## Privacy

Compute requests should carry a privacy class such as `public`, `internal`, `private` or `sensitive`.

Policy decides whether a provider may receive the workload. Sensitive workloads may be local-only, require explicit approval, or require sanitized/minimized data.

Compute availability never overrides the privacy rules of the owning module.

## Provider-policy invariant

Provider limits and acceptable-use rules are architectural constraints. The implementation must use providers only through authorized mechanisms and must not depend on bypassing their quotas or access policies.

If a provider is unavailable or exhausted, valid behavior is to use another authorized provider, queue/wait, fall back locally, reduce the workload, or fail/degrade clearly.

## Relationship with Cadinho

Cadinho is expected to be a major consumer of burst compute. Cadinho owns the experimental question, trial contract, comparison and evidence interpretation. Colab Infinity only supplies a bounded compute lease.

## Relationship with Ouroboros

Ouroboros may request burst compute when policy allows it and local execution is insufficient. It does not own provider credentials or lifecycle internals.

Colab Infinity being absent must not disable unrelated missions.

## Relationship with Runstead

Runstead may use bounded compute leases for appropriate software-engineering workloads. Colab Infinity does not become a coding agent and does not inherit Runstead's technical verification authority.

## Stack direction

The implementation direction is intentionally split by responsibility:

- **Go** owns the persistent/local control plane, `ComputeLease` lifecycle, provider adapters, cancellation, budgets, reconciliation and CLI/IPC surfaces;
- **SQLite** is the authoritative durable local store for lease lifecycle state;
- **Python** is the preferred ephemeral worker language for AI/ML, data, scientific and GPU-heavy workloads;
- **JSON manifests + artifacts** define workload input/output contracts and keep workloads provider-independent when practical;
- heavyweight Python/AI dependencies should exist only inside the worker/workload environment when required.

Playwright/Selenium, Ngrok, FastAPI as a mandatory worker server, Google Drive as authoritative state, and account/session rotation are **not** part of the new core architecture.

See [`12_stack_direction.md`](12_stack_direction.md) and `Anakyklos/architecture/adr/0002-colab-infinity-stack.md` for the detailed decision and rationale.

## Legacy direction

Documents `01`–`10` describe the historical design centered on continuous free LLM inference and Google-Colab-specific infrastructure.

Those documents are retained as historical design material unless separately rewritten, but they do **not** override this Direction document or the central Anakyklos architecture.

Potentially reusable primitives include environment bootstrap, bounded inference setup, health checks, output transfer and teardown experiments. Continuous-uptime behavior is not a future requirement.

## MVP

The first new implementation slice should prove exactly one honest lease:

```text
request
  → one authorized provider
  → provision
  → run one reproducible workload
  → collect output/evidence
  → teardown
  → persist local lease record
```

Do not begin with high availability, provider swarms or a permanent remote inference service.
