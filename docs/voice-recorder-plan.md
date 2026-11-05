# Always-on daytime voice recorder: evaluation and plan

Prepared 2026-11-04 (SAST, UTC+2). Research only: nothing was installed, pushed or changed. Repos were shallow-cloned read-only into
`/workspace/voice-recorder/src/` on the box for reading. Paths below are relative to each repo.

Updated after Barend's steering: **Homley = ingest + storage + light work; BMain = GPU box for STT and the large Q&A model;
raw audio kept indefinitely by default (configurable).**

Updated again 2026-11-04 with Barend's community research notes (new §9 "Community lessons"). Each claim about Nytka was
re-checked against the repo. **The recommendation stays Nytka.**

---

## 1. Recommendation: Nytka (nytka-app/android + nytka-app/server)

Pick **Nytka**, using the `oss` APK on the S25 with **Phone microphone** mode now and the Omi pendant later. Run the server on Homley
and point its speech-to-text and LLM URLs at BMain.

Why Nytka beats Chronicle *for this design* (evidence in section 2):

1. **The phone side works the way we need.** Nytka is a native Kotlin app. It writes every Opus frame to a Room queue on the phone and
   deletes it **only after the server answers 200/202** (android `README.md`, "How it works"). Chronicle's app is a React Native/Expo
   **live WebSocket streamer** (`app/README.md`, "Backend Configuration": `/ws/audio`). It has no on-phone queue: no hits for
   offline/queue/persist in `app/src/protocol/audioV2Socket.ts` or `useAudioStreamer.ts`. That fails "batched Wi-Fi uploads" and
   "delete once Homley confirms" at the design level.
2. **Installable without building.** Nytka ships a signed APK on every GitHub release (latest `v0.26.3`, 2026-10-27), and Obtainium
   can install and update it (android `README.md`, "Install without Google Play"). Chronicle's last Android APK
   (`android-v1.0.0-20260219-170414`, `friend-lite-android.apk`) predates its v0.3 "Audio V2" protocol, and its v0.3.x releases
   ship no APK (only TestFlight for iOS). We would have to build it with Expo/Android SDK.
3. **Remote STT and LLM are first-class.** Nytka's server sends speech batches (16 kHz mono WAV) to **any OpenAI-compatible
   `/v1/audio/transcriptions` URL** (`Nytka__Stt__Url`) and chat to any OpenAI-compatible base URL (`Nytka__Llm__BaseUrl`, Ollama
   documented as `http://<host>:11434/v1`) (server `docs/reference/configuration.md` lines 25-46, 186-199, 264). That is exactly
   the Homley→BMain split. It also already retries a failed STT batch for about 31 hours and keeps the audio (config.md "Failed
   batches"), which is the "BMain is asleep" backlog.
4. **Omi later needs no new stack.** Omi is Nytka's primary recorder, including offline sync from the pendant's storage on firmware
   3.0.20+ (android `README.md`, "Recorders").
5. **Smaller server footprint.** One .NET server plus Postgres (server `docker-compose.yml`: `postgres`, `server`, and optional
   `stt-local`/`llm-local` under profile `local`). Chronicle needs FastAPI backend + workers + cron + MongoDB + Redis + Caddy +
   Syncthing + Tailscale containers (`backend/docker-compose.yml`) and is Docker/K8s-oriented.
6. **Retention, search, Ask and MCP exist today.** `Nytka__Audio__RetentionDays` (0-3650). Full-text search, plus vector search with
   pgvector and embeddings. Ask answers with numbered sources. MCP has 22 tools (19 read, 3 write) at `POST /mcp` (server `README.md`;
   `docs/reference/api.md` line 135).

**Caveats to accept with Nytka (important):**
- **Young, single maintainer.** Both repos were created 2026-09-28/29. Essentially one author (`Egoushka`: 174 server commits;
  the rest are bots). The server README badge says **"Parked. It works; I am not changing it now and may come back."** The Android
  app says "Active", with commits on 2026-11-03. Expect to **fork and own the server patches** we need.
- **The server does not yet support the app's Phone microphone mode.** The app shows the switch only when `/api/v1/info` lists the
  `phone-mic` feature (android `docs/reference/device.md`, "Phone microphone"; `core/.../ApiModels.kt:50 FEATURE_PHONE_MIC`). The
  server source has no `phone-mic` string and no handling of the `X-Nytka-Source` header (searched `src/`). So phase 1 needs a
  small server patch. This is the biggest custom item.
- Ukrainian/English-centric (search dictionaries, UI). Afrikaans works through the `simple` text-search config (no stemming;
  `db/migrations/0018_people_search.sql`). Whisper large-v3 does transcribe Afrikaans.

Why not Chronicle (fairly): it is MIT, more contributors (AnkushMalaker 831, thestumonkey 188, others) and has a Home Assistant plugin
(`plugins/homeassistant/`), Parakeet/faster-whisper ASR services (`extras/asr-services/README.md`) and remote ASR URLs
(`config/defaults.yml`, `PARAKEET_ASR_URL` etc.). But it is live-streaming by design, needs an app build, is heavy and
Docker-centric, and is broad and experimental (ScreenPipe, Swiggy, VR vault graph). The v0.3.1 release notes report "154 passed, 13
failed" integration tests. Its last push was 2026-09-22 (about 6 weeks idle). Audio deletion is "intentionally disabled until a
capture-retention policy exists" (`backend/src/backend/workers/data_audit_jobs.py:10`). There is no MCP *server* yet: issue #152
"MCP server" is open, and `/mcp` appears only in a logging-exclusion list (`backend/src/backend/middleware/app_middleware.py:63`).

### Comparison snapshot (checked 2026-11-04)

| | Nytka | Chronicle |
|---|---|---|
| Licence | Apache-2.0 (app + server) | MIT |
| Last push | app 2026-11-03, server 2026-11-02 | 2026-09-22 (`dev`) |
| Status | app Active; **server Parked** | active project, recent 6-week gap |
| Open issues | app 1, server 59 (mostly maintainer work items) | 22 |
| Maturity | ~5 weeks old; 1,241 app tests, 1,391 server tests (READMEs) | ~1 yr+; releases v0.0.1 to v0.3.1 |
| Android install | Signed APK on Releases; Obtainium; Play/F-Droid/IzzyOnDroid **not yet** (drafts in `docs/fdroid/`) | Build yourself (Expo); stale APK Feb 2026 |
| Omi | Primary, incl. offline sync | Primary (BLE streaming) |
| Phone mic | Yes (app), **server support missing** | Yes, live "duplex" voice session |
| On-device VAD | **No** (server Silero VAD) | No (server) |
| Codec/bitrate | Opus 16 kHz mono, **32 kbps hard-coded** (`app/.../phonemic/OpusEncoder.kt` `BIT_RATE = 32_000`) | Opus 16 kHz mono, **24 kbps hard-coded** (`ChronicleDuplexAudioModule.kt` ~line 154) |
| Batch upload | Queue, drained every 15 min by WorkManager, `NetworkType.CONNECTED` (**any network**, not Wi-Fi-only) (`capture/UploadDrainWorker.kt:65`) | Live stream only |
| Delete after confirm | Yes (only on 200/202) | n/a (no local store) |
| Foreground service | `PhoneMicService` type `microphone` (`AndroidManifest.xml:84`) | Notifee FGS with microphone type (`plugins/with-ws-fgs.js`) |
| Battery-opt exemption | Not requested (`docs/explanation/link-drops.md:56-57`) | not found |
| On-phone STT | Only in `play` build, off by default; `oss` APK has none (`app/build.gradle.kts:121-122`) | none found |
| Server | Docker Compose (or .NET 10 + Postgres from source) | Docker Compose / K8s, Mongo, Redis |
| STT | Any OpenAI-compatible endpoint, whisper.cpp, Groq/OpenAI, Soniox; local profile = `faster-whisper-server` CPU, model `faster-whisper-small` | faster-whisper, NeMo Parakeet, transformers, VibeVoice providers |
| LLM | Any OpenAI-compatible (Ollama documented) | OpenAI / Ollama |
| Retention | `Nytka__Audio__RetentionDays`, default 14, range 0-3650; transcripts forever | screen-context 30 d; audio deletion disabled |
| Search/Q&A | FTS + optional pgvector; Ask with sources; daily digest | Memory search, chat |
| MCP | Server, 22 tools, token scopes | Client (HA plugin); server is open issue #152 |
| Home Assistant | none | plugin |
| Auth/LAN | Named tokens with `admin`/`read` scopes; binds 127.0.0.1:8080; app enforces HTTPS unless "private-network" switch | JWT users; Caddy/Tailscale |
| Cloud by default | None, except a daily GitHub firmware check (switchable) | Deepgram/OpenAI in the default diagram; local optional |

---

## 2. What I found about Homley and BMain (with citations)

From private repo **`bflutz/herbert-homley`** (read via `gh`, account bflutz):

| Item | Finding | Source |
|---|---|---|
| Hardware | GMKtec **NucBox M6**: Ryzen 5 6600H (6C/12T), Radeon 660M iGPU (Vulkan), **32 GB DDR5 (both slots full)**, 1x 954 GB NVMe, 2.5 GbE wired | `README.md` lines 86-95; repo description |
| OS | **Proxmox 9**, LXC containers (CT 110-230), host 192.168.50.50, tailnet 100.107.2.65 | `docs/system-map.md` line 34 |
| Docker | **Not used.** "Proxmox does not support Docker inside LXC…; a VM would avoid that" so apps run as native systemd services | `DECISIONS.md` lines 225-230 |
| LLM | CT 110: **llama.cpp + llama-swap on the 660M (Vulkan), not Ollama** (deliberate), OpenAI API at 192.168.50.52:8080, no auth; models qwen3-coder, gpt-oss-20b, qwen3.8-27b | `docs/system-map.md` line 35; `DECISIONS.md` line 17 |
| RAM headroom | CT 110 may take 24 GiB, leaving ~5 GB for everything else | `DECISIONS.md` lines 213-216 |
| Home Assistant | **Not installed.** VM 100 / .51 reserved, "needs the 64 GB upgrade first"; `ENABLE_HAOS=0` | `README.md` lines 103, 191; `stages/config.sh` lines 157-159 |
| Network | Wired only. Tailscale on host and CTs. Services exposed via `tailscale serve` (HQ, Marinara: "nothing listens on the LAN") | `README.md` line 191; `docs/system-map.md` lines 44, 47 |
| Backups | Nightly restic to Google Drive at 03:00. **Note:** that is an off-site cloud copy. Decide whether recordings may go there. | `docs/system-map.md` line 51 |
| Ollama / Piper | Not on Homley (Open WebUI has `ENABLE_OLLAMA_API=false`) | `stages/apps.sh:361` |

From **`BarendL/video-logs`** (existing Homley↔BMain pattern):

| Item | Finding | Source |
|---|---|---|
| BMain GPU | **NVIDIA RTX 4070 Ti SUPER, 16 GB VRAM** | `deploy/BMAIN.md` lines 8-12 |
| BMain OS | **Windows.** PowerShell, `F:\AI\Video Logs`, Scheduled Task; runs Python natively with `uv` + NVIDIA CUDA wheels (**not WSL/Docker**) | `deploy/BMAIN.md` lines 16-27, 52-62 |
| Whisper on BMain | faster-whisper `large-v3` float16 on CUDA. 7.8 min of video took 40 s (vs ~11 min on CPU). ~4.5 GB VRAM | `deploy/BMAIN.md` lines 8-12 |
| Afrikaans | Video Logs does a `base`-model scan for Afrikaans, so Barend speaks Afrikaans. **Parakeet TDT 0.6B v3 is European-languages only and (to my knowledge) does not cover Afrikaans. Use Whisper large-v3.** | `deploy/BMAIN.md` line 8 |
| Pattern | BMain worker **pulls** jobs from Homley (CT 190) over a job API. If BMain sleeps, the job requeues after 10 min. If Homley is unreachable it retries every 30 s. Started by hand. | `deploy/BMAIN.md` lines 39-48; `docs/system-map.md` lines 58-59 |
| ComfyUI on BMain | `bmain:8189`, firewalled to one CT: precedent for BMain serving a LAN port to Homley | `docs/system-map.md` line 47 |

**Unknown:** whether Ollama is installed on BMain, BMain's RAM/CPU, its sleep settings, and whether Docker Desktop/WSL2 exist there.
The Windows desktop "BMain" registered with Barend is very likely the same machine (same name; Video Logs paths are Windows). **Treat
BMain as Windows 11 until confirmed.** Other bflutz repos matching "homley": only `bflutz/Grassnyer` (`PLAN.md`, a kids' game hosted
in CT 120), with nothing relevant.

---

## 3. Requirement gap table (Nytka)

Legend: **Met** = works as shipped · **Config** = setting/ops step · **Custom** = code change (fork) · **Partial**

| # | Requirement | Status | Evidence / what to do |
|---|---|---|---|
| R1 | 16 kHz mono Opus/AAC ~12-16 kbps | **Custom (small)** | 16 kHz mono Opus is met, but `BIT_RATE = 32_000` is a constant (`OpusEncoder.kt:57`). Patch to 16 000 (or a setting). AAC is reserved in the chunk format but has no decoder (server `ChunkFormat.cs`), so stay on Opus. Speech at 16 kbps Opus is fine for Whisper. |
| R2 | Save audio only when speech (on-phone VAD) | **Partial / Custom (optional)** | The phone uploads continuously. The server drops silence with Silero VAD before storing (server `README.md`). With Opus VBR, silence frames are ~8 bytes (`core/.../SpeechSegments.kt:7`), roughly 1.5 MB/h of silence at 32 kbps. **Stored** audio is speech-only, so the storage requirement is met. Real on-device VAD (Silero ONNX in the app) is custom work. **Community lesson (Yuki): record continuously and run VAD on the server.** Server VAD is confirmed as **Silero VAD v5 via ONNX Runtime** (`src/Nytka.Audio/Vad/SileroVad.cs`). The phone records 16 kHz mono continuously while the switch is on (`OpusEncoder.kt` `SAMPLE_RATE = 16_000`; `device.md`). On-device VAD is now **dropped from the plan**, unless battery measurements force it. |
| R3 | Batched uploads, **Wi-Fi only** | **Custom (small)** | Batching is met (queue + 15-min WorkManager drain). Constraint is `NetworkType.CONNECTED` (`UploadDrainWorker.kt:65`). Change to `UNMETERED` (+ check `drainNow` path). Queue holds up to 1 GiB (~60 h) (android `README.md`). |
| R4 | Delete from phone after Homley confirms | **Met** | "Only a 200 or 202 removes audio from the queue" (android `README.md`). |
| R5 | No on-phone transcription | **Met** | Use the `oss` APK (from Releases). The recogniser exists only in the `play` flavor and is off by default (`app/build.gradle.kts:121-122`; `device.md`). |
| R6 | Battery < 20 % over 12 h | **Unknown, to measure** | "Battery use is not measured yet; expect it to cost more than the pendant's Bluetooth capture" (`device.md`, Phone microphone). Hardware Opus encode + 15-min batches is a good design. Measure on day 1-2. |
| R7 | Battery-optimisation exemption | **Config + Custom (small)** | The app doesn't request `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (`link-drops.md:56-57`). Barend sets Samsung battery to "Unrestricted" + "Never sleeping apps" by hand, or we add the prompt. |
| R7b | Survives reboot / all-day reliability | **Partial** | `PhoneMicService` is a `microphone` foreground service with an ongoing notification. **But phone mic is "off again after a restart"; Nytka never turns it on by itself** (`device.md`). Fine as a privacy default, but Barend must tap it on each morning or after a reboot. An optional auto-start/schedule (e.g. 07:00-19:00) is custom work, and Android 14+ restricts starting mic FGS from background. A morning notification tap is the realistic approach. |
| R8 | All data on Homley, no cloud | **Met + Config** | App talks only to your server, plus a daily GitHub firmware check (switch off). Point STT/LLM at BMain, not Groq/OpenAI. **Exclude the Nytka DB from (or encrypt) the restic→Google Drive backup**, or accept it explicitly. |
| R9 | Raw audio retention configurable, **default: keep indefinitely** | **Config (+ tiny custom)** | `Nytka__Audio__RetentionDays` range 0-3650 (default 14) (`configuration.md:41`). Set **3650** (10 years ≈ indefinite), or patch to allow "never delete". Storage: speech Opus ~18 MB/h of speech at the current 32 kbps (`operations.md`), ~9 MB/h at 16 kbps. 4 h speech/day ≈ 13-26 GB/year. The 954 GB NVMe is shared, so plan to move audio to the Seagate or an added disk eventually. |
| R10 | Transcripts kept forever | **Met** | "Transcripts, titles, summaries, tasks and memories stay until you delete them" (`operations.md`). |
| R11 | Searchable memory + Q&A with local models | **Met + Config** | **Hybrid search is built in:** full-text (`ts_rank_cd`) and vector hits merge by reciprocal-rank fusion once an embeddings endpoint + pgvector are set (`src/Nytka.Storage/SearchStore.cs:71-83`; `configuration.md:365`). Ask cites numbered sources; items open the lines they came from and play from that moment. Summaries roll up per week/month/year/all time (`features.md` "Summaries over time"), plus a daily digest. Point `Nytka__Llm__BaseUrl` at Ollama on BMain. Afrikaans search via the `simple` config (no stemming). |
| R12 | Remote STT on BMain | **Met + Config** | `Nytka__Stt__Url` = BMain's OpenAI-compatible endpoint (must support `verbose_json`). BMain needs such a server (see §4). |
| R13 | BMain off/asleep: backlog | **Met (31 h) + Custom (tiny)** | 3 tries (30 s, 2 min), then auto-retries at +1/2/4/8/16 h, audio kept. After that, `POST /api/v1/transcriptions/retry` (admin). Add a Homley timer: "if BMain health OK and failed batches > 0, call retry". LLM jobs retry similarly (3 rounds hourly). |
| R14 | Phone-mic mode works with server | **Custom (must)** | Server must list `phone-mic` in `/api/v1/info` and should keep `X-Nytka-Source: phone-mic` audio as its own source. Neither is in the server source today. |
| R15 | Omi compatible | **Met** | Omi firmware 3.0.x; offline sync with 3.0.20+. Uninstall the official Omi app first. |
| R16 | LAN-only + auth | **Config** | Server binds `127.0.0.1:8080`. Admin/read tokens. App needs HTTPS, or "private-network" switch for plain http on a VPN/LAN. Use `tailscale serve` (Homley's existing pattern) → `https://nytka.<tailnet>.ts.net`. |
| R17 | MCP | **Met** | `POST /mcp`, token-scoped. |
| R18 | Home Assistant (phase 4) | **Not in Nytka; separate** | No HA on Homley (needs 64 GB RAM per repo). Phase 4 is a separate build. |
| R19 | Licence OK for personal use | **Met** | Apache-2.0. |
| R20 | Two layers: raw transcript = source of truth; memories need a trust check and cite lines | **Met** | Transcripts are kept forever. Memory extraction is grounded: each fact cites its segment ids (`memory_sources`). A fact not backed by the wearer's own lines, or one contradicting a confirmed memory, is stored **unconfirmed** and is not listed, searched, sent to MCP or digested (`features.md` ~lines 624-628, 1288-1290). Tags and duplicate tasks wait in a review inbox. |
| R21 | Off-the-record: mute and redaction | **Met (mute) / Partial (redaction)** | Phone-mic Stop and mute switch. Pendant double-tap mute. `Nytka__Mute__Windows` drops audio by capture time **before transcription** (`configuration.md:44`). After the fact: delete a conversation (audio + transcript + derived items) or "Forget this person". **Not present:** a spoken "off the record" phrase or a per-span redaction tool. Optional custom work. |
| R22 | Diarization / voiceprints | **Partial** | Built in: wearer voice enrolment with TitaNet-small ONNX voiceprints on CPU (`src/Nytka.Audio/Voice/SpeakerEmbedder.cs:28`), and opt-in cross-conversation voice grouping (`Nytka__People__VoiceMatching`, default false, `configuration.md:90`). Per-batch speaker labels come only from the STT provider's `speaker` field. Plain Whisper/speaches gives none, so true diarization (pyannote) is custom (Phase 2b). |

---

## 4. Architecture

```
 Samsung S25 (Nytka oss APK)                Homley (Proxmox, 192.168.50.50)                    BMain (Windows, RTX 4070 Ti SUPER 16 GB)
 ┌──────────────────────────┐   Wi-Fi only  ┌──────────────────────────────────┐   LAN        ┌───────────────────────────────────┐
 │ PhoneMicService (FGS)    │  HTTPS (ts)   │ NEW CT/VM "recorder"             │  HTTP, fw'd  │ STT: OpenAI-compatible server     │
 │ 16 kHz mono Opus 16 kbps │──────────────►│  Nytka server (.NET) :8080       │─────────────►│  faster-whisper large-v3 (CUDA)   │
 │ Room queue (≤1 GiB)      │  30 s chunks, │   ingest → Silero VAD (CPU)      │  WAV batches │  /v1/audio/transcriptions         │
 │ delete on 200/202        │  15-min drain │   → speech Opus in Postgres      │◄─────────────│  verbose_json                     │
 │ (later: Omi over BLE)    │               │   → STT job queue (retry/backlog)│  segments    │                                   │
 └──────────────────────────┘               │   → conversations, FTS, pgvector │              │ LLM: Ollama :11434/v1             │
                                            │   → AI lane: titles/summaries/   │─────────────►│  e.g. 14-32B model for Ask/summary│
                                            │     memories, Ask, digest        │◄─────────────│  + embeddings model               │
                                            │  Postgres 17 (+pgvector)         │              └───────────────────────────────────┘
                                            │  timer: bmain-health → retry     │
                                            │  tailscale serve (HTTPS)         │   fallback when BMain is off:
                                            └──────────────────────────────────┘   LLM → CT 110 llama.cpp (192.168.50.52:8080, OpenAI API)
                                                                                    STT → wait in backlog (no CPU STT by default)
```

**Where each service runs**
- **Phone:** capture, encode, queue, upload. No transcription.
- **Homley, new guest "recorder":** Nytka server + Postgres. Silero VAD runs here on CPU (light, ONNX Runtime). All audio,
  transcripts and indexes live here. Docker is not allowed in LXC per `DECISIONS.md`, so choose one:
  - **(A, recommended) LXC with native install.** Postgres 17 + pgvector from Debian/PGDG, and the Nytka server as a .NET 10 build
    published to a systemd service. Matches Homley's "native packages" rule. Effort is a bit higher (we translate the Compose file).
  - **(B) small Proxmox VM (2 vCPU / 3-4 GB) with Docker Compose**, using the upstream `docker-compose.yml` as-is. Simpler upgrades,
    but costs more RAM on a tight 32 GB host (~5 GB free when CT 110 has a model loaded).
- **BMain:** the heavy models.
  - **STT:** an OpenAI-compatible transcription server with faster-whisper `large-v3` on CUDA. On Windows, two routes:
    1. **Native Python (recommended):** Video Logs already runs faster-whisper CUDA natively via `uv` with NVIDIA wheels
       (`BMAIN.md`). Run `speaches` (formerly `faster-whisper-server`, the image Nytka's local profile uses) or a ~150-line
       FastAPI wrapper natively in its own venv, as a Scheduled Task at logon, firewalled to the recorder CT's IP. Verify that
       speaches runs natively on Windows; if not, use the small wrapper.
    2. **Docker Desktop + WSL2 GPU:** works with NVIDIA's WSL CUDA driver (`--gpus all`). Costs: Docker Desktop licence terms
       (free for personal use), WSL2 VM RAM, port-forwarding from WSL to the LAN (Windows firewall + `localhost` forwarding quirks,
       or mirrored networking mode), and the WSL VM may stop when idle. More moving parts than native.
    - Parakeet: NeMo is Linux-first (WSL2 needed on Windows) and does not cover Afrikaans, so **not recommended**.
  - **LLM:** Ollama for Windows (native, CUDA) on `:11434`, bound to the LAN and firewalled to the recorder CT. A model that fits 16
    GB VRAM alongside Whisper (~4.5 GB): e.g. a 14B Q4 (~9 GB), or a 20-32B model with Whisper unloaded at night. Also an
    embeddings model (e.g. `nomic-embed-text`) for vector search. Because Video Logs and ComfyUI also use this GPU, VRAM contention
    is real. Set Ollama keep-alive short.
- **Jobs and audio between the machines.** Nytka **pushes**: each speech batch goes out as a WAV POST from Homley to BMain, and
  the transcript comes back in the response. The LLM is called the same way. Nothing is stored on BMain. (Video Logs uses a *pull*
  worker instead. Nytka's push model is fine on a LAN.)
- **When BMain is off or asleep:** batches fail, keep their audio and auto-retry for ~31 h. Our Homley timer (every 10 min:
  `GET bmain:port/health` → if up, `POST /api/v1/transcriptions/retry`) drains the backlog whenever BMain wakes. Summaries/Ask: set
  the LLM to BMain's Ollama with **CT 110 as the fallback**, either by a tiny failover proxy or by accepting "Ask unavailable while
  BMain sleeps". Optional: Wake-on-LAN from Homley at a set hour (e.g. 18:00) for a nightly drain. Needs WoL enabled in BMain's
  BIOS/NIC.
- **Daily GPU batch (community lesson, fits BMain sleeping):** rather than keeping Whisper loaded all day, let daytime batches
  queue on Homley (they fail fast and keep their audio while BMain's STT is off). Then once a day, e.g. evening or BMain's first
  wake, the Homley timer calls `POST /api/v1/transcriptions/retry` and the whole day goes through in one run. The STT server
  unloads the model after an idle timeout (verify speaches' model TTL, or have the wrapper exit after `--drain`-style
  completion), and Ollama uses a short `keep_alive`. Scale check: Video Logs measured large-v3 at 7.8 min of audio in 40 s on
  BMain. Yuki reports 7h46m in ~8 min with large-v3. So a full speech day should take minutes, not hours. The trade-off: no same-day
  Ask over today's speech until the batch runs. A one-tap "transcribe now" from HQ covers that. Stay within Nytka's ~31 h
  automatic retry window, or rely on the timer, which restarts the rounds.
- **GPU-use verification (community lesson: silent CPU fallbacks):** at setup, and in the HQ health tile, confirm work actually runs
  on the GPU. Check `nvidia-smi` shows the STT process and VRAM during a batch, `ollama ps` shows `100% GPU` for the loaded
  model, the STT log reports `device=cuda` / `float16`, and a fixed 10-minute test file finishes within an expected time (alert
  if it is ≥5× slower).
- **Away from home:** the queue holds ~60 h of audio on the phone, so Wi-Fi-only uploads resume at home. Q&A from away uses
  Tailscale on the phone (already in use). With the Wi-Fi-only patch, audio never uploads over mobile data.

---

## 5. Phased plan and effort (me = AI coding assistant doing the work; Barend = taps and approvals)

| Phase | Work | My effort | Barend |
|---|---|---|---|
| **1 MVP (phone → Homley → BMain)** | Fork `nytka-app/server` and `nytka-app/android` under bflutz. Server patch: advertise `phone-mic` and tag the source (R14). App patches: Opus 16 kbps (R1), Wi-Fi-only drain (R3), battery-optimisation prompt (R7). Release our own signed APK. Homley guest (option A or B, **open decision**) with a `homley` stage, Postgres, tokens, `tailscale serve`. BMain: STT server (native, idle unload) + Ollama (short keep-alive) + firewall + Scheduled Tasks. Config: STT/LLM URLs, retention (open decision), language, mute windows. **Daily-batch timer** (health check → retry) and **GPU-use verification** (§4). Pilot day, then measure battery/data. | **4.5-6.5 days** (+0.5 d vs before for the daily-batch timer and GPU checks) | Approve forks + new guest; install APK (Obtainium); grant mic/notification; Samsung battery settings; approve BMain installs and firewall; record a test day |
| **2 Polish: retention, search, reliability** | Retention per decision (3650 days or patch "never"). Backup handling per decision. Audio storage on a bigger disk. **Turn on hybrid search** (pgvector + Ollama embeddings; config only, built in). Afrikaans vocabulary. LLM failover proxy (BMain→CT 110). HQ tile (backlog, last upload, battery, GPU check). Morning "start recording" nudge. "Transcribe now" tap. Review the unconfirmed-memory behaviour with Barend's real data. On-device VAD **removed** (server VAD per community lesson). | **3-4 days** (was 3-5 d + optional 2-3 d on-device VAD) | Decide retention/backup; mark own voice (voice enrolment) |
| **2b (optional) Who said what** | Step 1, built in: wearer voice enrolment + `Nytka__People__VoiceMatching`, then tune thresholds on real days (**0.5-1 day**). Step 2, custom: pyannote 3.1 diarization on BMain inside the STT wrapper, returning `speaker` per segment in `verbose_json` so Nytka shows and uses it (**3-5 days** incl. evaluation on Afrikaans/English mixed audio). pyannote models are gated on Hugging Face (one-time download) and add GPU time (Aura reports ~2-2.5 h per 8 h day on a 3090 for its full pipeline). Community warning: wrong speaker labels break downstream memories and people, so ship only if accuracy is good, and keep it off otherwise. | **0.5-1 d** (step 1) / **+3-5 d** (step 2) | Enrol voice; label a few people; judge accuracy |
| **2c (optional) Off-the-record** | Spoken "off the record … back on" phrase detected after transcription, which drops the span's audio and lines before enrichment. Or a "delete the last N minutes" button in the app. | **1.5-3 days** | Decide whether wanted |
| **3 Omi pendant** | Pair Omi with Nytka. Nytka already does **direct Bluetooth capture + offline sync**, with no Omi cloud or Omi backend (Omi's self-host still needs Firebase/Pinecone/Redis/Typesense per LocusTether's notes). Check firmware ≥ 3.0.20 and link drops (known Doze issue, `nytka-app/server` #280). Keep phone-mic as a fallback source. Reference if Nytka's driver fails: `cgallic/omi-pendant-chronicle` (3-day ring buffer, push-to-sync, POST /raw). | **0.5-1 day** (+ debugging time if link drops bite) | Buy Omi; update firmware with official app, then uninstall it; pair; wear test |
| **4 Voice Q&A via earbuds** | Needs Home Assistant (VM 100 reserved, needs the 64 GB RAM upgrade per repo). Assist pipeline: Whisper on BMain (Wyoming), Ollama agent, Piper TTS, bridged to Nytka's Ask API or MCP. **Template: `moimart/conversation-hass`** (Wyoming/Piper, MCP, Postgres log, phone satellite) lowers design risk. | **4-6 days** after HA exists (was 4-7; the template trims the top end) | Decide on the RAM upgrade; HA companion app setup; earbuds |

**Core path (P1-P3): ~8-11.5 days** (was ~7.5-12 incl. optional on-device VAD). Optional 2b/2c add 0.5-8 days. Estimates assume upstream code behaves as documented. Add ~30 % buffer for a 5-week-old, single-maintainer project.

---

## 6. What Barend must physically do

1. Approve the plan, the forks under `bflutz`, and a new Homley guest (CT or VM). Approve installs on BMain.
2. On the S25: install Obtainium (or the APK), add our fork, install Nytka, scan/enter the server address and token, and grant
   microphone + notifications. Set **Settings → Battery → Background usage limits → Never sleeping apps → Nytka**, App battery =
   **Unrestricted**, and keep Tailscale on.
3. Each morning (or after a reboot), tap **Phone microphone** on. Optionally do voice enrolment once.
4. On BMain: allow the installer prompts / UAC, and decide sleep settings (or enable Wake-on-LAN in BIOS).
5. Phase 3: buy the Omi, update its firmware with the official app, uninstall the official app, and pair with Nytka.
6. Phase 4: RAM upgrade, if chosen.

## 7. Decisions Barend needs to make

**Still open (Barend has not decided):** (1) when Phase 1 starts; (2) whether recordings go in the Google Drive backup; (3) VM
with Docker vs direct install on Homley; (4) whether and when Homley wakes BMain (Wake-on-LAN / daily batch time); (5) 10-year
(3650-day) retention vs a code change for "never delete". The rest below are also his calls.


- **Retention (open):** the direction is to keep raw audio indefinitely, but Barend hasn't chosen yet between 3650 days and patching for "never". Storage is ~13-26 GB/year at 4 h of speech/day.
- **Backups (open):** include recordings in the nightly restic copy to **Google Drive** (cloud, encrypted by restic) or keep them
  Homley-only (no off-site copy)?
- **Who/what gets recorded:** e.g. pause at work meetings, with clients, family at home, children. Use the mute and mute windows
  (supported). Use the optional "recording" chime (server ROADMAP mentions it)?
- **Hosting choice on Homley (open):** native LXC (A) vs VM with Docker (B).
- **BMain availability (open):** Wake-on-LAN at a set hour for the daily batch, or "whenever it's on". Also when the daily batch runs.
- **Phase 1 start date (open).**
- **Diarization (Phase 2b) and off-the-record phrase (2c):** wanted or not.
- **Q&A model on BMain** and whether CT 110 is the fallback.
- **Phase 4:** the 64 GB RAM upgrade for Home Assistant.

**Consent / legal note (South Africa; cautious, not legal advice):**
- **RICA** (Act 70 of 2002) s 4 generally permits a person **who is a party to a communication** to record it (one-party consent).
  It does **not** cover recording conversations Barend is not part of. A phone left on a table, or recording others while he is
  away, may amount to unlawful interception.
- **POPIA** (Act 4 of 2013): recordings of identifiable people are personal information. There is an exclusion for processing "in
  the course of a purely personal or household activity" (s 6(1)(a)), but it is likely read narrowly. Recording at work, of clients
  or colleagues, or for business use probably falls outside it and brings POPIA duties (lawful basis, notice, security, retention
  limits). Voiceprints (speaker recognition) may be treated as biometric, i.e. special personal information.
- Practical guidance: tell household members and regulars, mute in sensitive settings (doctor, legal, HR, children's privacy), and
  get advice from a South African attorney before using recordings of others for anything beyond personal memory.

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Samsung background killing / FGS stops** | Microphone-type FGS + "Never sleeping apps" + Unrestricted battery. Watch the HQ tile for "no audio since X". Phone mic is intentionally off after a reboot (morning nudge). |
| **Battery > 20 %/12 h** (unmeasured upstream) | 16 kbps, Wi-Fi-only 15-min batches. Measure on day 1. If high, on-device VAD (pause encoding in silence) is the lever. |
| **Opus encoding** | Uses Android's MediaCodec Opus encoder; the switch says if the phone lacks one (the S25 should have it; verify). Bitrate change must stay compatible with the server's `OpusFs320` chunk format (20 ms frames, unchanged). |
| **LAN access when away** | ~60 h phone queue; Wi-Fi-only uploads resume at home. Tailscale for Ask. Android 17 `ACCESS_LOCAL_NETWORK` is handled by the app (`core/.../LocalNetwork.kt`). |
| **BMain off / asleep / gaming** | Backlog + auto-retry + Homley timer. VRAM contention with ComfyUI/Video Logs/games: short Ollama keep-alive, stop task when gaming. |
| **Windows on BMain** | Prefer native CUDA Python/Ollama over WSL2/Docker. If WSL2: GPU passthrough works with NVIDIA's driver, but LAN port exposure and the WSL VM idling are the usual pain points. |
| **Upstream abandonment** (server "Parked", 1 maintainer, 5 weeks old) | Own the fork; pin versions; keep patches small and documented; Nytka's export/import gives an exit path. |
| **Homley RAM/disk** | Recorder guest capped ~2-4 GB. Audio storage plan before ~1 year. |
| **Afrikaans/English mix** | Whisper large-v3 with `language=auto` + vocabulary. Avoid Parakeet. FTS has no Afrikaans stemming. |
| **Silent CPU fallback on BMain** (LocusTether lesson) | GPU-use verification in §4 and the HQ tile. |
| **Diarization errors poison memories/people** | Keep voice grouping off until enrolment works. Pyannote only behind an accuracy check. Memories already need grounding in the wearer's lines. |
| **Cloud leakage** | Turn off the firmware check, never set Groq/OpenAI/Soniox keys, and decide on the restic/Google Drive question. |
| **Legal/consent** | See §7. |

## 9. Community lessons (from Barend's research notes, checked against Nytka)

| Lesson (source) | How it applies to Nytka | Plan change |
|---|---|---|
| Record 16 kHz mono continuously and run VAD on the server, not the phone (Yuki, yukisato.dev) | Already Nytka's design: phone 16 kHz mono Opus, server Silero VAD v5 (`SileroVad.cs`). Capture times are kept, so the timeline is accurate. | On-device VAD dropped. |
| Process each day as one GPU batch and unload the model between runs; large-v3 ran 7h46m in ~8 min (Yuki) | Nytka transcribes per batch through one URL. A daily batch is achieved with the BMain-off backlog + timer-driven retry. | Daily-batch timer + idle unload in Phase 1. |
| Roll summaries up daily → weekly → monthly (Yuki) | Built in: daily digest, weekly/monthly/yearly "Over time" summaries. | None (config only). |
| Diarization is the biggest pain; errors break everything downstream (r/LocalLLaMA). Aura: pyannote 3.1 + ECAPA voiceprints, ~2-2.5 h per 8 h day on a 3090 | Nytka has wearer voiceprints (TitaNet-small) and opt-in voice grouping, but no diarization unless the STT returns `speaker`. | New optional Phase 2b with honest effort. |
| Silent CPU fallbacks; check `ollama ps` and GPU use (LocusTether) | Applies to the STT server and Ollama on BMain. | GPU-use verification step. |
| Omi's official "self-hosted" backend still needs Firebase/Pinecone/Redis/Typesense (LocusTether); direct BLE capture works (omi-pendant-chronicle) | Nytka captures Omi directly over Bluetooth, with no Omi backend. | Phase 3 unchanged; omi-pendant-chronicle noted as fallback reference. |
| Omi app can point at Speaches / whisper-server using the full `/v1/audio/transcriptions` path (r/OmiAI) | Same for Nytka: `Nytka__Stt__Url` is the **full** URL (`configuration.md:25`), e.g. `http://bmain:8000/v1/audio/transcriptions`. Must support `verbose_json`. | Config note. |
| Raw transcripts searchable but never automatically "memories"; trust check; off-the-record redaction before processing (Remnic) | Nytka: memories are grounded with cited lines; ungrounded ones are stored unconfirmed and hidden. Mute windows drop audio before transcription. No spoken off-the-record phrase. | R20/R21 added; optional Phase 2c. |
| BM25 + vector search with time filters; answers cite timestamps (atopile AI Pin) | Nytka: hybrid FTS + vector with RRF, range filters, Ask with numbered sources that play from the moment. Overnight upload while charging is possible, but our Wi-Fi 15-min drain is better for a 60 h queue. | Hybrid search turned on in Phase 2. |
| Separate retention for audio vs transcripts; DB localhost-only; push digests (im-watching-you) | Nytka: `RetentionDays` covers audio only, and transcripts stay forever. By default the server is published on 127.0.0.1 (`NYTKA_BIND`, `configuration.md:106`) behind `tailscale serve`. Digests and nudges go to the app. | None. |
| Local always-listening HA assistant with Wyoming/Piper, MCP, phone satellite (conversation-hass) | Template for Phase 4. | Phase 4 effort 4-6 d. |

Not adopted: Syncthing transport and daily JSONL files (Yuki). Nytka's queue-with-confirmation and Postgres already cover these
with delete-after-confirm. Separate stacks such as Aura or LocusTether would replace Nytka rather than extend it.
