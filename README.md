# Colab Infinity

> **Temporary compute amplification for Anakyklos.**

**Codename:** `Megingjörð`  
**Class:** `Mythica`  
**Role:** Ephemeral / Burst Compute Plane  
**Status:** Direction — architectural repositioning adopted; implementation does not yet satisfy the new contract.

Colab Infinity is being repositioned from a Google-Colab-specific continuous LLM service into the **ephemeral compute plane of Anakyklos**.

Its purpose is simple: when a bounded workload genuinely needs more CPU, GPU, memory or parallel capacity than the local machine can practically provide, Anakyklos may acquire a temporary compute lease, execute the workload, retrieve the declared outputs/evidence, and release the resource.

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

The system becomes temporarily stronger. It does **not** become permanently dependent on remote infrastructure.

## Architectural source of truth

The current direction is defined by:

- [`docs/11_ephemeral_compute_direction.md`](docs/11_ephemeral_compute_direction.md)
- `Anakyklos/architecture/rfcs/0003-ephemeral-burst-compute.md`
- `Anakyklos/architecture/SYSTEM-MAP.md`
- `Anakyklos/architecture/CODENAMES.md`

## What Colab Infinity owns

Target responsibilities include:

- accepting bounded compute requirements;
- selecting or using an authorized provider adapter;
- provisioning a temporary worker;
- enforcing duration/resource/cost/privacy constraints;
- moving only the required inputs to the worker;
- executing a declared workload;
- retrieving declared outputs and evidence;
- tearing the worker down;
- keeping a durable local record of the lease lifecycle.

## What it does not own

Colab Infinity is not:

- an always-on LLM server;
- a second Ouroboros;
- a coding agent competing with Runstead;
- a capability foundry competing with Cadinho;
- a universal remote shell;
- an authoritative database;
- a place for long-lived Katherine/LifeOS/Tecer/Ouroboros state;
- permission authority for the rest of Anakyklos.

**Compute amplification changes resources, not authority.**

## Core abstraction: `ComputeLease`

The new architecture centers on a bounded lease rather than a provider-specific notebook/session.

Reference shape:

```text
ComputeLease
  id
  provider
  resource_class
  requested_at
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
requested
  → provisioning
  → ready
  → active
  → draining
  → completed
  → destroyed
```

A lease must be disposable and explicitly bounded.

## Google Colab is a provider, not the product

The project originated around Google Colab and GPU-backed notebook sessions. That remains useful as an initial provider experiment, but the stable product boundary must not depend on Colab-specific behavior.

Potential future providers may include:

- Google Colab;
- local discrete GPU;
- rented or spot GPU infrastructure;
- temporary cloud CPU/GPU instances;
- other authorized compute backends.

Provider adapters may change independently without redefining the `ComputeLease` contract.

## Workloads

Burst compute is broader than LLM inference.

Potential justified consumers include:

### Cadinho

Candidate/model/tool trials and reproducible benchmarks that temporarily exceed local compute.

### Runstead

Bounded builds, test suites, benchmarks, static analysis or other software workloads when remote execution is safe and reproducible.

### Ouroboros

Ouroboros may request an authorized compute boost when local execution is insufficient, but it does not own provider lifecycle or credentials.

### Other modules

Scientific/data processing, parallel work or future ML workloads may use leases when privacy, cost and policy permit.

Remote execution should not be used merely because it exists. Local execution remains preferred when it is sufficient, safer or cheaper.

## Ephemeral worker invariant

Remote workers are disposable execution environments.

They must not become authoritative owners of:

- Katherine memory;
- LifeOS facts;
- Tecer health/wellness databases;
- Ouroboros Mission state;
- Cadinho authoritative candidate/evidence state;
- Capability Registry state;
- long-lived production secrets.

Durable state stays with its owning module. A remote worker may vanish without corrupting Anakyklos.

## Privacy

A compute request should carry enough classification for policy to determine whether external execution is acceptable, for example:

```text
public
internal
private
sensitive
```

Sensitive workloads may be prohibited from external providers, require explicit approval, or require minimized/sanitized inputs.

## Provider policy

Provider limits and acceptable-use rules are constraints, not obstacles to evade.

Colab Infinity must use provider resources only through authorized mechanisms. When a provider cannot supply a lease, valid outcomes include another authorized provider, local fallback, queue/wait, a reduced workload, or a clear failure/degradation.

## MVP direction

The first implementation should prove one bounded end-to-end lease:

```text
compute request
  → one authorized provider
  → provision one worker
  → execute one reproducible workload
  → retrieve declared output/evidence
  → teardown
  → persist local lease record
```

Do **not** start with high availability, a provider swarm, permanent remote inference or complex autonomous scheduling.

## Legacy documentation

The historical documents `docs/01_*` through `docs/10_*` were written for the previous direction centered on continuous zero-cost LLM inference and Google-Colab-specific infrastructure.

They are retained as **Legacy** design material and implementation archaeology. They do not override the current Direction document or the central Anakyklos architecture.

Useful primitives may still be extracted selectively if they satisfy the new contract. Existing mechanisms should not be preserved merely because they were previously documented.

## Identity: Megingjörð

In Norse mythology, Megingjörð is Thor's belt of strength. The metaphor fits the product because the belt does not replace the bearer; it **temporarily amplifies existing strength**.

The desktop mark should therefore suggest a compact belt/ring or clasp containing controlled energy, rather than using Google Colab, GPU or generic cloud branding.

## License

[MIT License](LICENSE) — where applicable to repository contents carrying that license.
