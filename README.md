# IPMI Fan Control Helm Chart

Generic CronJob-based IPMI fan control for any ipmitool-compatible BMC (e.g. Supermicro, Dell iDRAC). **No vendor-specific logic in the chart** — you supply the exact `command` and `env` per job.

## Design

- **`.Values.jobs`** is a list. Each entry becomes one CronJob with:
  - `name` — CronJob name
  - `command` — container command (e.g. shell + ipmitool invocations)
  - `env` — env vars and/or `valueFrom.secretKeyRef` for credentials
  - `schedule` — cron schedule
  - Optional: `image`, `annotations`, `securityContext`, `resources`, etc.
- Credentials: reference a Kubernetes Secret by name in `env`; sync that secret from 1Password via **onepassworditem** (set `onepassworditem.secrets.external-services[].name` to match the secret name used in the job).
- Add multiple list entries to run different commands or schedules (e.g. one per server, or separate Supermicro vs iDRAC jobs with different ipmitool raw commands).

## Examples (your values)

**Supermicro-style** (single host): one job, `command` that runs ipmitool with `-H ${IPMI_HOST}` and raw `0x30 0x70 0x66 ...`; env with `IPMI_HOST`, `FAN_SPEED`, and secretRef for `IPMI_USER` / `IPMI_PASSWORD`.

**iDRAC-style** (one or more hosts): one job, `command` that loops over `SERVERS` and runs ipmitool `-I lanplus` with raw `0x30 0x30 0x01 0x00` and `0x30 0x30 0x02 0xff ${FAN_SPEED}`; env with `SERVERS` (space-separated), `IDRAC_FAN_SPEED`, and secretRef for `IDRAC_USER` / `IDRAC_PASSWORD`.

**Multiple servers**: multiple entries in `jobs`, each with its own name, command, env (different IP or secret), and optional schedule.

## Install

```bash
helm dependency update
helm install ipmi-fan-control . -f my-values.yaml -n external-services --create-namespace
```
