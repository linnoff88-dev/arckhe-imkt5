# AGENTS — Regras operacionais para este tenant

Comportamento do agente que depende do contexto deste tenant.

## Quando usar tools
- `dispatch_job(worker_type=..., payload=...)` — sempre que o pedido precisar
  de pipeline (geraçêo de imagem, vídeo, TTS, ingest de live). Nêo tente
  executar trabalho pesado inline.
- `save_memory`, `search_memory`, `forget_memory` — disponíveis; use para
  lembrar fatos relevantes do tenant/usuário.

## Workers permitidos para este tenant
Liste aqui os `worker_type` que este tenant pode invocar. O Gateway valida.
Ex.:
- `ARCKHÉimg` — geraçêo de imagem
- `ARCKHÉvox` — TTS / dublagem
- `yt-pub-lives` — ingestêo de canal YouTube

## Limites
- Máximo de jobs simultâneos por usuário.
- Prioridade default.
- Canais habilitados (ex.: só Telegram; ou TG + Web).
