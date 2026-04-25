# Colab Infinity — Documento de Arquitetura de Software (SAD)

**Versão:** 1.0.0  
**Data:** 2025-07-14  
**Status:** Aprovado  
**Referências:** `01_project_charter.md`, `02_srs.md`

---

## Índice

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Diagrama de Componentes](#2-diagrama-de-componentes)
3. [Descrição Detalhada dos Componentes](#3-descrição-detalhada-dos-componentes)
4. [Integração com o Ouroboros Runtime](#4-integração-com-o-ouroboros-runtime)
5. [Modelo de Dados](#5-modelo-de-dados)
6. [Fluxos de Operação](#6-fluxos-de-operação)
7. [Diagramas de Sequência](#7-diagramas-de-sequência)
8. [Decisões Arquiteturais (ADR)](#8-decisões-arquiteturais-adr)
9. [Considerações de Segurança](#9-considerações-de-segurança)
10. [Considerações de Escalabilidade](#10-considerações-de-escalabilidade)

---

## 1. Visão Geral da Arquitetura

O Colab Infinity é uma infraestrutura de LLM baseada em quatro camadas funcionais que transformam
o Google Colab gratuito em um servidor de modelos de linguagem resiliente e contínuo.

### 1.1 Camadas Arquiteturais

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CAMADA 1 — CONSUMIDORES                                                     │
│  Agentes que consomem a API OpenAI-compatible                                │
│  [ Ouroboros Runtime ] [ Hermes Agent ] [ OpenClaw ] [ Qualquer Cliente ]   │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │  HTTP / OpenAI API Protocol
┌───────────────────────────────▼─────────────────────────────────────────────┐
│  CAMADA 2 — PROXY E ROTEAMENTO                                               │
│  Orquestrador Local (Python)                                                 │
│  [ Proxy Reverso :8081 ] [ Health Monitor ] [ Account Switcher ]            │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │  HTTPS via ngrok tunnel
┌───────────────────────────────▼─────────────────────────────────────────────┐
│  CAMADA 3 — COMPUTAÇÃO (Google Colab)                                        │
│  [ FastAPI Server :8000 ] [ LLM Engine ] [ ngrok Tunnel ] [ Quota Monitor ] │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │  Google Drive API
┌───────────────────────────────▼─────────────────────────────────────────────┐
│  CAMADA 4 — PERSISTÊNCIA                                                     │
│  Conta Armazém (Google Drive)                                                │
│  [ checkpoints/ ] [ pool_state.json ] [ ngrok_url.json ] [ logs/ ]          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Princípios Arquiteturais

| Princípio                  | Aplicação no Colab Infinity                                                     |
|----------------------------|---------------------------------------------------------------------------------|
| **Compatibilidade OpenAI** | A API exposta é um subconjunto da OpenAI Chat Completions API, permitindo zero‑reconfiguração nos agentes consumidores |
| **Resiliência por Pool**   | Múltiplas contas Google funcionam como réplicas de failover; a falha de uma não interrompe o serviço |
| **Estado Externalizado**   | Todo estado crítico reside no Google Drive (conta armazém), desacoplando o ciclo de vida das sessões Colab do estado do sistema |
| **Transparência de Troca** | O proxy local absorve a troca de conta, tornando‑a invisível para os agentes consumidores |
| **Observabilidade**        | Logs estruturados em JSON Lines, métricas de saúde via `/health` e `/v1/status` |

---

## 2. Diagrama de Componentes

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                         COLAB INFINITY — VISÃO COMPLETA                         ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────────────────┐    ║
║  │                        AGENTES CONSUMIDORES                              │    ║
║  │                                                                           │    ║
║  │  ┌────────────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │    ║
║  │  │   Ouroboros Runtime    │  │   Hermes Agent   │  │    OpenClaw     │  │    ║
║  │  │                        │  │                  │  │                 │  │    ║
║  │  │  Daemon (Bun/TS)       │  │  Python Agent    │  │  Agent Client   │  │    ║
║  │  │  ├─ Wave Orchestrator  │  │  ├─ Chat Client  │  │  ├─ LLM Client  │  │    ║
║  │  │  ├─ Agent Council      │  │  └─ Tool Runner  │  │  └─ Tool Use    │  │    ║
║  │  │  └─ RPC Bus            │  │                  │  │                 │  │    ║
║  │  └───────────┬────────────┘  └────────┬─────────┘  └───────┬─────────┘  │    ║
║  └─────────────┼──────────────────────────┼────────────────────┼────────────┘    ║
║                │  POST /v1/chat/completions│                    │                ║
║                │  (OpenAI-compatible)      │                    │                ║
║                └──────────────┬────────────┘────────────────────┘                ║
║                               │  http://127.0.0.1:8081                           ║
║  ┌────────────────────────────▼──────────────────────────────────────────────┐   ║
║  │                     ORQUESTRADOR LOCAL (orchestrator.py)                   │   ║
║  │                                                                             │   ║
║  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────────┐  │   ║
║  │  │  Proxy Reverso  │  │  Health Monitor │  │    Account Switcher      │  │   ║
║  │  │  :8081          │  │  (polling /     │  │                          │  │   ║
║  │  │  (httpx-based)  │  │   health, 30s)  │  │  ┌──────────────────┐   │  │   ║
║  │  └────────┬────────┘  └────────┬────────┘  │  │  Pool Manager    │   │  │   ║
║  │           │                    │            │  │  (pool_state.json│   │  │   ║
║  │  ┌────────▼────────────────────▼────────┐  │  └──────────────────┘   │  │   ║
║  │  │         State Machine                │  │  ┌──────────────────┐   │  │   ║
║  │  │  IDLE→STARTING→ACTIVE→FAILING→       │  │  │ Checkpoint Mgr   │   │  │   ║
║  │  │  SWITCHING→RECOVERING→ACTIVE         │  │  └──────────────────┘   │  │   ║
║  │  └──────────────────────────────────────┘  └──────────────────────────┘  │   ║
║  └────────────────────────────┬──────────────────────────────────────────────┘   ║
║                               │  HTTPS (ngrok public URL)                         ║
║  ┌────────────────────────────▼──────────────────────────────────────────────┐   ║
║  │                   GOOGLE COLAB SESSION (colab_server.ipynb)               │   ║
║  │                                                                             │   ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐  │   ║
║  │  │   NGROK TUNNEL                                                       │  │   ║
║  │  │   https://<hash>.ngrok-free.app  ──►  http://127.0.0.1:8000         │  │   ║
║  │  └─────────────────────────────────────────────────────────────────────┘  │   ║
║  │                                                                             │   ║
║  │  ┌─────────────────────┐  ┌──────────────────────────────────────────┐   │   ║
║  │  │   FastAPI Server    │  │         LLM ENGINE                        │   │   ║
║  │  │   :8000             │  │                                            │   │   ║
║  │  │  ├─ /health         │  │  ┌────────────────────────────────────┐  │   │   ║
║  │  │  ├─ /v1/chat/       │  │  │  Model (Mistral-7B / Llama-3)      │  │   │   ║
║  │  │  │   completions    │  │  │  Quantization: 4-bit (BitsAndBytes) │  │   │   ║
║  │  │  ├─ /v1/status      │◄─┤  │  Device: CUDA (T4 GPU)             │  │   │   ║
║  │  │  └─ /v1/checkpoint  │  │  └────────────────────────────────────┘  │   │   ║
║  │  └─────────────────────┘  │                                            │   │   ║
║  │                            │  ┌────────────────────────────────────┐  │   │   ║
║  │  ┌─────────────────────┐  │  │  Quota Monitor (tquota + timer)    │  │   │   ║
║  │  │  Checkpoint Manager │  │  └────────────────────────────────────┘  │   │   ║
║  │  │  (Drive API write)  │  └──────────────────────────────────────────┘   │   ║
║  │  └─────────────────────┘                                                  │   ║
║  └────────────────────────────┬──────────────────────────────────────────────┘   ║
║                               │  Google Drive API (OAuth2)                        ║
║  ┌────────────────────────────▼──────────────────────────────────────────────┐   ║
║  │                     CONTA ARMAZÉM (Google Drive)                           │   ║
║  │                                                                             │   ║
║  │  hermes_infinito/ (ou colab_infinity/)                                     │   ║
║  │  ├── checkpoints/                                                          │   ║
║  │  │   └── checkpoint_20250714_031500.json                                   │   ║
║  │  ├── pool_state/                                                           │   ║
║  │  │   ├── pool_state.json                                                   │   ║
║  │  │   └── ngrok_url.json                                                    │   ║
║  │  ├── notebooks/                                                            │   ║
║  │  │   └── colab_server.ipynb                                                │   ║
║  │  └── logs/                                                                 │   ║
║  │      └── metrics.jsonl                                                     │   ║
║  └────────────────────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 3. Descrição Detalhada dos Componentes

### 3.1 Notebook Colab (`colab_server.ipynb`)

**Responsabilidade:** Executar o modelo LLM em hardware GPU gratuito e expor a API de inferência.

**Tecnologias:**
- `FastAPI` 0.111+ — servidor HTTP ASGI de alta performance
- `uvicorn` — servidor ASGI com suporte a HTTP/1.1 e HTTP/2
- `transformers` 4.42+ — carregamento e inferência de modelos HuggingFace
- `bitsandbytes` 0.43+ — quantização 4-bit (NF4) e 8-bit
- `accelerate` 0.31+ — mapeamento automático de dispositivos (`device_map="auto"`)
- `pyngrok` 7.1+ — criação e gerenciamento do túnel ngrok
- `google-colab` — montagem do Google Drive (conta armazém)

**Ciclo de Vida da Sessão:**

```
[Início da sessão Colab]
        │
        ▼
[Verificar GPU disponível] ──── Falha ────► [Raise RuntimeError — abortar]
        │
        ▼
[Instalar dependências (pip)]
        │
        ▼
[Montar Google Drive da conta armazém]
        │
        ▼
[Carregar modelo com quantização 4-bit]  (~3-5 min para Mistral-7B)
        │
        ▼
[Inicializar FastAPI + Uvicorn em thread daemon]
        │
        ▼
[Criar túnel ngrok e publicar URL no Drive]
        │
        ▼
[Iniciar Quota Monitor (tquota polling)]
        │
        ▼
[Loop de keepalive — executa até sessão expirar ou checkpoint ser solicitado]
        │
        ├── A cada CHECKPOINT_INTERVAL_SECONDS:
        │       └── [Salvar checkpoint.json no Drive]
        │
        └── Quando QUOTA_WARNING_THRESHOLD atingido:
                └── [Salvar checkpoint final → sinalizar orquestrador → encerrar]
```

**Modelos Suportados (referência):**

| Modelo                            | VRAM (4-bit) | Tokens/s (T4) | Notas                       |
|-----------------------------------|-------------|---------------|-----------------------------|
| `mistralai/Mistral-7B-Instruct-v0.2` | ~5 GB    | ~25–35        | Recomendado — equilíbrio ideal |
| `meta-llama/Meta-Llama-3-8B-Instruct`| ~5.5 GB  | ~20–30        | Alta qualidade de instrução  |
| `google/gemma-2-9b-it`           | ~6 GB       | ~18–25        | Boa performance multilíngue  |
| `Qwen/Qwen2-7B-Instruct`         | ~5 GB       | ~22–32        | Forte em código              |

---

### 3.2 Túnel ngrok

**Responsabilidade:** Criar um canal HTTPS público para expor a porta 8000 do Colab à internet.

**Funcionamento:**

```
[Internet / Orquestrador Local]
        │
        │  HTTPS  https://a1b2c3d4.ngrok-free.app
        ▼
[ngrok Edge Server (cloud)]
        │
        │  Túnel TCP encriptado
        ▼
[ngrok Agent (dentro do Colab)]
        │
        │  HTTP  127.0.0.1:8000
        ▼
[FastAPI Server]
```

**Limitações do Plano Free:**
- 1 túnel ativo simultâneo por conta ngrok
- URL pública aleatória a cada nova sessão (muda a cada reinício)
- Sem domínio fixo (disponível apenas no plano pago)
- 20.000 conexões/mês (suficiente para uso pessoal de agentes)

**Propagação da URL:**
Após a criação do túnel, o notebook escreve o URL em `colab_infinity/pool_state/ngrok_url.json`
no Google Drive da conta armazém. O orquestrador local faz polling nesse arquivo a cada 10 segundos
durante a inicialização.

---

### 3.3 Orquestrador Local (`orchestrator.py`)

**Responsabilidade:** Gerenciar o ciclo de vida das sessões Colab, manter o proxy local ativo e
executar a troca automática de contas quando necessário.

**Máquina de Estados:**

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │
          ┌─────────────────┐                                      │
    ───►  │      IDLE       │  ◄── Estado inicial / após reset     │
          └────────┬────────┘                                      │
                   │  start()                                      │
                   ▼                                               │
          ┌─────────────────┐                                      │
          │    STARTING     │  ◄── Aguarda notebook subir          │
          └────────┬────────┘      (polling /health, timeout 300s) │
                   │  health OK                                    │
                   ▼                                               │
          ┌─────────────────┐                                      │
          │     ACTIVE      │  ◄── Operação normal; proxy ativo    │
          └────────┬────────┘                                      │
                   │  N falhas consecutivas no health check        │
                   ▼                                               │
          ┌─────────────────┐                                      │
          │     FAILING     │  ◄── Threshold atingido; inicia SLA  │
          └────────┬────────┘                                      │
                   │  switch_account()                             │
                   ▼                                               │
          ┌─────────────────┐                                      │
          │   SWITCHING     │  ◄── Salva checkpoint, troca conta   │
          └────────┬────────┘                                      │
                   │  nova sessão iniciada                         │
                   ▼                                               │
          ┌─────────────────┐                                      │
          │   RECOVERING    │  ◄── Aguarda novo servidor subir     │
          └────────┬────────┘                                      │
                   └──────────────────────────────────────────────┘
                     volta para ACTIVE
```

**Componentes Internos:**

| Componente          | Arquivo / Classe            | Responsabilidade                                                 |
|---------------------|-----------------------------|------------------------------------------------------------------|
| Proxy Reverso       | `proxy.py::LocalProxy`      | Forward de todas as requisições para a URL ngrok ativa           |
| Health Monitor      | `monitor.py::HealthMonitor` | Polling `/health` a cada 30s; atualiza estado da máquina         |
| Account Switcher    | `switcher.py::AccountSwitcher`| Seleciona próxima conta, inicia nova sessão Colab, atualiza proxy|
| Pool Manager        | `pool.py::AccountPool`      | Gerencia lista de contas, estados (available/exhausted/banned)   |
| Checkpoint Manager  | `checkpoint.py::CheckpointMgr`| Lê/escreve `checkpoint.json` no Drive                          |
| Drive Client        | `drive.py::DriveClient`     | Wrapper da Google Drive API (OAuth2, retry com backoff)          |

---

### 3.4 Conta Armazém (Google Drive)

**Responsabilidade:** Armazenar todo o estado persistente do sistema, visível tanto pelo notebook
Colab quanto pelo orquestrador local.

**Estrutura de Diretórios:**

```
Meu Drive/
└── colab_infinity/
    ├── checkpoints/
    │   ├── checkpoint_20250714_031500.json   ← Estado completo
    │   ├── checkpoint_20250714_033000.json
    │   └── ...  (retém os últimos MAX_CHECKPOINTS arquivos)
    ├── pool_state/
    │   ├── pool_state.json    ← Estado atual do pool de contas
    │   └── ngrok_url.json     ← URL pública atual do ngrok
    ├── notebooks/
    │   └── colab_server.ipynb ← Cópia versionada do notebook
    ├── config/
    │   └── colab_infinity_config.yaml  ← Config compartilhada (sem credenciais)
    └── logs/
        └── metrics.jsonl      ← Métricas exportadas (append-only)
```

**Padrão de Acesso:**

| Quem escreve             | Quem lê                  | Arquivo                   |
|--------------------------|--------------------------|---------------------------|
| Notebook Colab           | Orquestrador local       | `ngrok_url.json`          |
| Notebook Colab           | Orquestrador local       | `checkpoint_*.json`       |
| Orquestrador local       | Notebook Colab (startup) | `pool_state.json`         |
| Orquestrador local       | Notebook Colab (startup) | `colab_infinity_config.yaml` |
| Notebook Colab           | Orquestrador local       | `metrics.jsonl`           |

---

### 3.5 Agentes Consumidores

#### Ouroboros Runtime

O Ouroboros Runtime é um daemon persistente escrito em **Bun/TypeScript** que orquestra um conselho
de agentes especializados (Core, Vision, Architect, Guardian, Kinetic). Cada agente realiza chamadas
LLM via um módulo de cliente configurável.

**Integração:** O Daemon do Ouroboros aponta seu `LLM_BASE_URL` para `http://127.0.0.1:8081/v1`,
onde o orquestrador local do Colab Infinity mantém o proxy transparente. Nenhuma modificação
no código dos agentes é necessária — o protocolo é idêntico ao da OpenAI API.

**Fluxo interno do Ouroboros:**

```
Wave Trigger
    │
    ▼
Wave Orchestrator seleciona agente do Conselho
    │
    ▼
Agente constrói o prompt (system + contexto + instrução)
    │
    ▼
LLM Client → POST http://127.0.0.1:8081/v1/chat/completions
    │
    ▼
[Colab Infinity processa e retorna]
    │
    ▼
Protocolo Anti-Vibe: resposta passa pelos gates de qualidade
(especificação → validação → revisão humana → testes)
    │
    ▼
Resultado promovido para o próximo estágio da Wave
```

#### Hermes Agent

Agente Python de uso geral configurável via `hermes_config.yaml`. Aponta `llm.base_url` para
`http://127.0.0.1:8081/v1` e consome a API de forma transparente.

#### OpenClaw

Cliente de agente compatível com OpenAI API. Configuração análoga: substituir a base URL da OpenAI
pela URL do proxy local do Colab Infinity.

---

## 4. Integração com o Ouroboros Runtime

### 4.1 Configuração de Ambiente

O Ouroboros Runtime lê sua configuração de LLM de variáveis de ambiente ou de arquivo `.env`
na raiz do projeto:

```dotenv
# .env do Ouroboros Runtime
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=http://127.0.0.1:8081/v1
LLM_API_KEY=dummy                          # Colab Infinity aceita qualquer valor se auth desabilitada
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2
LLM_TIMEOUT_MS=120000
LLM_MAX_RETRIES=3
LLM_STREAM=true
```

### 4.2 Fluxo de Chamada no Daemon (TypeScript — Ilustrativo)

```typescript
// Exemplo ilustrativo — não é o código final do Ouroboros Runtime
// Mostra o padrão de chamada que o Daemon usa internamente

interface LLMRequest {
  model: string;
  messages: Array<{ role: string; content: string }>;
  temperature?: number;
  max_tokens?: number;
  stream?: boolean;
}

// O cliente LLM do Ouroboros é configurado com a base URL do Colab Infinity
const llmClient = new OpenAICompatibleClient({
  baseURL: process.env.LLM_BASE_URL,      // http://127.0.0.1:8081/v1
  apiKey: process.env.LLM_API_KEY,
  timeout: parseInt(process.env.LLM_TIMEOUT_MS ?? "120000"),
});

// Chamada dentro de uma Wave — ex: agente Architect gerando especificação
async function callAgentLLM(agentName: string, messages: Message[]): Promise<string> {
  const response = await llmClient.chat.completions.create({
    model: process.env.LLM_MODEL,
    messages,
    temperature: 0.7,
    max_tokens: 2048,
    stream: false,
  });

  return response.choices[0].message.content;
}
```

### 4.3 Compatibilidade com o Protocolo Anti-Vibe

O Protocolo Anti-Vibe do Ouroboros Runtime define gates de qualidade que a resposta do LLM deve
passar antes de ser promovida. O Colab Infinity é neutro a esse protocolo — ele serve como
infraestrutura de inferência e não interfere no conteúdo das respostas.

```
┌───────────────────────────────────────────────────────────────────────┐
│                      PROTOCOLO ANTI-VIBE                               │
│                                                                        │
│  [LLM Response via Colab Infinity]                                     │
│          │                                                             │
│          ▼                                                             │
│  Gate 1: ESPECIFICAÇÃO — resposta estruturada e completa?             │
│          │  sim                                                        │
│          ▼                                                             │
│  Gate 2: VALIDAÇÃO — código/lógica passa nos testes automatizados?    │
│          │  sim                                                        │
│          ▼                                                             │
│  Gate 3: REVISÃO HUMANA — operador aprova via TUI?                    │
│          │  sim                                                        │
│          ▼                                                             │
│  Gate 4: TESTES — suite de testes passa no sandbox Python?            │
│          │  sim                                                        │
│          ▼                                                             │
│  [Código / Artefato promovido para a próxima Wave]                    │
│                                                                        │
│  ► Colab Infinity atua ANTES do Gate 1. É transparente aos demais.   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 5. Modelo de Dados

### 5.1 `checkpoint.json`

Arquivo de estado persistido no Drive a cada `CHECKPOINT_INTERVAL_SECONDS` (padrão: 300).

```json
{
  "schema_version": "1.1",
  "saved_at": "2025-07-14T03:30:00Z",
  "save_reason": "periodic",
  "session": {
    "id": "sess_colab_20250714_031500",
    "started_at": "2025-07-14T03:15:00Z",
    "account_index": 1,
    "account_email_masked": "usr***@gmail.com",
    "ngrok_url": "https://a1b2c3d4.ngrok-free.app",
    "model_id": "mistralai/Mistral-7B-Instruct-v0.2",
    "quantization": "4bit",
    "gpu_name": "Tesla T4"
  },
  "pool": {
    "total_accounts": 4,
    "accounts": [
      { "index": 0, "status": "exhausted", "exhausted_at": "2025-07-14T03:14:00Z" },
      { "index": 1, "status": "active",    "activated_at": "2025-07-14T03:15:00Z" },
      { "index": 2, "status": "available", "last_used_at": null },
      { "index": 3, "status": "available", "last_used_at": null }
    ],
    "next_account_index": 2
  },
  "metrics": {
    "requests_served": 312,
    "tokens_generated": 128443,
    "tokens_consumed": 54210,
    "errors_total": 3,
    "uptime_seconds": 900
  },
  "orchestrator": {
    "state": "ACTIVE",
    "proxy_host": "127.0.0.1",
    "proxy_port": 8081,
    "health_failures_consecutive": 0
  }
}
```

**Campos Críticos:**

| Campo                           | Tipo    | Descrição                                                       |
|---------------------------------|---------|-----------------------------------------------------------------|
| `schema_version`                | string  | Versão do schema; verificado ao carregar para detectar migração  |
| `saved_at`                      | string  | ISO 8601 UTC; usado para ordenar checkpoints por recência        |
| `save_reason`                   | string  | `"periodic"`, `"pre_switch"`, `"manual"`, `"quota_warning"`     |
| `session.account_index`         | integer | Índice da conta ativa no pool (0-based)                          |
| `pool.accounts[].status`        | string  | `"available"`, `"active"`, `"exhausted"`, `"banned"`, `"cooldown"` |
| `pool.next_account_index`       | integer | Próxima conta a ser usada em uma troca                           |
| `metrics.requests_served`       | integer | Total de requisições bem-sucedidas nesta sessão                  |
| `orchestrator.state`            | string  | Estado da máquina de estados do orquestrador no momento do save  |

---

### 5.2 `pool_state.json`

Estado mínimo do pool, escrito pelo orquestrador e lido pelo notebook na inicialização.

```json
{
  "schema_version": "1.0",
  "updated_at": "2025-07-14T03:30:00Z",
  "active_account_index": 1,
  "orchestrator_state": "ACTIVE",
  "accounts": [
    {
      "index": 0,
      "status": "exhausted",
      "exhausted_at": "2025-07-14T03:14:00Z",
      "cooldown_until": "2025-07-15T03:14:00Z"
    },
    {
      "index": 1,
      "status": "active",
      "activated_at": "2025-07-14T03:15:00Z"
    },
    {
      "index": 2,
      "status": "available"
    },
    {
      "index": 3,
      "status": "available"
    }
  ]
}
```

---

### 5.3 `ngrok_url.json`

Arquivo de rendez-vous: o notebook escreve após criar o túnel; o orquestrador lê para atualizar o proxy.

```json
{
  "url": "https://a1b2c3d4.ngrok-free.app",
  "session_id": "sess_colab_20250714_031500",
  "account_index": 1,
  "published_at": "2025-07-14T03:16:47Z",
  "model_id": "mistralai/Mistral-7B-Instruct-v0.2",
  "status": "active"
}
```

---

### 5.4 Formato da Mensagem da API (`/v1/chat/completions`)

**Request:**
```json
{
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "messages": [
    { "role": "system",    "content": "Você é um assistente técnico preciso." },
    { "role": "user",      "content": "Explique quantização 4-bit em 2 parágrafos." }
  ],
  "temperature": 0.7,
  "max_tokens": 512,
  "stream": false
}
```

**Response:**
```json
{
  "id": "chatcmpl-sess20250714-0312",
  "object": "chat.completion",
  "created": 1720926130,
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "choices": [{
    "index": 0,
    "message": { "role": "assistant", "content": "Quantização 4-bit..." },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 45,
    "completion_tokens": 189,
    "total_tokens": 234
  },
  "x_session_id": "sess_colab_20250714_031500",
  "x_account_index": 1
}
```

---

## 6. Fluxos de Operação

### 6.1 Fluxo de Operação Normal

```
Agente (ex: Ouroboros/Vision)
        │
        │  1. POST /v1/chat/completions
        │     { messages: [...], max_tokens: 1024 }
        ▼
Proxy Local (:8081)
        │
        │  2. Forward da requisição para URL ngrok ativa
        │     (lida do cache interno; atualizada após cada troca de conta)
        ▼
Túnel ngrok (internet)
        │
        │  3. Encaminhamento para :8000 dentro do Colab
        ▼
FastAPI Server (:8000)
        │
        │  4. Validação de request (Pydantic)
        │  5. Formatação do prompt com chat template do modelo
        ▼
LLM Engine (Transformers + bitsandbytes)
        │
        │  6. Tokenização do prompt
        │  7. model.generate() na GPU T4
        │  8. Decodificação dos tokens gerados
        ▼
FastAPI Server
        │
        │  9.  Construção do response JSON (formato OpenAI)
        │  10. Incremento de _requests_served
        ▼
Túnel ngrok → Proxy Local
        │
        │  11. Resposta retornada ao agente
        ▼
Agente recebe { choices[0].message.content }
```

---

### 6.2 Fluxo de Fallback — Checkpoint e Troca de Conta

```
Health Monitor (orquestrador local)
        │
        │  1. GET /health → 3 falhas consecutivas (ou timeout)
        │     OU Quota Monitor detecta ~10h de sessão
        ▼
Orquestrador: estado ACTIVE → FAILING
        │
        │  2. Proxy local começa a retornar 503 SESSION_SWITCHING
        │     para novas requisições (requisições em andamento aguardam)
        ▼
Orquestrador: FAILING → SWITCHING
        │
        │  3. Salvar checkpoint.json no Drive (save_reason: "pre_switch")
        │  4. Atualizar pool_state.json: conta atual → "exhausted"
        │  5. Selecionar próxima conta disponível (pool.next_account_index)
        ▼
Orquestrador: iniciar nova sessão Colab
        │
        │  6. Abrir colab_server.ipynb na nova conta Google
        │  7. Aguardar notebook executar todas as células (~3-5 min)
        │  8. Polling em ngrok_url.json no Drive (a cada 10s, timeout 300s)
        ▼
Novo notebook publica URL ngrok no Drive
        │
        │  9. ngrok_url.json atualizado com nova URL
        ▼
Orquestrador: SWITCHING → RECOVERING
        │
        │  10. GET /health → sucesso com nova URL
        │  11. Proxy local atualiza URL de destino
        │  12. pool_state.json: nova conta → "active"
        ▼
Orquestrador: RECOVERING → ACTIVE
        │
        │  13. Proxy local volta a aceitar requisições normalmente
        │  14. Requisições em espera são reprocessadas
        ▼
Sistema restaurado — downtime total: geralmente 4–8 minutos
```

---

## 7. Diagramas de Sequência

### 7.1 Cenário A: Requisição Bem-Sucedida

```
Ouroboros      Proxy Local     ngrok Tunnel    FastAPI     LLM Engine     Drive
Runtime        :8081           (internet)      :8000       (GPU T4)       (Armazém)
    │               │               │               │               │           │
    │──POST /v1/──► │               │               │               │           │
    │  chat/        │               │               │               │           │
    │  completions  │               │               │               │           │
    │               │──HTTPS GET──► │               │               │           │
    │               │               │──HTTP POST──► │               │           │
    │               │               │               │──tokenize────►│           │
    │               │               │               │               │           │
    │               │               │               │◄──model.gen()─│           │
    │               │               │               │               │           │
    │               │               │◄──200 JSON────│               │           │
    │               │◄──200 JSON────│               │               │           │
    │◄──200 JSON────│               │               │               │           │
    │               │               │               │               │           │
    │               │               │               │  (async)      │           │
    │               │               │               │───ckpt save──────────────►│
    │               │               │               │               │           │
```

---

### 7.2 Cenário B: Checkpoint e Troca de Conta

```
Health          Orquestrador    Drive           Colab           Proxy
Monitor         (State Machine) (Armazém)       Session N       Local :8081
    │               │               │               │               │
    │──3x timeout──►│               │               │               │
    │               │               │               │               │
    │               │──503 to new──────────────────────────────────►│
    │               │  requests     │               │               │
    │               │               │               │               │
    │               │──write ckpt──►│               │               │
    │               │──update pool──►               │               │
    │               │               │               │               │
    │               │               │         [Session N encerra]   │
    │               │               │               │               │
    │               │   [Abre Colab Session N+1 na conta índice K]   │
    │               │               │               │               │
    │               │◄──poll ngrok──│               │               │
    │               │  url.json     │               │               │
    │               │               │◄──write URL───│ (novo Colab)  │
    │               │◄──nova URL────│               │               │
    │               │               │               │               │
    │◄──GET /health─│               │               │               │
    │  nova URL     │               │               │               │
    │──200 OK──────►│               │               │               │
    │               │──update proxy URL────────────────────────────►│
    │               │──200 ok to new requests──────────────────────►│
    │               │               │               │               │
```

---

## 8. Decisões Arquiteturais (ADR)

### ADR-001 — FastAPI em vez de Flask

**Contexto:** Escolha do framework HTTP para o servidor no notebook Colab.

**Decisão:** Usar FastAPI.

**Justificativa:**
- Suporte nativo a async/await, permitindo que health checks e inferência coexistam sem bloqueio
- Validação automática de request/response via Pydantic (reduz bugs de parsing)
- Geração automática de documentação OpenAPI (`/docs`) útil durante desenvolvimento
- Performance superior ao Flask em benchmarks de I/O (relevante para streaming SSE)
- Compatibilidade com streaming SSE via `StreamingResponse` sem biblioteca adicional

**Alternativas Rejeitadas:** Flask (síncrono, verboso para validação), aiohttp (sem validação integrada).

---

### ADR-002 — Quantização 4-bit (NF4 via bitsandbytes)

**Contexto:** A GPU T4 do Colab free tem 16 GB de VRAM. Modelos como Mistral-7B em FP16 ocupam ~14 GB.

**Decisão:** Carregar modelos com quantização NF4 (4-bit normal float) via `BitsAndBytesConfig`.

**Justificativa:**
- Mistral-7B em NF4: ~5 GB de VRAM — margem de 11 GB para KV-cache e ativações
- Degradação de qualidade mínima (< 2% em benchmarks padrão vs. FP16)
- Permite carregar modelos maiores (até ~13B parâmetros na T4)
- Sem necessidade de compilação customizada — bitsandbytes instala via pip

**Alternativas Rejeitadas:** FP16 (risco de OOM), GPTQ (requer modelos pré-quantizados), GGUF/llama.cpp (menos integrado ao ecossistema HuggingFace).

---

### ADR-003 — ngrok em vez de Cloudflare Tunnel ou bore.pub

**Contexto:** Necessidade de expor a porta 8000 do Colab para a internet.

**Decisão:** Usar ngrok (plano free) via `pyngrok`.

**Justificativa:**
- Integração Python trivial (`ngrok.connect(8000)` retorna a URL pública)
- SDK Python maduro e bem mantido (`pyngrok`)
- Sem necessidade de conta paga para o caso de uso (1 túnel simultâneo é suficiente)
- Latência adicional mínima (~20–50 ms para regiões US/EU)
- Amplamente usado e documentado

**Alternativas Rejeitadas:** Cloudflare Tunnel (requer instalação de binário `cloudflared`), bore.pub (menos estável, sem SDK Python), serveo.net (instável), localtunnel (timeouts frequentes).

---

### ADR-004 — Google Drive como Conta Armazém em vez de Banco de Dados Externo

**Contexto:** Necessidade de persistência de estado entre sessões Colab (que são efêmeras).

**Decisão:** Usar Google Drive de uma conta dedicada como armazenamento de estado.

**Justificativa:**
- Nativo ao ecossistema Google Colab (montagem via `google.colab.drive.mount`)
- Zero custo adicional (15 GB gratuitos são mais que suficientes para o estado do sistema)
- Acesso tanto do notebook (via Drive mount) quanto do orquestrador local (via Drive API)
- Sem necessidade de configurar banco de dados externo (Redis, PostgreSQL, etc.)
- Arquivos JSON são simples de debugar manualmente

**Alternativas Rejeitadas:** Redis (custo, complexidade), Firebase (custo acima do free tier), banco de dados local (não acessível pelo Colab), GitHub (latência, não adequado para estado de runtime).

---

### ADR-005 — Compatibilidade com OpenAI Chat Completions API

**Contexto:** Múltiplos agentes (Ouroboros Runtime, Hermes Agent, OpenClaw) precisam consumir o servidor.

**Decisão:** Implementar a API seguindo o contrato da OpenAI Chat Completions API.

**Justificativa:**
- Zero modificação necessária nos agentes consumidores — basta trocar a `base_url`
- Ecossistema rico de clientes (SDKs Python, TypeScript, Go, Rust)
- Qualquer agente futuro compatível com OpenAI funcionará automaticamente
- Facilita testes com ferramentas existentes (LangChain, LlamaIndex, OpenAI SDK)
- Reduz acoplamento entre Colab Infinity e agentes específicos

**Trade-off:** Subset do protocolo — campos como `function_calling`, `vision` e `logit_bias` não são suportados nesta versão.

---

## 9. Considerações de Segurança

### 9.1 Superfície de Ataque

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SUPERFÍCIE DE ATAQUE — COLAB INFINITY                                       │
│                                                                              │
│  [ALTA EXPOSIÇÃO]    URL ngrok pública — acessível por qualquer IP          │
│                      Mitigação: API Key opcional (header Authorization)      │
│                                                                              │
│  [MÉDIA EXPOSIÇÃO]   Google Drive da conta armazém                          │
│                      Mitigação: OAuth2 com escopo mínimo (Drive.file)       │
│                                                                              │
│  [BAIXA EXPOSIÇÃO]   Proxy local :8081                                      │
│                      Mitigação: bind em 127.0.0.1 apenas                    │
│                                                                              │
│  [ZERO EXPOSIÇÃO]    FastAPI :8000 dentro do Colab                          │
│                      Não acessível diretamente da internet                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Proteção de Credenciais

| Credencial             | Armazenamento Recomendado               | O que NÃO fazer                     |
|------------------------|-----------------------------------------|-------------------------------------|
| Token ngrok            | Variável de ambiente `NGROK_TOKEN`      | Nunca hardcodar no notebook         |
| OAuth2 Drive token     | `~/.hermes_infinito/drive_token.json`   | Nunca commitar em repositório       |
| API Key do servidor    | `hermes_config.yaml` (chmod 600)        | Nunca expor em logs                 |
| Credenciais Google     | Variáveis de ambiente do Colab          | Nunca em células visíveis do notebook |

### 9.3 Isolamento do Orquestrador

O proxy local faz bind **exclusivamente em `127.0.0.1`**, nunca em `0.0.0.0`. Isso garante que
apenas processos na mesma máquina (o Ouroboros Runtime, Hermes Agent e OpenClaw rodando localmente)
possam acessar o proxy. A URL ngrok pública, por outro lado, pode ter autenticação opcional habilitada.

---

## 10. Considerações de Escalabilidade

### 10.1 Limites Atuais (Versão 1.0)

| Dimensão                    | Limite Atual             | Impacto                                         |
|-----------------------------|--------------------------|--------------------------------------------------|
| Contas no pool              | Até ~10                 | Acima disso, cooldown de 24h entre reusos cria gaps |
| Sessões simultâneas         | 1 (por design)           | Não há paralelismo; fila de requisições no proxy |
| Modelos simultâneos         | 1 por sessão             | Trocar de modelo requer reiniciar sessão         |
| Throughput                  | ~1–2 req/s (T4 + 7B)    | Limitado pela GPU do Colab free                  |

### 10.2 Caminho de Migração para Escala

```
[Colab Free — desenvolvimento e prototipagem]
        │
        ▼  quando uptime > 95% requerido ou >10 req/min
[Colab Pro/Pro+ — sessões mais longas, A100]
        │
        ▼  quando custo justifica ou dados são sensíveis
[Cloud Run / Modal / Replicate — deploy gerenciado]
        │
        ▼  quando infraestrutura própria é necessária
[GKE / EKS com vLLM — produção empresarial]
```

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*