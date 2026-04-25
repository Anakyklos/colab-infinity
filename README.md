# Colab Infinity

> **Poder computacional gratuito e contínuo para agentes autônomos.**
> Transforma o Google Colab gratuito em um servidor de LLM resiliente, com rotação automática de contas e compatibilidade total com a API OpenAI.

---

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![API: OpenAI Compatible](https://img.shields.io/badge/API-OpenAI%20Compatible-green.svg)](https://platform.openai.com/docs/api-reference)
[![Status: Beta](https://img.shields.io/badge/status-beta-orange.svg)]()

---

## O que é o Colab Infinity?

O **Colab Infinity** é um módulo de infraestrutura de LLM projetado para fornecer capacidade de inferência **gratuita, contínua e resiliente** ao ecossistema de agentes autônomos — em especial o [Ouroboros Runtime](https://github.com/RenyEnnos/ouroboros-runtime), o Hermes Agent e o OpenClaw.

Em vez de depender de APIs pagas (OpenAI, Anthropic) durante o desenvolvimento, o Colab Infinity transforma o **Google Colab gratuito** — com sua GPU T4 de 16 GB — em um servidor de modelos de linguagem de disponibilidade contínua, usando um pool de múltiplas contas Google com rotação automática e checkpointing no Google Drive.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         COLAB INFINITY                               │
│                                                                      │
│   Agentes Autônomos          Infraestrutura LLM Gratuita             │
│   ─────────────────          ─────────────────────────               │
│                                                                      │
│   Ouroboros Runtime ──┐                                              │
│   Hermes Agent      ──┼──► Proxy Local ──► ngrok ──► Google Colab   │
│   OpenClaw          ──┘    (127.0.0.1)              (GPU T4 + LLM)  │
│   Qualquer cliente         :11434                                    │
│   OpenAI-compat.                                                     │
│                         Rotação Automática de Contas                 │
│                         + Checkpoint no Google Drive                 │
└──────────────────────────────────────────────────────────────────────┘
```

### Principais características

- **API compatível com OpenAI** — qualquer agente que fale `POST /v1/chat/completions` funciona sem modificações
- **Rotação automática de contas** — pool de contas Google com troca transparente quando a cota expira (~12h)
- **Checkpointing no Drive** — estado persistido no Google Drive; zero perda de contexto entre sessões
- **Proxy local transparente** — os agentes apontam sempre para `http://127.0.0.1:11434`; o orquestrador cuida do resto
- **Tolerância a falhas** — MTTR ≤ 8 minutos; uptime alvo ≥ 95%
- **Custo zero** — Google Colab Free + ngrok Free + Google Drive Free

---

## Início Rápido (3 passos)

### Pré-requisitos

- 3+ contas Google com acesso ao Colab e verificação por telefone
- 1 conta ngrok por conta Google (plano free)
- Python 3.10+ e pip instalados localmente

### Passo 1 — Clonar e instalar

```bash
git clone https://github.com/RenyEnnos/colab-infinity.git
cd colab-infinity
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### Passo 2 — Configurar

```bash
mkdir -p ~/.colab_infinity/logs
cp config/colab_infinity_config.yaml.example ~/.colab_infinity/colab_infinity_config.yaml
# Edite o arquivo com suas contas e tokens ngrok
nano ~/.colab_infinity/accounts.json
nano ~/.colab_infinity/colab_infinity_config.yaml
```

### Passo 3 — Iniciar

```bash
# 1. Iniciar o orquestrador local
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml &

# 2. Abrir o notebook no Google Colab e executar todas as células:
#    colab_infinity/notebooks/colab_server.ipynb

# 3. Verificar que está funcionando
curl -s http://127.0.0.1:11434/health | python3 -m json.tool
```

**Pronto.** Seu servidor LLM está disponível em `http://127.0.0.1:11434/v1`.

---

## Integração com o Ouroboros Runtime

Adicione ao arquivo `.env` do Ouroboros Runtime — **zero modificação de código**:

```dotenv
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=dummy
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2
LLM_TIMEOUT_MS=120000
LLM_MAX_RETRIES=3
LLM_STREAM=true
```

Consulte o [Guia de Integração completo](docs/09_integration_guide.md) para exemplos em TypeScript, tratamento de erros e configuração por agente do Conselho.

---

## Documentação

| Documento | Descrição |
|---|---|
| [01 — Project Charter](docs/01_project_charter.md) | Visão, escopo, objetivos e métricas de sucesso |
| [02 — SRS](docs/02_srs.md) | Especificação completa de requisitos (RF01–RF28) |
| [03 — Arquitetura (SAD)](docs/03_sad.md) | Componentes, fluxos e decisões arquiteturais |
| [04 — API Spec](docs/04_api_spec.md) | Contratos de API, exemplos com curl e TypeScript |
| [05 — Setup Guide](docs/05_setup_guide.md) | Instalação passo a passo do zero |
| [06 — Runbook](docs/06_runbook.md) | Operação diária, falhas comuns e recuperação |
| [07 — Test Plan](docs/07_test_plan.md) | Casos de teste e critérios de aceite |
| [08 — Risk Analysis](docs/08_risk_analysis.md) | Riscos técnicos, operacionais e legais |
| [09 — Integration Guide](docs/09_integration_guide.md) | Integração detalhada com o Ouroboros Runtime |
| [10 — Deploy Checklist](docs/10_checklist.md) | Verificação antes de iniciar operação contínua |

---

## Aviso Legal

> ⚠️ Este projeto utiliza o Google Colab, Google Drive e ngrok de acordo com seus planos gratuitos.
> Você é responsável pelo cumprimento dos Termos de Serviço de cada plataforma.
> O uso comercial intensivo pode violar os ToS do Google Colab. **Use por sua conta e risco.**
> Não envie dados sensíveis ou confidenciais através deste sistema.

## Licença

[MIT License](LICENSE) — Copyright (c) 2025 Colab Infinity Contributors.