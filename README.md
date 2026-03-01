# IPMI Fan Control Helm Chart

Generic CronJob-based IPMI fan control for any ipmitool-compatible BMC (e.g. Supermicro, Dell iDRAC). **No vendor-specific logic in the chart** — you supply the exact `command` and `env` per job.

## Chart contents

- **CronJob(s):** Each entry in `.Values.jobs` becomes one CronJob running `ipmitool` (or your command) on a schedule.
- **Secrets:** 1Password item (onepassworditem subchart); create a Secret per job (e.g. `ipmi`, `idrac`) with keys such as `username`, `password`. Reference in `jobs[].env` via `valueFrom.secretKeyRef`.
- **Reloader** (optional): Annotate CronJobs with `secret.reloader.stakater.com/reload` so Stakater Reloader restarts when credentials change.

## Prerequisites

- **1Password Connect** (or equivalent) if you use `onepassworditem.enabled: true` to sync credentials. The chart depends on [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm); secrets are created in the release namespace (`.Release.Namespace`).
- **Reloader** (optional): if you set `secret.reloader.stakater.com/reload` on the CronJob, Stakater Reloader will roll the job when the secret changes.

## Design

- **`.Values.jobs`** is a list. Each entry becomes one CronJob with:
  - `name` — CronJob name
  - `command` — container command (e.g. shell + ipmitool invocations)
  - `env` — env vars and/or `valueFrom.secretKeyRef` for credentials
  - `schedule` — cron schedule
  - Optional: `image`, `annotations`, `securityContext`, `resources`, etc.
- **Credentials:** Reference a Kubernetes Secret by name in `env` (e.g. `secretKeyRef.name: ipmi`). Sync that secret from 1Password via **onepassworditem**: set `onepassworditem.enabled: true` and `onepassworditem.items[]` with `name` matching the secret name used in the job. Each item uses `item` (1Password path), `name` (Secret name), `type` (e.g. Opaque). Namespace is taken from the release.
- Add multiple entries in `jobs` to run different commands or schedules (e.g. one per server, or separate Supermicro vs iDRAC jobs).

## Values (1Password)

| Key | Description |
|-----|-------------|
| `onepassworditem.enabled` | If `true` (default), the subchart creates OnePasswordItem resources; secrets are synced to the release namespace. Set `false` if you supply secrets another way. |
| `onepassworditem.items` | List of `{ item, name, type }`. `name` must match the Secret name referenced in `jobs[].env[].valueFrom.secretKeyRef.name`. |

## Examples (your values)

**Supermicro-style** (single host): one job, `command` that runs ipmitool with `-H ${IPMI_HOST}` and raw `0x30 0x70 0x66 ...`; env with `IPMI_HOST`, `FAN_SPEED`, and secretRef for `IPMI_USER` / `IPMI_PASSWORD`.

**iDRAC-style** (one or more hosts): one job, `command` that loops over `SERVERS` and runs ipmitool `-I lanplus` with raw `0x30 0x30 0x01 0x00` and `0x30 0x30 0x02 0xff ${FAN_SPEED}`; env with `SERVERS` (space-separated), `IDRAC_FAN_SPEED`, and secretRef for `IDRAC_USER` / `IDRAC_PASSWORD`.

**Multiple servers**: multiple entries in `jobs`, each with its own name, command, env (different IP or secret), and optional schedule.

## Install

**From this repo:**

```bash
helm dependency update
helm install ipmi-fan-control . -f my-values.yaml -n <your-namespace> --create-namespace
```

**From Helm repo (expectedbehaviors):**

```bash
helm repo add expectedbehaviors https://expectedbehaviors.github.io/ipmi-fan-control
helm install ipmi-fan-control expectedbehaviors/ipmi-fan-control -f my-values.yaml -n <your-namespace> --create-namespace
```

## Key values (summary)

| Area | Where in values | What to set |
|------|-----------------|-------------|
| IPMI hosts / command | `jobs[].command` | Shell script with `ipmitool` calls; use env for `IPMI_HOST`, `FAN_SPEED`, and secretRef for credentials. |
| Schedule | `jobs[].schedule` | Cron expression (e.g. `*/5 * * * *`). |
| 1Password items | `onepassworditem.items` | List of `{ item, name, type }`; `name` must match `secretKeyRef.name` in job env. |

## Render & validation

```bash
helm dependency update . && helm template ipmi-fan-control . -f my-values.yaml -n <your-namespace>
```

## Argo CD

Application in your chosen project (e.g. `external-services`), path `.`.

## Next steps

1. Ensure 1Password items exist and Connect has access.
2. Sync in Argo CD; verify CronJob runs and secrets are populated.
3. Adjust schedule and command per BMC (Supermicro, iDRAC, etc.).

## Template filenames

This chart uses **cronJobs.yaml** (camelCase plural) for CronJob resources.
