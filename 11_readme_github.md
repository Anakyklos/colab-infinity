# Colab Infinity

> **Poder computacional gratuito e contínuo para agentes autônomos.**  
> Transforma o Google Colab gratuito em um servidor de LLM resiliente, com rotação automática de contas e compatibilidade total com a API OpenAI.

---

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![Bun 1.1+](https://img.shields.io/badge/bun-1.1%2B-black.svg)](https://bun.sh)
[![API: OpenAI Compatible](https://img.shields.io/badge/API-OpenAI%20Compatible-green.svg)](https://platform.openai.com/docs/api-reference)
[![Status: Beta](https://img.shields.io/badge/status-beta-orange.svg)]()

---

## O que é o Colab Infinity?

O **Colab Infinity** é um módulo de infraestrutura de LLM (Large Language Model) projetado para fornecer capacidade de inferência **gratuita, contínua e resiliente** ao ecossistema de agentes autônomos.

Em vez de depender de APIs pagas (OpenAI, Anthropic) durante o desenvolvimento, o Colab Infinity transforma o **Google Colab gratuito** — com sua GPU T4 de 16 GB — em um servidor de modelos de linguagem de disponibilidade contínua.

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
│   OpenAI-compat.           ▲                                         │
│                            │  Rotação Automática de Contas           │
│                            └  + Checkpoint no Google Drive           │
└──────────────────────────────────────────────────────────────────────┘
```

### Principais características

- **API compatível com OpenAI** — qualquer agente que fale `POST /v1/chat/completions` funciona sem modificações
- **Rotação automática de contas** — pool de contas Google com troca transparente quando a cota expira
- **Checkpointing no Drive** — estado persistido no Google Drive; zero perda de contexto entre sessões
- **Proxy local transparente** — os agentes apontam sempre para `http://127.0.0.1:11434`; o orquestrador cuida do resto
- **Tolerância a falhas** — MTTR ≤ 8 minutos; uptime alvo ≥ 95%
- **Custo zero** — Google Colab free + ngrok free + Google Drive free

---

## Para quem é este projeto?

| Usuário | Benefício |
|---|---|
| **Desenvolvedor do Ouroboros Runtime** | LLM disponível 24/7 sem custo para o daemon de agentes |
| **Usuário do Hermes Agent** | Provider de LLM local e gratuito em vez da API OpenAI |
| **Usuário do OpenClaw** | Backend de inferência configurável e de custo zero |
| **Qualquer dev com agente OpenAI-compatible** | Servidor LLM gratuito com 1 linha de configuração |

---

## Início Rápido (3 passos)

### Pré-requisitos

- 3 contas Google com acesso ao Google Colab
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
# Criar estrutura de configuração
mkdir -p ~/.colab_infinity/logs

# Copiar template de configuração
cp config/colab_infinity_config.yaml.example \
   ~/.colab_infinity/colab_infinity_config.yaml

# Editar com suas contas e tokens ngrok
nano ~/.colab_infinity/accounts.json
nano ~/.colab_infinity/colab_infinity_config.yaml
```

### Passo 3 — Iniciar

```bash
# 1. Iniciar o orquestrador local
python3 -m colab_infinity.orchestrator \
  --config ~/.colab_infinity/colab_infinity_config.yaml &

# 2. Abrir o notebook no Google Colab e executar todas as células
#    (colab_infinity/notebooks/colab_server.ipynb)

# 3. Verificar que está funcionando
curl -s http://127.0.0.1:11434/health | python3 -m json.tool
```

**Pronto.** Seu servidor LLM está disponível em `http://127.0.0.1:11434/v1`.

---

## Integração com o Ouroboros Runtime

Adicione ao arquivo `.env` do Ouroboros Runtime:

```dotenv
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=dummy
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2
LLM_TIMEOUT_MS=120000
LLM_MAX_RETRIES=3
LLM_STREAM=true
```

Nenhuma modificação de código é necessária no Daemon ou nos agentes.

### Teste rápido de integração

```bash
# Verificar disponibilidade
curl -s http://127.0.0.1:11434/health \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('OK:', d['model']['id'])"

# Testar inferência
curl -s -X POST http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Olá, Colab Infinity!"}],
    "max_tokens": 50
  }' | python3 -c \
  "import sys,json; r=json.load(sys.stdin); \
   print(r['choices'][0]['message']['content'])"
```

---

## Modelos Suportados

| Modelo | Parâmetros | VRAM (4-bit) | GPU Mínima |
|---|---|---|---|
| `mistralai/Mistral-7B-Instruct-v0.2` | 7B | ~5 GB | T4 (Free) |
| `meta-llama/Meta-Llama-3-8B-Instruct` | 8B | ~5.5 GB | T4 (Free) |
| `microsoft/Phi-3-mini-4k-instruct` | 3.8B | ~3 GB | T4 (Free) |
| `Qwen/Qwen2-7B-Instruct` | 7B | ~5 GB | T4 (Free) |
| `mistralai/Mixtral-8x7B-Instruct-v0.1` | 47B (MoE) | ~24 GB | A100 (Pro+) |

---

## Documentação Completa

| Documento | Descrição |
|---|---|
| [01 — Project Charter](docs/01_project_charter.md) | Visão, escopo e métricas de sucesso |
| [02 — SRS](docs/02_srs.md) | Especificação completa de requisitos (RF01–RF28) |
| [03 — Arquitetura (SAD)](docs/03_sad.md) | Componentes, fluxos e decisões arquiteturais |
| [04 — API Spec](docs/04_api_spec.md) | Contratos de request/response, exemplos com curl e TypeScript |
| [05 — Setup Guide](docs/05_setup_guide.md) | Instalação passo a passo do zero |
| [06 — Runbook](docs/06_runbook.md) | Manual de operação, falhas comuns e recuperação |
| [07 — Test Plan](docs/07_test_plan.md) | Casos de teste, critérios de aceite e métricas |
| [08 — Risk Analysis](docs/08_risk_analysis.md) | Riscos técnicos, operacionais e legais com mitigações |
| [09 — Integration Guide](docs/09_integration_guide.md) | Integração detalhada com o Ouroboros Runtime |
| [10 — Deploy Checklist](docs/10_checklist.md) | Lista de verificação antes de iniciar operação contínua |

---

## Arquitetura em uma Imagem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MÁQUINA LOCAL                                                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  AGENTES CONSUMIDORES (Ouroboros, Hermes, OpenClaw, etc.)             │  │
│  └─────────────────────────────┬────────────────────────────────────────┘  │
│                                 │  POST /v1/chat/completions                │
│  ┌─────────────────────────────▼────────────────────────────────────────┐  │
│  │  ORQUESTRADOR (orchestrator.py)                                       │  │
│  │  ├── Proxy Local :11434                                               │  │
│  │  ├── Health Monitor (polling /health a cada 30s)                      │  │
│  │  ├── Account Switcher (troca automática ao detectar falha)            │  │
│  │  ├── Checkpoint Manager (salva estado no Drive a cada 5 min)         │  │
│  │  └── Pool Manager (gerencia N contas Google)                          │  │
│  └─────────────────────────────┬────────────────────────────────────────┘  │
│                                 │  HTTPS via ngrok                          │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │
           ┌──────────────────────▼──────────────────────┐
           │  GOOGLE COLAB (sessão ativa)                  │
           │  ├── FastAPI Server :8000                      │
           │  ├── LLM Engine (Transformers + 4-bit GPU)    │
           │  ├── ngrok Tunnel                              │
           │  └── Quota Monitor + Checkpoint Periódico     │
           └──────────────────────┬──────────────────────┘
                                  │  Google Drive API
           ┌──────────────────────▼──────────────────────┐
           │  CONTA ARMAZÉM (Google Drive)                 │
           │  ├── checkpoints/ci_ckpt_*.json               │
           │  ├── pool_state/ngrok_url.json                 │
           │  ├── pool_state/pool_state.json                │
           │  └── logs/metrics.jsonl                        │
           └─────────────────────────────────────────────┘
```

---

## Estrutura do Repositório

```
colab-infinity/
├── README.md
├── requirements.txt
├── colab_infinity/             ← Pacote Python do orquestrador
│   ├── __init__.py
│   ├── orchestrator.py         ← Entry point principal
│   ├── proxy.py                ← Proxy local HTTP
│   ├── monitor.py              ← Health checker
│   ├── switcher.py             ← Account switcher
│   ├── pool.py                 ← Pool manager
│   ├── checkpoint.py           ← Checkpoint manager
│   ├── drive.py                ← Google Drive client
│   └── cli.py                  ← Interface de linha de comando
├── notebooks/
│   └── colab_server.ipynb      ← Notebook do servidor LLM (executar no Colab)
├── config/
│   ├── colab_infinity_config.yaml.example
│   └── accounts.json.example
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── stress/
└── docs/                       ← Documentação completa (11 documentos)
```

---

## Perguntas Frequentes

**P: Preciso pagar pelo Google Colab Pro?**  
R: Não. O Colab Infinity foi projetado para funcionar inteiramente com o plano **gratuito** do Google Colab (GPU T4, ~12h por sessão). O Colab Pro aumenta o tempo de sessão e pode melhorar a disponibilidade de GPU, mas não é necessário.

**P: Quantas contas Google preciso?**  
R: Mínimo de 3 contas operacionais + 1 conta armazém. Com 3 contas, a rotação cobre ~36h de GPU antes de aguardar o cooldown de 24h. Recomendamos 5+ contas para operação mais confortável.

**P: O Colab Infinity viola os Termos de Serviço do Google?**  
R: O uso do Colab para desenvolvimento pessoal e pesquisa não-comercial é permitido. O uso de automação para simular comportamento humano e o uso comercial intensivo são as principais áreas de risco. Consulte `docs/08_risk_analysis.md` para a análise completa de riscos legais e as mitigações recomendadas. **Use por sua conta e risco.**

**P: Posso usar com qualquer modelo do HuggingFace?**  
R: Sim, qualquer modelo que caiba na GPU T4 (≤15 GB de VRAM com quantização 4-bit) e seja compatível com `transformers.AutoModelForCausalLM` pode ser usado. A tabela de modelos suportados é um guia, não uma lista fechada.

**P: O Ouroboros Runtime precisa de modificações para usar o Colab Infinity?**  
R: Não. Basta configurar as variáveis de ambiente `LLM_BASE_URL` e `LLM_MODEL` no arquivo `.env` do Ouroboros. Veja `docs/09_integration_guide.md` para o guia completo.

**P: O que acontece durante uma troca de conta?**  
R: O proxy local retorna `503 SESSION_SWITCHING` temporariamente. Os agentes consumidores devem implementar retry com backoff (recomendado: aguardar 90s). O tempo médio de troca (MTTR) é de 4–8 minutos.

---

## Contribuindo

Contribuições são bem-vindas! Por favor:

1. Faça um fork do repositório
2. Crie uma branch: `git checkout -b feature/minha-feature`
3. Execute os testes: `pytest tests/unit/ tests/integration/ -v`
4. Verifique a formatação: `black colab_infinity/ && isort colab_infinity/`
5. Abra um Pull Request descrevendo as mudanças

Para bugs ou dúvidas, abra uma [Issue](https://github.com/RenyEnnos/colab-infinity/issues).

---

## Aviso Legal

> ⚠️ **IMPORTANTE — LEIA ANTES DE USAR**

O Colab Infinity utiliza recursos do **Google Colab**, **Google Drive** e **ngrok** de acordo com seus planos gratuitos. Ao usar este software, você concorda com os seguintes pontos:

1. **Responsabilidade de uso:** Você é o único responsável pelo uso deste software e pelo cumprimento dos Termos de Serviço do Google Colab, Google Drive e ngrok.

2. **Sem garantias:** Este software é fornecido "como está" (AS-IS), sem garantias de disponibilidade, desempenho ou conformidade com qualquer TOS de terceiros.

3. **Uso não comercial:** Este projeto destina-se exclusivamente a uso pessoal, educacional e de pesquisa. O uso comercial intensivo dos recursos gratuitos do Google Colab pode violar os Termos de Serviço do Google.

4. **Risco de banimento:** O uso excessivo ou automatizado das contas Google pode resultar em suspensão das contas. O projeto não se responsabiliza por contas banidas.

5. **Dados e privacidade:** Prompts enviados ao servidor Colab transitam pela infraestrutura do ngrok e do Google. Não envie dados sensíveis, confidenciais ou pessoais através deste sistema.

**Use com consciência e moderação.**

---

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

```
MIT License

Copyright (c) 2025 Colab Infinity Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

*Feito com ☕ e GPUs gratuitas do Google.*