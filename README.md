# imkt5

Plataforma modular multi-tenant para pipelines de mÃ­dia (imagem, Ã¡udio, vÃ­deo, texto) com **entrada conversacional** (Telegram/WhatsApp/Web), **fila de jobs**, e **local-first com fallback remoto**.

```
Canais (TG Â· WA Â· Web)  â†’  Gateway  â†’  Fila  â†’  Workers
```

Cada worker Ã© um serviÃ§o HTTP independente. Cada pedido Ã© um `Job` que o Gateway despacha para o worker certo baseado em **capability** (`image.generation`, `audio.tts`, `research.market`, etc.). MÃºltiplas instÃ¢ncias do mesmo worker sÃ£o distribuÃ­das pelo matcher (local antes de remoto).

## Status

**Funciona end-to-end** â€” gateway HTTP + UI web + 4 workers reais plugados. 35 testes passando.

| Componente | Estado |
|---|---|
| Contratos multi-tenant | âœ… implementado |
| Recipe engine (YAML declarativo + 3 modos de aprovaÃ§Ã£o) | âœ… implementado |
| Capability registry com local-first + health check | âœ… implementado |
| Quick dispatch (1 capability â†’ 1 job) | âœ… implementado |
| HTTP dispatcher + saturaÃ§Ã£o + logging | âœ… implementado |
| UI web em `/ui` | âœ… implementado |
| Config centralizado 3 nÃ­veis (global/tenant/user) | âœ… implementado |
| Worker `research` (Tavily) | âœ… validado live |
| Worker `auto-reviewer` (Ollama+OpenRouter) | âœ… validado live |
| Worker `inemaimg-adapter` (flux2-klein, qwen-edit, ernie) | âœ… validado live |
| Worker `inemavox-adapter` (TTS/dubbing/transcribe) | âœ… adapter pronto |
| Canal Telegram/WhatsApp | â¸ pendente |
| PersistÃªncia Postgres | â¸ pendente (hoje in-memory) |
| 12 workers do timesmkt3 (copywriter, ad-designer, etc.) | â¸ a portar |
| Workers yt-pub-lives (substitui 10 clones) | â¸ a portar |

## Quickstart

```bash
# instalar
python -m venv .venv
.venv/bin/pip install -e ".[dev]" pyyaml fakeredis httpx

# subir tudo
./scripts/start-dev.sh

# abrir UI
open http://localhost:8080/ui

# parar tudo
./scripts/stop-dev.sh
```

PrÃ©-requisitos:
- **Ollama** rodando (`qwen2.5:14b` baixado) â€” pra `auto-reviewer`
- **inemaimg** rodando em `:8000` (opcional â€” pra gerar imagem real)
- **inemavox** rodando em `:8010` (opcional â€” pra TTS real)
- `TAVILY_API_KEY` no `.env` (jÃ¡ populado a partir do timesmkt3)

Teste via `curl`:
```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{"capability":"research.market","payload":{"queries":["cafÃ© 2026"]}}'
```

## Arquitetura em 3 linhas

1. **Quick dispatch** â€” pedido simples (1 capability) Ã© um `POST /jobs` direto.
2. **Recipe Runner** â€” fluxos compostos (mÃºltiplas capabilities encadeadas) vÃªm de YAML em `recipes/*.yaml`, com dependÃªncias/paralelismo/aprovaÃ§Ãµes declarativas.
3. **Capability matcher** â€” registry de workers com `local: true/false` + `priority` + `health`; fallback automÃ¡tico local â†’ remoto.

Diagramas em `doc/diagrams/` (SVG + PNG).

## Estrutura

```
imkt5/
â”œâ”€â”€ imkt5/              pacote Python (gateway, dispatcher, runner, tipos)
â”‚   â”œâ”€â”€ config.py       loader de config 3-nÃ­veis
â”‚   â”œâ”€â”€ gateway/        HTTP dispatcher, FastAPI, UI web, jobs store
â”‚   â”œâ”€â”€ capabilities/   registry + matcher (local-first)
â”‚   â”œâ”€â”€ recipes/        YAML loader, state-machine runner, approvals
â”‚   â”œâ”€â”€ memory/         SQLite FTS5 multi-tenant
â”‚   â”œâ”€â”€ tools/          dispatch_job + run_recipe (tools pro LLM)
â”‚   â””â”€â”€ types/          dataclasses canÃ´nicas
â”œâ”€â”€ workers/            serviÃ§os HTTP independentes
â”‚   â”œâ”€â”€ _base/          BaseWorker + storage helper
â”‚   â”œâ”€â”€ auto-reviewer/  LLM review (Ollama local + OpenRouter fallback)
â”‚   â”œâ”€â”€ inemaimg-adapter/  wrapper do inemaimg (image.generation)
â”‚   â”œâ”€â”€ inemavox-adapter/  wrapper do inemavox (audio.*)
â”‚   â””â”€â”€ research/       Tavily (research.market)
â”œâ”€â”€ config/
â”‚   â”œâ”€â”€ defaults.yaml   fonte Ãºnica dos magic numbers globais
â”‚   â””â”€â”€ workers.yaml    catÃ¡logo de workers registrados
â”œâ”€â”€ profiles/           config por tenant
â”‚   â”œâ”€â”€ _template/      stack de identidade (SOUL/AGENTS/USER/MEMORY)
â”‚   â””â”€â”€ <tenant_id>/    SOUL.md + config.yaml + knowledge/
â”œâ”€â”€ recipes/            fluxos compostos (YAML)
â”œâ”€â”€ scripts/            start-dev.sh, stop-dev.sh, notify.sh
â”œâ”€â”€ data/               artefatos locais (gitignored)
â”œâ”€â”€ logs/               logs de cada processo (gitignored)
â”œâ”€â”€ tests/              35 testes
â””â”€â”€ doc/                documentaÃ§Ã£o
```

## DocumentaÃ§Ã£o

| Doc | Quando ler |
|---|---|
| [`doc/architecture.md`](doc/architecture.md) | Entender o design canÃ´nico |
| [`doc/backlog.md`](doc/backlog.md) | ðŸ—ºï¸ **Roadmap completo do que falta** â€” priorizado, com esforÃ§o |
| [`doc/QUICKSTART.md`](doc/QUICKSTART.md) | Testar end-to-end com curl |
| [`doc/creating-workers.md`](doc/creating-workers.md) | Criar um worker novo |
| [`doc/creating-recipes.md`](doc/creating-recipes.md) | Criar uma receita nova |
| [`doc/defaults.md`](doc/defaults.md) | Onde mora cada valor default |
| [`doc/concurrency-model.md`](doc/concurrency-model.md) | Como o sistema escala |
| [`doc/env-inventory.md`](doc/env-inventory.md) | Mapa de credenciais dos projetos-fonte |
| [`doc/api-mapping-inemaimg-inemavox.md`](doc/api-mapping-inemaimg-inemavox.md) | Contratos reais dos upstreams |
| [`doc/admin-ui-viability.md`](doc/admin-ui-viability.md) | AnÃ¡lise da UI de admin (viabilidade) |
| [`doc/diagrams/`](doc/diagrams/) | Diagramas visuais (arquitetura, provider chain, fluxos) |
| [`CLAUDE.md`](CLAUDE.md) | Regras pra futuras sessÃµes de Claude Code |

## Origens

- **Blueprint arquitetural**: `intelecto` (spec â€” contratos `BaseProvider/Channel/Tool` + stack de identidade em Markdown)
- **PeÃ§as reutilizadas**: `openpcbot` (schema SQLite, WhatsApp bridge, dashboard structure)
- **Workers a absorver** (prÃ³ximas fases): `timesmkt3` (12 agentes) Â· `yt-pub-lives*` (consolidaÃ§Ã£o dos 10 clones)

## LicenÃ§a

MIT

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe