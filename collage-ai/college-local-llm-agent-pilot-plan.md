# College Local LLM + Agent Pilot — Quick Win to Advanced

Audience: a teacher who understands AI concepts but not server/network admin, with a college hardware/IT team available to consult. Goal: students can ask basic AI questions from a LAN desktop, with phone access as a stretch goal.

---

## 0. Read this first — the real risk isn't the model

Software stack choice is the easy part. The thing that actually kills these pilots is **network policy you don't control**:

- Most institutional Wi-Fi runs **AP/client isolation** (devices on the same SSID can't reach each other). This is *the* most common reason "it worked on my laptop but not on my phone."
- Locked-down desktops often don't have local admin rights to install Docker or open firewall ports.
- IT may not allow new inbound ports on the campus network without a ticket.

**Action before touching software:** send the IT question script in Section 6 to the hardware team and get answers on isolation, admin rights, and firewall policy. Everything below assumes you'll have at least one machine with admin rights.

---

## 1. Tier map

| Tier | What you build | Effort | Result |
|---|---|---|---|
| 0 — Quick win, not "owned" | Free cloud API (Groq, Google AI Studio free tier) in a browser tab | 10 min | Validate the teaching use case with zero install, before asking for hardware. Skip if self-hosted is a hard requirement from day one. |
| 1 — Target pilot | Ollama (model engine) + Open WebUI (chat UI) on one desktop, LAN only | 1–2 hrs | Desktop-based basic Q&A for a small group |
| 2 — Phone access | Add Tailscale, or a dedicated pilot router | +30 min | Phone access when same-Wi-Fi access fails |
| 3 — Agent-lite | Open WebUI's built-in document RAG (upload syllabus/notes) | +1 hr | Answers grounded in actual course material |
| 4 — Real agent | n8n + Ollama node (tool calling, web search, scheduling) | +half day | Multi-step agent behavior, not just chat |
| 5 — Production | Dedicated GPU box, vLLM for serving, bigger model, optional fine-tuning, SSO | days–weeks | Many concurrent users, classroom-scale |

Start at Tier 1. Do not skip ahead — each later tier assumes the one before it is stable.

---

## 2. Architecture — Tier 1

```
                    College LAN (Wi-Fi + Ethernet)
   ┌───────────────────────────────────────────────────────────┐
   │                                                             │
   │   ┌─────────────────────────┐        ┌──────────────────┐ │
   │   │   Desktop (the "server") │        │ Student desktop  │ │
   │   │  ┌─────────┐ ┌─────────┐ │        │  browser → IP:3000│ │
   │   │  │ Ollama  │←│OpenWebUI│ │◄───────│                  │ │
   │   │  │(model)  │ │ (chat   │ │  LAN   └──────────────────┘ │
   │   │  │port 11434│ │  UI)   │ │  HTTP                       │
   │   │  └─────────┘ │port 3000│ │        ┌──────────────────┐ │
   │   └──────────────┴─────────┘ │◄───────│ Phone (same WiFi)│ │
   │                               │        │ browser → IP:3000│ │
   │                               │        └──────────────────┘ │
   └───────────────────────────────────────────────────────────┘
```

One desktop runs both pieces. Everyone else just needs a browser — no installs on student devices.

---

## 3. Step-by-step build (Tier 1)

1. **Pick the host machine.** Spare desktop, no GPU required to start. Confirm with IT: ≥8GB RAM minimum, 16GB preferred, and that you have local admin rights on it.
2. **Install Ollama** (Windows/macOS/Linux installer, or `curl -fsSL https://ollama.com/install.sh | sh` on Linux/macOS).
3. **Pull a model sized to the hardware** (see Section 4 for the pick): `ollama pull <model-name>`.
4. **Verify it works locally first:** `ollama run <model-name>` and ask it a question in the terminal. Don't move to the network step until this works.
5. **Install Open WebUI via Docker** (fastest path):
   ```
   docker run -d \
     -p 3000:8080 \
     --add-host=host.docker.internal:host-gateway \
     -v open-webui:/app/backend/data \
     --name open-webui \
     --restart always \
     ghcr.io/open-webui/open-webui:main
   ```
6. **Make Ollama reachable from the LAN, not just localhost.** Set `OLLAMA_HOST=0.0.0.0` (environment variable, or edit the systemd service on Linux) and restart Ollama.
7. **Open firewall ports 11434 (Ollama) and 3000 (Open WebUI) to the local subnet only** — not the whole internet. On Windows this is usually a one-click "Allow on private network" prompt; on Linux, `ufw allow from <subnet> to any port 3000`.
8. **Find the host's LAN IP** (`ipconfig` / `ip addr`) and browse to `http://localhost:3000` on the host machine itself — create the admin account on first load.
9. **Test from a second desktop** on the same LAN: `http://<host-LAN-IP>:3000`.
10. **Test from a phone** on the same Wi-Fi — see Section 5's decision tree if this fails.
11. **In Open WebUI admin panel:** disable open self-signup, create individual accounts for the pilot group, set default role to "pending" so new signups need approval.
12. **Run the pilot** with 5–10 users asking basic questions only. Don't add RAG or agents yet — get this layer boring and stable first.

---

## 4. Model pick by hardware

CPU-only is the realistic default for a college desktop — no GPU needed to start, just expect slower replies (a few tokens/second rather than instant).

| Hardware tier | Model | Why |
|---|---|---|
| 8GB RAM, no GPU | Phi-4-mini (3.8B), Q4_K_M quant | Currently the most CPU-friendly capable option; 128K context but keep actual usage short on weak hardware |
| 16GB RAM, no/weak GPU | Gemma 4 12B, or Qwen3 7B–8B (Q4_K_M) | Practical sweet spot for a Q&A pilot |
| 8–12GB VRAM (dedicated GPU) | Qwen3 14B, or Llama 3.3 8B | If the hardware team can free up even a mid-range GPU box |
| 24GB+ VRAM | Qwen3 30B-class, or Gemma 4 26B MoE | Don't reach for this until Tier 3+ actually needs it — bigger isn't better for a basic-Q&A pilot |

Rule of thumb: pick the *smallest* model that answers your test questions well. A bigger model that takes 30 seconds per reply will kill pilot adoption faster than a smaller, faster, slightly less polished one.

---

## 5. Phone access — decision tree

```
Is the phone on the same campus Wi-Fi as the desktop?
 │
 ├─ No (mobile data, different network)
 │     → Use Tailscale (free tier, up to 6 user accounts, unlimited
 │       devices per account) to put both devices on the same
 │       private network. Or skip phone access for this pilot.
 │
 └─ Yes
       │
       ├─ Try http://<desktop-LAN-IP>:3000 in the phone's browser
       │
       ├─ It works → done, no extra tools needed
       │
       └─ It times out / connection refused
             → AP/client isolation is almost certainly on
             │
             ├─ Ask IT to disable isolation for this device or
             │   VLAN/SSID → best long-term fix, do this even if
             │   you also use the workaround below
             │
             └─ Can't get an IT change in time
                   → Install Tailscale on desktop + phone (this
                     tunnels around isolation, still free for ≤6
                     user accounts), or buy a cheap dedicated
                     router (~₹1,500) just for the pilot, bypassing
                     the campus network entirely
```

Tailscale free-plan note: 6 *user accounts*, unlimited *devices per account*. Fine for instructor + a few pilot students/TAs logging in individually. Not a real solution for a whole classroom — that's a Tier 5 problem (proper reverse proxy + HTTPS cert, owned by IT), not a Tier 1–2 one.

---

## 6. Questions for the college hardware/IT team

Send this as a short list, not a vague "can you help with AI setup":

1. Is there a spare desktop or mini-PC we can dedicate to this, and do we have local admin rights on it?
2. Is client/AP isolation enabled on the Wi-Fi network the students will use? Can it be disabled for one SSID or VLAN for this pilot?
3. Can we open two ports (11434 and 3000, or equivalents) for LAN-only traffic on that machine?
4. Is Docker installation allowed on college machines, or is there a policy against it?
5. Does any lab already have an idle GPU workstation (data science / graphics lab) we could borrow for Tier 4–5 later?
6. Who do we contact if the pilot needs to move from "one desktop" to "a server with a fixed IP" later?

---

## 7. Stuck points and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Model dropdown empty in Open WebUI | Open WebUI container can't reach Ollama | Set `OLLAMA_BASE_URL=http://host.docker.internal:11434` (Linux/Windows) and confirm with `docker exec open-webui curl -s http://host.docker.internal:11434` |
| Works on the host desktop, fails from another LAN device | Ollama still bound to localhost, or firewall blocking the port | Set `OLLAMA_HOST=0.0.0.0`, restart Ollama, open the firewall rule for the subnet, not just the app |
| Phone can't reach it even on the same Wi-Fi | AP/client isolation | See Section 5 decision tree |
| Replies are painfully slow | Model too large for CPU-only hardware | Drop to a smaller parameter count (3–4B class) or a more aggressive quant (Q4_K_M) |
| Server bogs down with 3+ people asking at once | No GPU = requests processed mostly serially | Cap concurrent sessions in the pilot group, or move the model to a GPU machine before scaling past ~5 simultaneous users |
| Need more than chat — actions, lookups, scheduling | Open WebUI alone is chat + RAG, not a task-executing agent | Move to Tier 4: n8n with an Ollama node, or use Open WebUI's native "Tools" function-calling with a tool-calling-capable model (Qwen3, Llama 3.3) |
| IT won't allow Docker/port changes on the managed desktop | No admin rights on locked-down machines | Run the pilot on a personal laptop first to demonstrate value, then request a dedicated machine with admin rights once there's something to show |

---

## 8. Tier 2–5: scaling beyond the pilot

- **Tier 2 (RAG / "agent-lite"):** Open WebUI has a built-in RAG pipeline (ChromaDB backend, no separate vector DB to manage). Upload syllabus PDFs, lecture notes, FAQs. Pull an embedding model first (`ollama pull nomic-embed-text`). This alone covers most "explain this from our course material" requests without needing a real agent framework.
- **Tier 3 (real agent):** n8n + an Ollama node gives actual tool-calling behavior — web search, calculator, calendar lookups, multi-step workflows — reusing the n8n experience already on hand rather than building a fresh LangChain pipeline from scratch.
- **Tier 4 (more concurrent users):** swap Ollama for vLLM on a machine with a real GPU once user count or response-time complaints justify it. Ollama is fine for a pilot of a handful of users; it is not built for serving a full classroom concurrently.
- **Tier 5 (production):** dedicated GPU server, bigger model (14B–30B class), optional LoRA fine-tuning on college-specific Q&A pairs, SSO/role-based access if IT wants this folded into the official network, multi-agent orchestration only if a single agent genuinely can't cover the use case.

Don't jump to Tier 5 because it sounds more impressive — every tier above 1 should be justified by a specific complaint or limitation hit during the pilot, not adopted preemptively.

---

## 9. Caveat on the model/tooling specifics above

The open-model landscape (Qwen, Gemma, Llama, GLM, etc.) and tool versions (Ollama, Open WebUI) ship new releases roughly monthly. The model picks and exact commands above reflect what's current now, but verify model names and exact VRAM/RAM figures against the Ollama model library before pulling, since names and recommended quantizations shift between releases.
