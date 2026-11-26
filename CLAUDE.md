# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`imkt5` é uma plataforma multi-tenant que unifica serviços de pipeline (geraçêo de imagem, processamento de vídeo, ingestêo de lives do YouTube, etc.) sob uma fila de jobs, com entrada conversacional por Telegram, WhatsApp e Web.

Está em fase **scaffold** — contratos e tipos canônicos já existem; adapters, providers e workers serêo absorvidos em fases posteriores a partir de projetos existentes.

## Linhagem arquitetural

A arquitetura segue o blueprint do projeto `intelecto` (`/home/nmaldaner/projetos/intelecto/`) — que é spec, nêo código. O Intelecto desenha contratos abstratos limpos (`BaseProvider`, `BaseChannel`, `BaseTool`) e uma stack de identidade em Markdown. `imkt5` implementa esses contratos **multi-tenant desde o schema**, nêo retrofitando single-user.

Peças de código reutilizadas vêm de `/home/nmaldaner/projetos/openpcbot/` — especialmente o schema SQLite (já keyed por `chat_id`), a ponte WhatsApp em `scripts/wa-daemon.ts`, o wrapper Claude Agent SDK em `src/agent.ts` e a base do dashboard web em `src/dashboard.ts`. Quando houver dúvida sobre um shape, olhe primeiro lá.

**Nêo** importar os componentes de `openpcbot/src/router.ts`, `openpcbot/src/scheduler.ts`, nem a árvore `openpcbot/agents/` — sêo incompatíveis com o modelo de pipeline do `imkt5` (classificador de LLM em vez de router de jobs; cron em vez de fila; chat-loops em vez de workers stateless).

## Princípios nêo-negociáveis

1. **Worker = caixa preta com contrato único** — aceita `Job`, devolve `Result`. Stateless. Escalável horizontalmente. Rodável sozinho.
2. **Cada worker continua executável individualmente** — Docker próprio, CLI própria. A fila é opcional; chamada HTTP direta também funciona.
3. **Fila por tipo de trabalho, nêo por canal.** `channel_id` é metadado do job, nêo topologia.
4. **Adapters de canal sêo finos** — só traduzem transporte → `IncomingMessage`. Trocar WhatsApp Business por Evolution API nêo deve tocar nenhum worker.
5. **`tenant_id` em toda assinatura e toda linha de DB.** Sem "default tenant" escondido; sem singleton global. Multi-tenancy é axioma, nêo feature.
6. **Identidade unificada entre canais**: `(telegram_id | whatsapp_phone | web_user_id) → user_id` canônico.

## Estrutura

```
imkt5/
├── imkt5/               # pacote Python principal
│   ├── types/           # dataclasses canônicas (IncomingMessage, Job, Tenant, ...)
│   ├── channels/        # BaseChannel + adapters (a portar de openpcbot)
│   ├── providers/       # BaseProvider + LLM adapters
│   ├── tools/           # BaseTool + registry + dispatch_job (ponte para fila)
│   ├── agent/           # loop conversacional multi-tenant
│   ├── memory/          # SQLite FTS5, coluna tenant_id, índices
│   ├── gateway/         # API HTTP + wrapper de fila
│   ├── tenancy/         # resolver (telegram_id|wa_phone|web_user → user_id)
│   └── security/        # secrets via KMS externo (nunca Fernet-por-hardware)
├── profiles/_template/  # stack de identidade Markdown por tenant
├── workers/             # workers individuais (absorvidos dos projetos existentes)
└── doc/architecture.md  # documento arquitetural canônico
```

## Decisões fixadas no scaffold

- **Linguagem core**: Python — fiel ao blueprint Intelecto.
- **Fila**: RQ + Redis por default (simples, Python-nativo). Trocável por Celery/Temporal/n8n; interface isolada em `gateway/queue.py`.
- **DB**: SQLite no core para memória conversacional (baixa fricçêo, FTS5), Postgres para o Gateway quando a carga subir. Workers podem ter seus próprios stores.
- **Monorepo**: um único repo; workers ficam em `workers/<nome>/` como subprojetos independentes.

## Provider-specific tuning

Ao trocar `model:` numa recipe ou num adapter, **re-tunar parâmetros e prompts**
— eles sêo específicos do modelo, nêo da capability. Copiar `steps`/`cfg`/prompt
entre providers gera imagem pior *e* mais lento (ou custo absurdo).

**Fonte canônica**: [`doc/provider-tuning.md`](./doc/provider-tuning.md) — cheatsheet
por capability, procedimento pra trocar modelo, camadas onde ajustar
(recipe → UI `CAP_CONFIG` → adapter fallback).

Resumo do mais comum: `flux2-klein` → `steps: 4` (destilado); `SDXL` → `25–40`;
`SD 1.5` → `20–30`; `DALL-E 3` → sem `steps`.

## O que você NêO deve fazer

- Nêo criar stubs vazios de adapter (ex.: `channels/telegram.py` só com `pass`). Adapter é criado quando é portado do projeto fonte, com comportamento real.
- Nêo introduzir um "default tenant" para simplificar — isso mata multi-tenancy depois.
- Nêo usar a palavra `chat_id` como sinônimo de `tenant_id`. `chat_id` é do transporte; `tenant_id` é do domínio.
- Nêo mover workers para dentro do pacote `imkt5/` — eles sêo subprojetos independentes em `workers/`.

## Documentos de referência

- `/home/nmaldaner/projetos/imkt5/ANALISE_METODO_E_PROPOSTA.md` — método de análise dos projetos a absorver + arquitetura-alvo em texto.
- `/home/nmaldaner/projetos/imkt5/doc/architecture.md` — arquitetura canônica detalhada.
- `/home/nmaldaner/projetos/imkt5/doc/provider-tuning.md` — parâmetros/prompts por modelo (steps, cfg, estilo de prompt).
- `/home/nmaldaner/.claude/plans/steady-wobbling-fountain.md` — plano aprovado que originou este scaffold.
