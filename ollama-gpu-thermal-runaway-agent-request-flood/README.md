---
title: "GPU Thermal Runaway From an Agent Request Flood: Finding the Real Caller and Building Three Layers of Protection"
date: 2026-10-09
featured: false
category: "AI Infrastructure"
tags: ["Ollama", "GPU", "n8n", "Wazuh", "systemd", "Hermes"]
summary: "A GPU-passthrough node ran hot with no one using it. The cause was an agent gateway flooding Ollama with back-to-back requests, triggered by an n8n workflow that fired one webhook per Wazuh alert. Fix was a three-layer safety design: a thermal watchdog, a circuit-breaker proxy, and a hardware power cap."
---

# GPU Thermal Runaway From an Agent Request Flood

**Category:** AI infrastructure / Reliability
**Environment:** Proxmox VE, Debian 13 VM with RTX 4000 SFF Ada (VFIO), Ollama, Hermes agent, n8n, Wazuh
**Date:** September and October 2026
**Status:** Mitigated, with three independent safeguards in place

---

## Summary

The GPU on Node-A overheated twice. The first time (9 September), a retry loop with no backoff in the Jocasta Discord integration hammered Ollama. The second time (October), the GPU sat at 91 degrees C with nobody actively using it. The second cause was the Hermes gateway on a different VM sending continuous back-to-back chat requests to Ollama, driven by an n8n workflow that posted one webhook per Wazuh alert with no batching or throttling. The incident led to a three-layer protection design that was tested with a kill test and an outage simulation.

---

## Impact

- GPU idle temperature returned to a baseline near 67 degrees C (it was 91 degrees C under the flood), and with the gateway stopped the GPU dropped to 2W and 0% utilization.
- Three independent safeguards now exist (thermal watchdog above 88 degrees C, a concurrency limit of 1 behind a proxy, a 45W hardware power cap), and each was tested.
- Two incidents shared one failure class (a retry loop with no backoff, then an unthrottled webhook), so the fix addressed the pattern, not a single caller.

---

## Environment

| Component | Detail |
|-----------|--------|
| GPU host | Tantive-III VM (Node-A), RTX 4000 SFF Ada passed through via VFIO, Ollama in Docker |
| Agent host | Coruscant VM (Node-D), `hermes-gateway.service` and `hermes-dashboard.service` |
| Automation | n8n Command Bridge on Phoenix-Nest, Wazuh alerts via webhook |
| Alert path | Wazuh to n8n to Discord, with an AI enrichment tail through the Hermes webhook (port 8644) |

---

## Symptom

Node-A's GPU was hot while idle by every expectation. Measured at the time:

| Metric | Reading |
|--------|---------|
| GPU utilization | 96% |
| Power | 69W of a 70W cap |
| Temperature | 91 degrees C |
| Ollama connections | 10 parallel |
| Request pattern | The same 5,218-token prompt repeating back-to-back |

---

## Isolation

1. **Look at the caller, not the GPU.** The load was a steady stream of `/v1/chat/completions` requests, so something upstream was generating it.
2. **Find the process.** The requests came from `hermes-gateway.service` on Coruscant (`python -m hermes_cli.main gateway run`), a separate unit from the dashboard.
3. **Prove causality.** With the gateway killed, the GPU fell to idle and Node-A returned to about 67 degrees C.
4. **Find what drove the gateway.** The gateway log showed hundreds of parallel `webhook:alert-enrichment` sessions. The n8n Command Bridge enrichment workflow was posting one request per Wazuh alert to the gateway's webhook port, with no batching or throttle. The session database had also grown to 162MB from stale sessions.
5. **Rule out the other inputs.** The gateway also listens to Discord and Home Assistant, but the log showed the webhook path was the flood, not those two.

---

## Root Cause

An unthrottled alert-to-agent pipeline. Each Wazuh alert produced one webhook, each webhook produced a long-running agent session, and each session produced model requests to a single GPU. Nothing in the chain limited concurrency or detected the thermal consequence. The September incident was the same class of failure from a different source (a Discord retry loop with no backoff).

A detail that cost time: `hermes-gateway.service` has `Restart=always`, so killing the process did nothing. It had to be stopped and disabled through systemd.

---

## Resolution

Immediate: the gateway was stopped and disabled (GPU dropped to 2W, 0% utilization, 72 degrees C), and the Wazuh enrichment workflow in n8n was deactivated so alerts kept reaching Discord through the workflow's direct webhook posts without the AI enrichment tail.

Durable: three independent layers, so no single failure can recreate the incident.

| Layer | Mechanism | Behavior |
|-------|-----------|----------|
| 1. Watchdog | Cron job on Tantive-III | Kills the Ollama inference process above 88 degrees C and reports to an n8n webhook |
| 2. Circuit breaker | aiohttp proxy on Coruscant at `127.0.0.1:11435` | Maximum 1 concurrent inference, 60s queue timeout (503), 5 minute request timeout, 30s stream chunk timeout. The gateway's `base_url` now points at the proxy |
| 3. Hardware cap | `nvidia-powercap.service` | 45W power limit, persistent across boots. The cap did not persist until `nvidia-persistenced` was enabled; the limit was then confirmed at 45W (minimum 30W, maximum 70W) |

One watchdog bug found along the way: Ollama's inference process had been renamed, so the `pkill` pattern for `ollama runner` matched nothing. The pattern was changed to `llama-server`.

---

## Verification

- **Watchdog kill test:** a model was loaded (5,511 MiB of VRAM), the watchdog's kill was run through the container, VRAM fell to 187 MiB, and the n8n webhook returned HTTP 200.
- **Outage simulation:** outbound port 443 was blocked on Coruscant with `iptables` and a webhook request was sent to the gateway on port 8644. The GPU stayed flat (2 MiB, 0%, 15W).

---

## Remaining Work

The enrichment workflow needs batching or throttling before the AI enrichment tail is re-enabled.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**During the flood (9 October, 23:08, on the GPU VM):**

```
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util |
|100%   91C    P2             69W /   70W |   15220MiB /  20475MiB |     99%  |
|    0   N/A  N/A    524647      C   /usr/lib/ollama/llama-server     15214MiB |
```

**Ollama request log and connections (agent host and GPU VM):**

```
10 simultaneous ESTABLISHED connections from the agent VM to port 11434 (ss -tnp)
Back-to-back POST /v1/chat/completions, ~5,218 tokens in, ~588 out, ~33 s each
Prompt cache similarity 0.997 between consecutive requests
hermes dashboard: 2m53s CPU over 20 hours (harmless)
hermes gateway run: 60 minutes CPU (the source)
```

**Isolation test (7 October):** powering off the GPU VM dropped Node-A from 88 degrees C to 67 degrees C, which separated the GPU from the chassis and CPU.

**After stopping the gateway (23:12):**

```
| 80%   72C    P0              2W /   70W |       1MiB /  20475MiB |      0%  |
|  No running processes found                                                  |
```

**Watchdog kill test:** `qwen3:8b` loaded (5,511 MiB), `docker exec ollama pkill -f llama-server`, VRAM fell to 187 MiB at 0% utilization and 15W, webhook to n8n returned HTTP 200. (`pkill` worked even though `command -v pkill` printed nothing inside the container, a PATH quirk of `sh`.)

**Outage simulation:** outbound 443 blocked on the agent VM, a request sent to the gateway webhook on port 8644, GPU stayed flat at 2 MiB, 0%, 15W with no processes.

---

## Verify It Yourself

These are the commands that check the claims above. Run them on the relevant host.

```bash
# GPU state, once per second (run on the GPU VM)
nvidia-smi --query-gpu=temperature.gpu,power.draw,utilization.gpu,memory.used --format=csv -l 1

# How many connections are hitting Ollama right now
ss -tn state established '( sport = :11434 )' | wc -l

# Will systemd bring the gateway back? (run on the agent VM)
systemctl show hermes-gateway -p ActiveState -p Restart
sudo systemctl disable --now hermes-gateway

# Circuit breaker health and power cap persistence
curl -s http://127.0.0.1:11435/health
systemctl status nvidia-powercap.service
```

---

## What I Would Change

- Add batching or a rate limit to the n8n enrichment workflow before re-enabling it.
- Alert when the watchdog fires, so a thermal event is a page and not a silent kill.
- Chart GPU temperature and power in the Grafana AI Ops dashboard with a warning threshold below the watchdog's 88 degrees C.

---

## Takeaways

- **Trace load to its caller.** A hot GPU with no users means find the client, not the card.
- **`Restart=always` means kill is not stop.** Use `systemctl stop` and `disable`.
- **Event-per-webhook pipelines need a concurrency limit.** One alert should never equal one unbounded model session.
- **Layer the safeguards.** Software limits, a proxy, and a hardware cap fail independently.
- **Test the safeguard, not just the config.** The kill test and the outage simulation are what made the design credible.

---

## Related

| Repo | Description |
|------|-------------|
| [Proxmox VFIO Lockup Forensics](../proxmox-vfio-lockup-forensics/) | The earlier GPU passthrough incident on the same node |
