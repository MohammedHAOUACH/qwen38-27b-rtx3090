# Projekt-Gedächtnis: Qwen3.8-27B vLLM Serving (RTX 3090)

## Nutzer-Zielkonfiguration (Stand 2026-08-31)

Der Nutzer betreibt den Stack via `docker compose --profile single up -d` und hat
seine Wünsche in `.env` (75 Bytes, existiert) und Anfragen festgelegt:

1. **SPEC=dflash2** — DFlash2 Block-Drafter, NICHT das MTP-Default. Steht bereits
   in `.env` (`SPEC=dflash2`, `PREFIX_CACHE=1`). Nicht auf MTP "zurückkorrigieren".
2. **Kein erzwungenes "reasoning low"** — Anfrage nach thinking/reasoning-Steuerung
   wurde geklärt: der Start-Script hat keinen serverweiten Reasoning-Effort-Schalter;
   Thinking steuert der Client pro Request über `chat_template_kwargs`
   (`enable_thinking: false`) oder `reasoning_effort`. Serverseitig gäbe es nur
   `--reasoning-parser qwen3` (Parsing, kein Level). Default im Stack: thinking an.
3. **docker compose** ist der gewählte Betriebsweg (nicht das venv-Setup).

## Wichtige Repo-Fakten

- `.env` wird von docker-compose.yml per `env_file` in den Container gereicht;
  dort gehören die Knobs (SPEC, DFLASH_TOKENS, CTX, KV_MEM, ...), nie ins Repo.
- `SPEC=dflash2` braucht den Drafter `models/Qwen3.8-27B-DFlash2-W4A16`
  (fetcht `docker compose run --rm prepare` automatisch, außer DFLASH2=0).
- DFlash2 läuft nur auf vLLMs V2 Model Runner; CTX=fast (bf16/64k) ist der
  Standard-Pfad; CTX=long nutzt int8-KV (Triton), CTX=huge KVarN.
- Thinking/Reasoning: Modell hat thinking an als Default; Off pro Request via
  `"chat_template_kwargs": {"enable_thinking": false}` (verify.sh, bench/* so).
- `single-user/start_qwen.sh` baut `--speculative-config` aus SPEC; SPEC_CFG für
  dflash2: `{"method":"dflash","model":$DRAFT,"num_speculative_tokens":$DRAFT_TOKENS}`.

## Konventionen

- Kommentare im Repo sind lang und messbasiert (jede Zahl hat einen Messkontext) —
  beim Editieren von start_qwen.sh diesen Stil halten, keine "Optimierungen" ohne
  Messung rückgängig machen (steht explizit in mehreren Kommentaren).
- Deutsch mit dem Nutzer kommunizieren (Nutzer schreibt Deutsch; eine Antwort
  auf Wunsch auch französisch).

## LLM-Client des Nutzers (separat vom Serving-Stack)

- Der Nutzer nutzt den **DeepSeek Harness `@deepseek-ai/dsh`** (`npx @deepseek-ai/dsh`),
  installiert, Konfiguration unter `~/.dsh/`. Wichtig: `~/.dsh/settings.yaml` zeigt
  `baseURL: http://127.0.0.1:1234/v1` mit model id `qwen3.8-27b` — der Harness ist
  also auf den lokalen vLLM-Server dieses Repos gerichtet (PORT 1234 = docker compose
  Default). `reasoningEffort: off` / `thinking: disabled` sind dort bereits gesetzt.
- Wunsch "reasoning low" für den Client: dsh-seitig über `reasoningEffort` in
  `~/.dsh/settings.yaml` steuern (aktuell `off`); serverseitig gibt es keinen
  globalen Effort-Schalter.
