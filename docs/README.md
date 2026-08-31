# Documentation status

This directory contains both historical Colab Infinity design material and the current architectural direction.

## Direction

- [`11_ephemeral_compute_direction.md`](11_ephemeral_compute_direction.md) — current product direction: `Colab Infinity · Megingjörð` as Anakyklos's ephemeral/burst compute plane.

The cross-repository source of truth is `Anakyklos/architecture/rfcs/0003-ephemeral-burst-compute.md`.

## Legacy

Documents `01` through `10` describe the former product direction centered on continuous Google Colab LLM inference and provider-specific infrastructure:

- `01_project_charter.md`
- `02_srs.md`
- `03_architecture.md`
- `04_api_spec.md`
- `05_setup_guide.md`
- `06_runbook.md`
- `07_test_plan.md`
- `08_risk_analysis.md`
- `09_integration_guide.md`
- `10_deploy_checklist.md`

They are retained for design history and selective extraction of useful primitives. Their former `Approved` labels describe the historical design state and do not make them authoritative over the current Direction.

Do not implement continuous-uptime, provider-quota workarounds, permanent remote inference, or provider-specific coupling merely to preserve those documents.

When a legacy primitive is reused, re-evaluate it against the `ComputeLease`, ephemeral-worker, privacy, provider-policy and module-autonomy boundaries before adoption.
