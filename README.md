ï»¿# imkt5

Plataforma modular multi-tenant para pipelines de mêÂ­dia (imagem, êÂ¡udio, vêÂ­deo, texto) com **entrada conversacional** (Telegram/WhatsApp/Web), **fila de jobs**, e **local-first com fallback remoto**.

```
Canais (TG êÂ· WA êÂ· Web)  ê¢â â  Gateway  ê¢â â  Fila  ê¢â â  Workers
```

Cada worker êÂ© um serviêÂ§o HTTP independente. Cada pedido êÂ© um `Job` que o Gateway despacha para o worker certo baseado em **capability** (`image.generation`, `audio.tts`, `research.market`, etc.). MêÂºltiplas instêÂ¢ncias do mesmo worker sêÂ£o distribuêÂ­das pelo matcher (local antes de remoto).

## Status

**Funciona end-to-end** ê¢â¬â gateway HTTP + UI web + 4 workers reais plugados. 35 testes passando.

| Componente | Estado |
|---|---|
| Contratos multi-tenant | ê¢Åâ¦ implementado |
| Recipe engine (YAML declarativo + 3 modos de aprovaêÂ§êÂ£o) | ê¢Åâ¦ implementado |
| Capability registry com local-first + health check | ê¢Åâ¦ implementado |
| Quick dispatch (1 capability ê¢â â 1 job) | ê¢Åâ¦ implementado |
| HTTP dispatcher + saturaêÂ§êÂ£o + logging | ê¢Åâ¦ implementado |
| UI web em `/ui` | ê¢Åâ¦ implementado |
| Config centralizado 3 nêÂ­veis (global/tenant/user) | ê¢Åâ¦ implementado |
| Worker `research` (Tavily) | ê¢Åâ¦ validado live |
| Worker `auto-reviewer` (Ollama+OpenRouter) | ê¢Åâ¦ validado live |
| Worker `ARCKHÉimg-adapter` (flux2-klein, qwen-edit, ernie) | ê¢Åâ¦ validado live |
| Worker `ARCKHÉvox-adapter` (TTS/dubbing/transcribe) | ê¢Åâ¦ adapter pronto |
| Canal Telegram/WhatsApp | ê¢ÂÂ¸ pendente |
| PersistêÂªncia Postgres | ê¢ÂÂ¸ pendente (hoje in-memory) |
| 12 workers do timesmkt3 (copywriter, ad-designer, etc.) | ê¢ÂÂ¸ a portar |
| Workers yt-pub-lives (substitui 10 clones) | ê¢ÂÂ¸ a portar |

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

PrêÂ©-requisitos:
- **Ollama** rodando (`qwen2.5:14b` baixado) ê¢â¬â pra `auto-reviewer`
- **ARCKHÉimg** rodando em `:8000` (opcional ê¢â¬â pra gerar imagem real)
- **ARCKHÉvox** rodando em `:8010` (opcional ê¢â¬â pra TTS real)
- `TAVILY_API_KEY` no `.env` (jêÂ¡ populado a partir do timesmkt3)

Teste via `curl`:
```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{"capability":"research.market","payload":{"queries":["cafêÂ© 2026"]}}'
```

## Arquitetura em 3 linhas

1. **Quick dispatch** ê¢â¬â pedido simples (1 capability) êÂ© um `POST /jobs` direto.
2. **Recipe Runner** ê¢â¬â fluxos compostos (mêÂºltiplas capabilities encadeadas) vêÂªm de YAML em `recipes/*.yaml`, com dependêÂªncias/paralelismo/aprovaêÂ§êÂµes declarativas.
3. **Capability matcher** ê¢â¬â registry de workers com `local: true/false` + `priority` + `health`; fallback automêÂ¡tico local ê¢â â remoto.

Diagramas em `doc/diagrams/` (SVG + PNG).

## Estrutura

```
imkt5/
ê¢âÅê¢ââ¬ê¢ââ¬ imkt5/              pacote Python (gateway, dispatcher, runner, tipos)
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ config.py       loader de config 3-nêÂ­veis
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ gateway/        HTTP dispatcher, FastAPI, UI web, jobs store
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ capabilities/   registry + matcher (local-first)
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ recipes/        YAML loader, state-machine runner, approvals
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ memory/         SQLite FTS5 multi-tenant
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ tools/          dispatch_job + run_recipe (tools pro LLM)
ê¢ââ   ê¢ââê¢ââ¬ê¢ââ¬ types/          dataclasses canêÂ´nicas
ê¢âÅê¢ââ¬ê¢ââ¬ workers/            serviêÂ§os HTTP independentes
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ _base/          BaseWorker + storage helper
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ auto-reviewer/  LLM review (Ollama local + OpenRouter fallback)
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ ARCKHÉimg-adapter/  wrapper do ARCKHÉimg (image.generation)
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ ARCKHÉvox-adapter/  wrapper do ARCKHÉvox (audio.*)
ê¢ââ   ê¢ââê¢ââ¬ê¢ââ¬ research/       Tavily (research.market)
ê¢âÅê¢ââ¬ê¢ââ¬ config/
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ defaults.yaml   fonte êÂºnica dos magic numbers globais
ê¢ââ   ê¢ââê¢ââ¬ê¢ââ¬ workers.yaml    catêÂ¡logo de workers registrados
ê¢âÅê¢ââ¬ê¢ââ¬ profiles/           config por tenant
ê¢ââ   ê¢âÅê¢ââ¬ê¢ââ¬ _template/      stack de identidade (SOUL/AGENTS/USER/MEMORY)
ê¢ââ   ê¢ââê¢ââ¬ê¢ââ¬ <tenant_id>/    SOUL.md + config.yaml + knowledge/
ê¢âÅê¢ââ¬ê¢ââ¬ recipes/            fluxos compostos (YAML)
ê¢âÅê¢ââ¬ê¢ââ¬ scripts/            start-dev.sh, stop-dev.sh, notify.sh
ê¢âÅê¢ââ¬ê¢ââ¬ data/               artefatos locais (gitignored)
ê¢âÅê¢ââ¬ê¢ââ¬ logs/               logs de cada processo (gitignored)
ê¢âÅê¢ââ¬ê¢ââ¬ tests/              35 testes
ê¢ââê¢ââ¬ê¢ââ¬ doc/                documentaêÂ§êÂ£o
```

## DocumentaêÂ§êÂ£o

| Doc | Quando ler |
|---|---|
| [`doc/architecture.md`](doc/architecture.md) | Entender o design canêÂ´nico |
| [`doc/backlog.md`](doc/backlog.md) | ê°Å¸âÂºê¯Â¸Â **Roadmap completo do que falta** ê¢â¬â priorizado, com esforêÂ§o |
| [`doc/QUICKSTART.md`](doc/QUICKSTART.md) | Testar end-to-end com curl |
| [`doc/creating-workers.md`](doc/creating-workers.md) | Criar um worker novo |
| [`doc/creating-recipes.md`](doc/creating-recipes.md) | Criar uma receita nova |
| [`doc/defaults.md`](doc/defaults.md) | Onde mora cada valor default |
| [`doc/concurrency-model.md`](doc/concurrency-model.md) | Como o sistema escala |
| [`doc/env-inventory.md`](doc/env-inventory.md) | Mapa de credenciais dos projetos-fonte |
| [`doc/api-mapping-ARCKHÉimg-ARCKHÉvox.md`](doc/api-mapping-ARCKHÉimg-ARCKHÉvox.md) | Contratos reais dos upstreams |
| [`doc/admin-ui-viability.md`](doc/admin-ui-viability.md) | AnêÂ¡lise da UI de admin (viabilidade) |
| [`doc/diagrams/`](doc/diagrams/) | Diagramas visuais (arquitetura, provider chain, fluxos) |
| [`CLAUDE.md`](CLAUDE.md) | Regras pra futuras sessêÂµes de Claude Code |

## Origens

- **Blueprint arquitetural**: `intelecto` (spec ê¢â¬â contratos `BaseProvider/Channel/Tool` + stack de identidade em Markdown)
- **PeêÂ§as reutilizadas**: `openpcbot` (schema SQLite, WhatsApp bridge, dashboard structure)
- **Workers a absorver** (prêÂ³ximas fases): `timesmkt3` (12 agentes) êÂ· `yt-pub-lives*` (consolidaêÂ§êÂ£o dos 10 clones)

## LicenêÂ§a

MIT

---

Projeto adaptado da comunidade ARCKHÉ TDS - Marca Arckhe