# Colab Infinity — Guia de Integração com o Ouroboros Runtime

**Versão:** 1.0.0  
**Data:** 2025-07-14  
**Status:** Aprovado  
**Referências:** `03_sad.md`, `04_api_spec.md`, `05_setup_guide.md`

---

## Índice

1. [Visão Geral da Integração](#1-visão-geral-da-integração)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Configuração do Ambiente do Ouroboros Runtime](#3-configuração-do-ambiente-do-ouroboros-runtime)
4. [Integração via openai SDK (TypeScript)](#4-integração-via-openai-sdk-typescript)
5. [Integração com o Protocolo Anti-Vibe](#5-integração-com-o-protocolo-anti-vibe)
6. [Configuração por Agente do Conselho](#6-configuração-por-agente-do-conselho)
7. [Tratamento de Erros e Retry no Daemon](#7-tratamento-de-erros-e-retry-no-daemon)
8. [Monitoramento da Integração](#8-monitoramento-da-integração)
9. [Exemplos de Waves com Colab Infinity](#9-exemplos-de-waves-com-colab-infinity)
10. [Migração de Provider de LLM](#10-migração-de-provider-de-llm)
11. [Troubleshooting da Integração](#11-troubleshooting-da-integração)

---

## 1. Visão Geral da Integração

O **Colab Infinity** atua como um **provider de LLM transparente** para o Ouroboros Runtime.
Do ponto de vista do Daemon do Ouroboros, o Colab Infinity é indistinguível de qualquer outro
endpoint compatível com a API OpenAI — a única diferença é a `base_url` apontando para o
proxy local em vez dos servidores da OpenAI.

### 1.1 Posicionamento na Arquitetura do Ouroboros

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         OUROBOROS RUNTIME                                     │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  DAEMON (Bun/TypeScript)                                                 │ │
│  │                                                                           │ │
│  │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │ │
│  │  │  Wave Orch.      │   │  Agent Council   │   │  Memory (SQLite)    │   │ │
│  │  │                  │   │                  │   │  WAL mode           │   │ │
│  │  │  Dispara e        │   │  Ouroboros Core  │   │                     │   │ │
│  │  │  coordena Waves   │   │  Vision          │   │  Persiste contexto  │   │ │
│  │  │                  │   │  Architect       │   │  e histórico        │   │ │
│  │  │                  │   │  Guardian        │   │                     │   │ │
│  │  │                  │   │  Kinetic         │   │                     │   │ │
│  │  └────────┬─────────┘   └────────┬─────────┘   └─────────────────────┘  │ │
│  │           │                      │                                        │ │
│  │           └──────────────────────┘                                        │ │
│  │                        │                                                  │ │
│  │                        │ LLM API calls (OpenAI-compatible)                │ │
│  │                        ▼                                                  │ │
│  │           ┌────────────────────────────┐                                  │ │
│  │           │  LLM Client Module         │                                  │ │
│  │           │  base_url: configurável    │                                  │ │
│  │           └────────────┬───────────────┘                                  │ │
│  └────────────────────────┼────────────────────────────────────────────────┘ │
│                           │                                                   │
└───────────────────────────┼───────────────────────────────────────────────────┘
                            │
           ┌────────────────▼───────────────────┐
           │  COLAB INFINITY                      │
           │  Proxy Local http://127.0.0.1:11434  │
           │  → ngrok → Google Colab GPU → LLM    │
           └────────────────────────────────────┘
```

### 1.2 O Que a Integração Fornece

| Capacidade                         | Benefício para o Ouroboros Runtime                              |
|------------------------------------|-----------------------------------------------------------------|
| Inferência LLM gratuita            | Elimina custo de API OpenAI/Anthropic durante desenvolvimento   |
| API compatível com OpenAI          | Zero modificação no código do Daemon ou dos agentes             |
| Disponibilidade contínua           | Pool de contas + rotação automática → uptime ≥ 95%             |
| Tolerância a falhas                | Troca automática de sessão Colab transparente para o Daemon     |
| Streaming de tokens                | Reduz latência percebida em Waves longas                        |
| Métricas de quota                  | Campo `x_colab_infinity.quota_remaining_minutes` na resposta    |

### 1.3 O Que a Integração NÃO Altera

- A lógica interna do Protocolo Anti-Vibe
- O esquema SQLite de memória do Ouroboros
- O sistema de Waves e coordenação de agentes
- A TUI React Ink
- O sandbox Python isolado (`.ouroboros/venv`)

---

## 2. Pré-requisitos

### 2.1 Colab Infinity

- Colab Infinity instalado e operacional (ver `05_setup_guide.md`)
- Proxy local respondendo em `http://127.0.0.1:11434`
- Verificação rápida:

```bash
curl -s http://127.0.0.1:11434/health \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('OK' if d['status']=='ok' else 'ERRO:', d.get('model',{}).get('id','?'))"
# Saída esperada: OK: mistralai/Mistral-7B-Instruct-v0.2
```

### 2.2 Ouroboros Runtime

- Repositório clonado: `https://github.com/RenyEnnos/ouroboros-runtime`
- Dependências instaladas: `bun install`
- Bun ≥ 1.1.0: `bun --version`

### 2.3 Dependências TypeScript/Bun

O Ouroboros Runtime usa o pacote `openai` do npm para comunicação com LLMs:

```bash
# No diretório do Ouroboros Runtime
bun add openai@^4.47.0
```

---

## 3. Configuração do Ambiente do Ouroboros Runtime

### 3.1 Arquivo `.env` — Configuração Mínima

Na raiz do repositório do Ouroboros Runtime, crie ou edite o arquivo `.env`:

```dotenv
# ─────────────────────────────────────────────────────────────────────
# COLAB INFINITY — Provider de LLM para o Ouroboros Runtime
# ─────────────────────────────────────────────────────────────────────

# URL base do proxy local do Colab Infinity
# Nunca altere diretamente para a URL ngrok — o proxy abstrai as trocas de conta
LLM_BASE_URL=http://127.0.0.1:11434/v1

# API Key (deixar como "dummy" se require_auth: false no Colab Infinity)
LLM_API_KEY=dummy

# Modelo carregado na sessão Colab ativa
# Deve corresponder ao MODEL_ID configurado no notebook
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2

# Provider type — identifica o tipo de cliente a ser instanciado
LLM_PROVIDER=openai_compatible

# Timeout por requisição em milissegundos
# Inferência em T4 com 4bit pode levar até 120s para prompts longos
LLM_TIMEOUT_MS=120000

# Número máximo de retentativas em caso de erro transitório (503, 429)
LLM_MAX_RETRIES=3

# Habilitar streaming de tokens (recomendado para UX responsiva)
LLM_STREAM=true

# ─────────────────────────────────────────────────────────────────────
# CONFIGURAÇÕES OPCIONAIS POR AGENTE DO CONSELHO
# ─────────────────────────────────────────────────────────────────────

# Sobrescreve as configurações globais para agentes específicos
# Útil para ajustar temperatura por tipo de tarefa

# Agente Architect: mais determinístico para especificações técnicas
ARCHITECT_AGENT_TEMPERATURE=0.4
ARCHITECT_AGENT_MAX_TOKENS=2048

# Agente Vision: equilíbrio entre criatividade e precisão
VISION_AGENT_TEMPERATURE=0.6
VISION_AGENT_MAX_TOKENS=1024

# Agente Guardian: altamente determinístico para validação
GUARDIAN_AGENT_TEMPERATURE=0.2
GUARDIAN_AGENT_MAX_TOKENS=512

# Agente Kinetic: mais criativo para geração de código
KINETIC_AGENT_TEMPERATURE=0.7
KINETIC_AGENT_MAX_TOKENS=4096

# Agente Ouroboros Core: balanceado para decisões de alto nível
OUROBOROS_CORE_TEMPERATURE=0.5
OUROBOROS_CORE_MAX_TOKENS=1024
```

### 3.2 Arquivo `.env` — Configuração com Autenticação

Quando o Colab Infinity está configurado com `require_auth: true`:

```dotenv
# API Key deve corresponder ao valor em colab_infinity_config.yaml → server.api_key
LLM_API_KEY=ci-sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 3.3 Verificar que o `.env` foi carregado

```bash
# No diretório do Ouroboros Runtime
bun -e "console.log('LLM_BASE_URL:', process.env.LLM_BASE_URL)"
# Saída esperada: LLM_BASE_URL: http://127.0.0.1:11434/v1
```

---

## 4. Integração via openai SDK (TypeScript)

### 4.1 Módulo de Cliente LLM (Padrão de Implementação)

O código abaixo ilustra o **padrão de integração recomendado** para o módulo de cliente LLM
do Ouroboros Runtime. Trata-se de código de referência — a implementação real pode diferir.

```typescript
// Arquivo de referência: src/llm/client.ts
// Padrão de instanciação do cliente OpenAI compatível com Colab Infinity

import OpenAI from "openai";

// ─── Tipos ───────────────────────────────────────────────────────────────────

export interface LLMClientConfig {
  baseURL: string;
  apiKey: string;
  defaultModel: string;
  defaultTimeoutMs: number;
  maxRetries: number;
  stream: boolean;
}

export interface AgentCallOptions {
  agentName: string;
  messages: Array<{ role: "system" | "user" | "assistant"; content: string }>;
  temperature?: number;
  maxTokens?: number;
  stream?: boolean;
}

export interface LLMResponse {
  content: string;
  usage: {
    promptTokens: number;
    completionTokens: number;
    totalTokens: number;
  };
  sessionId: string;
  accountIndex: number;
  inferenceMs: number;
  quotaRemainingMinutes: number;
}

// ─── Configuração padrão (lida do ambiente) ───────────────────────────────────

function buildDefaultConfig(): LLMClientConfig {
  return {
    baseURL:        process.env.LLM_BASE_URL      ?? "http://127.0.0.1:11434/v1",
    apiKey:         process.env.LLM_API_KEY        ?? "dummy",
    defaultModel:   process.env.LLM_MODEL          ?? "mistralai/Mistral-7B-Instruct-v0.2",
    defaultTimeoutMs: parseInt(process.env.LLM_TIMEOUT_MS ?? "120000"),
    maxRetries:     parseInt(process.env.LLM_MAX_RETRIES   ?? "3"),
    stream:         (process.env.LLM_STREAM        ?? "true") === "true",
  };
}

// ─── Classe de cliente ────────────────────────────────────────────────────────

export class ColabInfinityLLMClient {
  private client: OpenAI;
  private config: LLMClientConfig;

  constructor(config?: Partial<LLMClientConfig>) {
    this.config = { ...buildDefaultConfig(), ...config };
    this.client = new OpenAI({
      baseURL:    this.config.baseURL,
      apiKey:     this.config.apiKey,
      timeout:    this.config.defaultTimeoutMs,
      maxRetries: 0, // Gerenciamos retry manualmente para melhor controle
    });
  }

  // ─── Verificação de saúde ──────────────────────────────────────────────────

  async healthCheck(): Promise<{ ok: boolean; model: string; quotaRemainingMin: number }> {
    try {
      const resp = await fetch(`${this.config.baseURL.replace("/v1", "")}/health`, {
        signal: AbortSignal.timeout(5_000),
      });
      if (!resp.ok) return { ok: false, model: "", quotaRemainingMin: 0 };
      const body = await resp.json() as Record<string, unknown>;
      const session = body.session as Record<string, unknown>;
      const model = body.model as Record<string, unknown>;
      return {
        ok: body.status === "ok",
        model: (model?.id as string) ?? "",
        quotaRemainingMin: (session?.estimated_quota_remaining_minutes as number) ?? 0,
      };
    } catch {
      return { ok: false, model: "", quotaRemainingMin: 0 };
    }
  }

  // ─── Aguardar disponibilidade ─────────────────────────────────────────────

  async waitForReady(maxWaitMs = 300_000): Promise<void> {
    const start = Date.now();
    const POLL_INTERVAL = 15_000;

    while (Date.now() - start < maxWaitMs) {
      const health = await this.healthCheck();
      if (health.ok) {
        console.log(`[ColabInfinity] Servidor pronto. Modelo: ${health.model}`);
        return;
      }
      console.log(`[ColabInfinity] Aguardando servidor... (${Math.round((Date.now() - start) / 1000)}s)`);
      await new Promise(r => setTimeout(r, POLL_INTERVAL));
    }

    throw new Error(`[ColabInfinity] Servidor não ficou pronto em ${maxWaitMs / 1000}s`);
  }

  // ─── Chamada principal de inferência ──────────────────────────────────────

  async call(options: AgentCallOptions): Promise<LLMResponse> {
    const {
      agentName,
      messages,
      temperature = 0.7,
      maxTokens = 1024,
      stream = this.config.stream,
    } = options;

    let lastError: Error | null = null;

    for (let attempt = 1; attempt <= this.config.maxRetries; attempt++) {
      try {
        const startTime = Date.now();

        if (stream) {
          return await this._callStreaming(agentName, messages, temperature, maxTokens, startTime);
        } else {
          return await this._callSync(agentName, messages, temperature, maxTokens, startTime);
        }

      } catch (err) {
        lastError = err as Error;
        const isRetryable = this._isRetryableError(err);

        console.warn(
          `[ColabInfinity] Tentativa ${attempt}/${this.config.maxRetries} falhou ` +
          `(${agentName}): ${(err as Error).message}`
        );

        if (!isRetryable || attempt === this.config.maxRetries) break;

        // Backoff específico por tipo de erro
        const backoffMs = this._getBackoffMs(err, attempt);
        console.log(`[ColabInfinity] Aguardando ${backoffMs / 1000}s antes de retentar...`);
        await new Promise(r => setTimeout(r, backoffMs));
      }
    }

    throw new Error(
      `[ColabInfinity] Todas as tentativas falharam para agente "${agentName}": ${lastError?.message}`
    );
  }

  // ─── Implementação síncrona ────────────────────────────────────────────────

  private async _callSync(
    agentName: string,
    messages: AgentCallOptions["messages"],
    temperature: number,
    maxTokens: number,
    startTime: number
  ): Promise<LLMResponse> {
    const response = await this.client.chat.completions.create({
      model:        this.config.defaultModel,
      messages,
      temperature,
      max_tokens:   maxTokens,
      stream:       false,
      user:         `ouroboros-${agentName.toLowerCase().replace(/\s+/g, "-")}`,
    });

    const content = response.choices[0]?.message?.content ?? "";
    if (!content) throw new Error("Resposta vazia do modelo LLM.");

    // Extrair metadados do Colab Infinity (campos proprietários)
    const xci = (response as Record<string, unknown>).x_colab_infinity as Record<string, unknown> | undefined;

    return {
      content,
      usage: {
        promptTokens:     response.usage?.prompt_tokens     ?? 0,
        completionTokens: response.usage?.completion_tokens ?? 0,
        totalTokens:      response.usage?.total_tokens      ?? 0,
      },
      sessionId:            (xci?.session_id             as string)  ?? "",
      accountIndex:         (xci?.account_index          as number)  ?? -1,
      inferenceMs:          (xci?.inference_ms           as number)  ?? (Date.now() - startTime),
      quotaRemainingMinutes:(xci?.quota_remaining_minutes as number) ?? -1,
    };
  }

  // ─── Implementação com streaming ──────────────────────────────────────────

  private async _callStreaming(
    agentName: string,
    messages: AgentCallOptions["messages"],
    temperature: number,
    maxTokens: number,
    startTime: number
  ): Promise<LLMResponse> {
    const stream = await this.client.chat.completions.create({
      model:       this.config.defaultModel,
      messages,
      temperature,
      max_tokens:  maxTokens,
      stream:      true,
      user:        `ouroboros-${agentName.toLowerCase().replace(/\s+/g, "-")}`,
    });

    let content = "";
    let promptTokens = 0;
    let completionTokens = 0;

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta?.content;
      if (delta) content += delta;

      // Último chunk pode conter usage (quando stream_options disponível)
      if (chunk.usage) {
        promptTokens     = chunk.usage.prompt_tokens     ?? 0;
        completionTokens = chunk.usage.completion_tokens ?? 0;
      }
    }

    if (!content) throw new Error("Stream vazio — nenhum token gerado.");

    return {
      content,
      usage: { promptTokens, completionTokens, totalTokens: promptTokens + completionTokens },
      sessionId:             "",  // não disponível no stream com esta versão
      accountIndex:          -1,
      inferenceMs:           Date.now() - startTime,
      quotaRemainingMinutes: -1,
    };
  }

  // ─── Lógica de retry ──────────────────────────────────────────────────────

  private _isRetryableError(err: unknown): boolean {
    if (!(err instanceof Error)) return false;
    const msg = err.message.toLowerCase();

    // Erros retryáveis: troca de conta em andamento, rate limit, timeout transitório
    if (msg.includes("session_switching"))  return true;
    if (msg.includes("pool_exhausted"))     return true;
    if (msg.includes("model_loading"))      return true;
    if (msg.includes("rate_limit"))         return true;
    if (msg.includes("503"))               return true;
    if (msg.includes("429"))               return true;
    if (msg.includes("econnrefused"))       return true;
    if (msg.includes("fetch failed"))       return true;
    if (msg.includes("timeout"))           return true;

    // Erros não-retryáveis: erro de configuração, autenticação, payload inválido
    if (msg.includes("401"))   return false;
    if (msg.includes("400"))   return false;
    if (msg.includes("422"))   return false;

    return false;
  }

  private _getBackoffMs(err: unknown, attempt: number): number {
    const msg = ((err as Error)?.message ?? "").toLowerCase();

    // SESSION_SWITCHING: aguardar tempo fixo para troca completar
    if (msg.includes("session_switching") || msg.includes("503")) {
      return 90_000; // 90 segundos — tempo médio de troca de conta
    }

    // RATE_LIMIT: backoff exponencial curto
    if (msg.includes("rate_limit") || msg.includes("429")) {
      return Math.min(1_000 * Math.pow(2, attempt), 30_000);
    }

    // MODEL_LOADING: aguardar modelo carregar
    if (msg.includes("model_loading")) {
      return 20_000;
    }

    // Default: backoff exponencial
    return Math.min(5_000 * Math.pow(2, attempt - 1), 60_000);
  }
}

// ─── Singleton para uso global ────────────────────────────────────────────────

let _defaultClient: ColabInfinityLLMClient | null = null;

export function getLLMClient(): ColabInfinityLLMClient {
  if (!_defaultClient) {
    _defaultClient = new ColabInfinityLLMClient();
  }
  return _defaultClient;
}
```

### 4.2 Uso no Handler de Wave

```typescript
// Exemplo ilustrativo: src/daemon/wave-handler.ts
// Padrão de invocação do LLM dentro de um handler de Wave

import { getLLMClient, LLMResponse } from "../llm/client";

interface WaveContext {
  waveId: string;
  task: string;
  memoryContext: string[];  // Itens relevantes da memória SQLite
}

async function executeArchitectWave(ctx: WaveContext): Promise<string> {
  const llm = getLLMClient();

  // Verificar disponibilidade antes de iniciar a Wave
  const health = await llm.healthCheck();
  if (!health.ok) {
    throw new Error(
      `[Wave:${ctx.waveId}] Colab Infinity não disponível. ` +
      `Verifique se o proxy local está ativo em http://127.0.0.1:11434`
    );
  }

  // Alertar se a quota estiver baixa
  if (health.quotaRemainingMin > 0 && health.quotaRemainingMin < 30) {
    console.warn(
      `[Wave:${ctx.waveId}] ⚠️ Quota restante: ${health.quotaRemainingMin} minutos. ` +
      `Troca de conta pode ocorrer em breve.`
    );
  }

  // Construir as mensagens para o agente Architect
  const messages: Array<{ role: "system" | "user" | "assistant"; content: string }> = [
    {
      role: "system",
      content:
        "Você é o agente Architect do Ouroboros Runtime. " +
        "Sua especialidade é design de sistemas, arquitetura de software " +
        "e criação de especificações técnicas detalhadas e verificáveis. " +
        "Seja preciso, estruturado e sempre inclua critérios de aceite mensuráveis.",
    },
    // Injetar contexto relevante da memória SQLite
    ...ctx.memoryContext.slice(-5).map(m => ({
      role: "user" as const,
      content: `[Contexto de memória]: ${m}`,
    })),
    {
      role: "user",
      content: ctx.task,
    },
  ];

  // Invocar o LLM
  let result: LLMResponse;
  try {
    result = await llm.call({
      agentName:   "Architect",
      messages,
      temperature: parseFloat(process.env.ARCHITECT_AGENT_TEMPERATURE ?? "0.4"),
      maxTokens:   parseInt(process.env.ARCHITECT_AGENT_MAX_TOKENS    ?? "2048"),
    });
  } catch (err) {
    // Erro irrecuperável após todos os retries
    throw new Error(
      `[Wave:${ctx.waveId}:Architect] LLM falhou: ${(err as Error).message}`
    );
  }

  // Registrar métricas relevantes
  console.log(
    `[Wave:${ctx.waveId}:Architect] ` +
    `tokens=${result.usage.totalTokens} ` +
    `ms=${result.inferenceMs} ` +
    `quota=${result.quotaRemainingMinutes}min ` +
    `session=${result.sessionId.slice(-8)}`
  );

  return result.content;
}
```

---

## 5. Integração com o Protocolo Anti-Vibe

O Protocolo Anti-Vibe define gates de qualidade rigorosos que as respostas do LLM devem
passar antes de serem promovidas. O Colab Infinity opera **antes do Gate 1** e é completamente
transparente ao protocolo.

### 5.1 Posicionamento nos Gates

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                      PROTOCOLO ANTI-VIBE — FLUXO COMPLETO                     │
│                                                                               │
│  Input da Wave                                                                │
│       │                                                                       │
│       ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐             │
│  │  COLAB INFINITY (transparente ao protocolo)                  │             │
│  │  Recebe prompt → executa inferência → retorna texto          │             │
│  └─────────────────────────────────────────────────────────────┘             │
│       │                                                                       │
│       ▼  Resposta bruta do LLM                                                │
│  ┌─────────────────────────────────────────────────────────────┐             │
│  │  GATE 1: ESPECIFICAÇÃO                                       │             │
│  │  ✓ Resposta é estruturada conforme o schema esperado?        │             │
│  │  ✓ Todos os campos obrigatórios presentes?                   │             │
│  │  ✓ Critérios de aceite definidos e mensuráveis?              │             │
│  └────────────────────────┬────────────────────────────────────┘             │
│                           │ sim                                               │
│                           ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────┐             │
│  │  GATE 2: VALIDAÇÃO AUTOMÁTICA                                │             │
│  │  ✓ Código gerado é sintaticamente válido?                    │             │
│  │  ✓ Testes unitários passam no sandbox Python?                │             │
│  │  ✓ Sem dependências proibidas ou imports suspeitos?          │             │
│  └────────────────────────┬────────────────────────────────────┘             │
│                           │ sim                                               │
│                           ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────┐             │
│  │  GATE 3: REVISÃO HUMANA (TUI React Ink)                      │             │
│  │  ✓ Operador revisa e aprova via interface                    │             │
│  │  ✓ Anotações e feedback registrados no SQLite               │             │
│  └────────────────────────┬────────────────────────────────────┘             │
│                           │ aprovado                                          │
│                           ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────┐             │
│  │  GATE 4: TESTES DE SISTEMA                                   │             │
│  │  ✓ Suite de testes completa passa no sandbox isolado?        │             │
│  │  ✓ Sem regressões em funcionalidades existentes?             │             │
│  └────────────────────────┬────────────────────────────────────┘             │
│                           │ sim                                               │
│                           ▼                                                   │
│  Artefato promovido para produção                                             │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Considerações Específicas por Gate

**Gate 1 — Especificação:**
- O Colab Infinity retorna texto livre; o Daemon é responsável por validar a estrutura
- Recomendado: incluir no `system_prompt` do agente instruções explícitas sobre o schema esperado
- Exemplo de instrução de schema no system prompt:

```typescript
// Exemplo ilustrativo: instruir o LLM a retornar um formato específico
const systemPromptWithSchema = `
Você é o agente Architect do Ouroboros Runtime.
SEMPRE responda no seguinte formato JSON:
{
  "title": "Título conciso da especificação",
  "description": "Descrição técnica detalhada",
  "acceptance_criteria": ["Critério 1 mensurável", "Critério 2 verificável"],
  "implementation_notes": "Notas de implementação relevantes",
  "risks": ["Risco 1", "Risco 2"]
}
Não inclua texto fora do objeto JSON.
`;
```

**Gate 2 — Validação:**
- O sandbox Python (`.ouroboros/venv`) é completamente independente do Colab Infinity
- A validação roda localmente; o LLM não tem acesso ao sandbox
- Se o código gerado falhar na validação, o Daemon pode solicitar correção ao LLM

```typescript
// Exemplo ilustrativo: loop de correção com o LLM
async function validateAndRetry(
  llm: ColabInfinityLLMClient,
  code: string,
  validationError: string,
  originalMessages: Array<{ role: "system" | "user" | "assistant"; content: string }>,
  maxAttempts = 2
): Promise<string> {
  for (let i = 0; i < maxAttempts; i++) {
    const correctionMessages = [
      ...originalMessages,
      { role: "assistant" as const, content: code },
      {
        role: "user" as const,
        content:
          `O código acima falhou na validação com o seguinte erro:\n\n` +
          `\`\`\`\n${validationError}\n\`\`\`\n\n` +
          `Por favor, corrija o problema e retorne APENAS o código corrigido.`,
      },
    ];
    const result = await llm.call({
      agentName: "Kinetic-Correction",
      messages: correctionMessages,
      temperature: 0.2, // mais determinístico para correção
      maxTokens: 2048,
    });
    code = result.content;
    // Executar validação novamente (lógica do sandbox Python)
    // Se passar, retornar; se não, continuar o loop
  }
  return code;
}
```

**Gate 3 — Revisão Humana:**
- O operador vê na TUI a resposta do LLM e decide aprovar, rejeitar ou modificar
- O `sessionId` e `accountIndex` do Colab Infinity podem ser exibidos na TUI para diagnóstico
- Exemplo de exibição na TUI (React Ink — conceitual):

```typescript
// Exemplo conceitual de como exibir metadados do Colab Infinity na TUI
// Este não é código de produção — apenas ilustra o padrão de integração

interface ReviewPanelProps {
  llmResponse: LLMResponse;
  agentName: string;
}

// Em um componente React Ink hipotético:
// const ReviewPanel = ({ llmResponse, agentName }: ReviewPanelProps) => (
//   <Box flexDirection="column">
//     <Text color="cyan">[{agentName}] LLM Response</Text>
//     <Text>{llmResponse.content}</Text>
//     <Text color="gray" dimColor>
//       Session: {llmResponse.sessionId.slice(-8)} |
//       Account: {llmResponse.accountIndex} |
//       Tokens: {llmResponse.usage.totalTokens} |
//       {llmResponse.quotaRemainingMinutes > 0 &&
//         ` Quota: ${llmResponse.quotaRemainingMinutes}min`}
//     </Text>
//   </Box>
// );
```

---

## 6. Configuração por Agente do Conselho

Cada agente do Conselho do Ouroboros tem um perfil de uso do LLM diferente. A tabela abaixo
descreve as configurações recomendadas para otimizar custo de quota e qualidade de resposta.

### 6.1 Tabela de Configurações Recomendadas

| Agente           | Temperatura | max_tokens | Características da Tarefa                     | Notas                                |
|------------------|-------------|------------|-----------------------------------------------|--------------------------------------|
| Ouroboros Core   | 0.5         | 1024       | Decisões de alto nível, coordenação           | Balanceado; contexto médio           |
| Vision           | 0.6         | 1024       | Análise de requisitos, visão do produto       | Ligeiramente criativo                |
| Architect        | 0.4         | 2048       | Especificações técnicas, design de sistemas   | Mais determinístico; respostas longas|
| Guardian         | 0.2         | 512        | Validação, revisão de segurança, qualidade    | Alta precisão; respostas curtas      |
| Kinetic          | 0.7         | 4096       | Geração de código, implementação              | Criativo; respostas muito longas     |

### 6.2 System Prompts Recomendados por Agente

```typescript
// Exemplo ilustrativo: system prompts otimizados para o Colab Infinity
// Os modelos open-source respondem melhor a instruções explícitas e estruturadas

const AGENT_SYSTEM_PROMPTS = {
  "Architect": `
Você é o agente Architect do Ouroboros Runtime, um sistema de orquestração multiagente.
Sua especialidade é design de sistemas e especificações técnicas.

INSTRUÇÕES OBRIGATÓRIAS:
1. Sempre produza especificações completas com critérios de aceite verificáveis
2. Use linguagem técnica precisa; evite ambiguidades
3. Inclua diagramas ASCII quando relevante para a arquitetura
4. Estruture a resposta com seções claramente delimitadas
5. Considere edge cases e casos de falha

CONTEXTO DO SISTEMA:
- Você opera dentro do Ouroboros Runtime (Bun/TypeScript + SQLite)
- Código gerado será executado no sandbox Python isolado (.ouroboros/venv)
- Todas as respostas passarão por revisão humana (Protocolo Anti-Vibe)
`,

  "Guardian": `
Você é o agente Guardian do Ouroboros Runtime.
Sua função é validação, revisão de segurança e garantia de qualidade.

INSTRUÇÕES OBRIGATÓRIAS:
1. Seja conservador e preciso — erros de segurança têm alto impacto
2. Identifique vulnerabilidades, edge cases e pontos de falha
3. Responda com uma lista estruturada de aprovações e rejeições
4. Inclua sempre a justificativa para cada decisão
5. Em caso de dúvida, rejeite e solicite esclarecimento

FORMATO DE RESPOSTA:
{
  "decision": "APPROVE" | "REJECT" | "REQUEST_CLARIFICATION",
  "findings": ["item 1", "item 2"],
  "critical_issues": [],
  "recommendations": []
}
`,

  "Kinetic": `
Você é o agente Kinetic do Ouroboros Runtime.
Sua especialidade é implementação: escrever código funcional, testável e bem documentado.

INSTRUÇÕES OBRIGATÓRIAS:
1. Escreva código Python a menos que especificado de outra forma
2. Sempre inclua docstrings e type hints
3. Escreva código modular e testável (sem dependências ocultas)
4. Prefira código simples e legível a soluções complexas e "inteligentes"
5. Inclua exemplos de uso e testes unitários básicos

AMBIENTE DE EXECUÇÃO:
- Python 3.10+ no sandbox .ouroboros/venv
- Sem acesso à internet dentro do sandbox
- Dependências devem ser instaláveis via pip
`,
};
```

---

## 7. Tratamento de Erros e Retry no Daemon

### 7.1 Mapeamento de Erros do Colab Infinity para o Daemon

| Código HTTP | Código Interno          | Comportamento no Daemon                                               |
|-------------|-------------------------|-----------------------------------------------------------------------|
| `503`       | `SESSION_SWITCHING`     | Retry após 90s (troca de conta em andamento)                          |
| `503`       | `MODEL_LOADING`         | Retry após 20s (modelo ainda inicializando)                           |
| `503`       | `POOL_EXHAUSTED`        | Falhar imediatamente; notificar operador; aguardar intervenção        |
| `504`       | `INFERENCE_TIMEOUT`     | Retry com `max_tokens` reduzido em 50%                                |
| `429`       | `RATE_LIMIT_EXCEEDED`   | Respeitar header `Retry-After`; backoff exponencial                   |
| `500`       | `MODEL_INFERENCE_ERROR` | Retry 1×; se persistir, escalar como erro crítico                     |
| `401`       | `UNAUTHORIZED`          | Falha imediata; não retentar; verificar `LLM_API_KEY` no `.env`       |
| `400`       | `INVALID_REQUEST`       | Falha imediata; inspecionar payload enviado                           |
| `ECONNREFUSED` | —                    | Proxy local parado; reiniciar orquestrador do Colab Infinity          |

### 7.2 Estratégia de Retry com Jitter

```typescript
// Exemplo ilustrativo: função de backoff com jitter
// Evita thundering herd quando múltiplos agentes retentam simultaneamente

function calculateBackoffMs(
  errorType: string,
  attemptNumber: number,
  baseMs = 5_000
): number {
  let backoffMs: number;

  switch (errorType) {
    case "SESSION_SWITCHING":
      // Troca de conta leva 4–8 minutos; esperar um tempo fixo considerável
      backoffMs = 90_000 + Math.random() * 30_000; // 90s + jitter de até 30s
      break;

    case "RATE_LIMIT_EXCEEDED":
      // Backoff exponencial com cap de 30s
      backoffMs = Math.min(baseMs * Math.pow(2, attemptNumber - 1), 30_000);
      break;

    case "MODEL_LOADING":
      // Modelo carregando; tentar a cada 20s
      backoffMs = 20_000 + Math.random() * 10_000;
      break;

    case "INFERENCE_TIMEOUT":
      // Sem aguardar; retentar imediatamente com payload menor
      backoffMs = 1_000;
      break;

    default:
      // Backoff exponencial padrão
      backoffMs = Math.min(baseMs * Math.pow(2, attemptNumber - 1), 60_000);
  }

  // Adicionar jitter de ±10% para evitar sincronização de retries
  const jitter = backoffMs * 0.1 * (Math.random() * 2 - 1);
  return Math.round(backoffMs + jitter);
}
```

### 7.3 Circuit Breaker (Proteção Adicional)

Para evitar sobrecarga do Colab Infinity durante períodos de falha prolongada, o Daemon
pode implementar um circuit breaker:

```typescript
// Exemplo ilustrativo: padrão de circuit breaker para o cliente LLM

enum CircuitState { CLOSED, OPEN, HALF_OPEN }

class LLMCircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private lastFailureTime = 0;
  private readonly failureThreshold = 5;
  private readonly recoveryTimeMs = 120_000; // 2 minutos

  canCall(): boolean {
    if (this.state === CircuitState.CLOSED) return true;
    if (this.state === CircuitState.OPEN) {
      const elapsed = Date.now() - this.lastFailureTime;
      if (elapsed >= this.recoveryTimeMs) {
        this.state = CircuitState.HALF_OPEN;
        return true;
      }
      return false;
    }
    return true; // HALF_OPEN: permite uma chamada de teste
  }

  onSuccess(): void {
    this.failureCount = 0;
    this.state = CircuitState.CLOSED;
  }

  onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
      console.error(
        `[ColabInfinity] Circuit breaker ABERTO após ${this.failureCount} falhas. ` +
        `Aguardando ${this.recoveryTimeMs / 1000}s para tentar novamente.`
      );
    }
  }
}
```

---

## 8. Monitoramento da Integração

### 8.1 Métricas a Coletar no Daemon

O Daemon deve registrar as seguintes métricas para monitorar a saúde da integração:

```typescript
// Exemplo ilustrativo: interface de métricas de integração LLM

interface LLMIntegrationMetrics {
  // Por sessão de Wave
  waveId: string;
  agentName: string;
  attemptNumber: number;

  // Métricas de desempenho
  requestStartTs: number;       // Unix timestamp ms
  responseReceivedTs: number;   // Unix timestamp ms
  totalLatencyMs: number;       // = responseReceivedTs - requestStartTs
  inferenceLatencyMs: number;   // Reportado pelo x_colab_infinity.inference_ms

  // Métricas de uso
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;

  // Estado do Colab Infinity
  sessionId: string;
  accountIndex: number;
  quotaRemainingMinutes: number;

  // Resultado
  success: boolean;
  errorCode?: string;
  retryCount: number;
}
```

### 8.2 Alertas Recomendados

Configure alertas no Daemon quando:

| Condição                                        | Severidade | Ação Recomendada                              |
|-------------------------------------------------|------------|-----------------------------------------------|
| `quotaRemainingMinutes` < 30                    | WARNING    | Informar operador; troca iminente             |
| `quotaRemainingMinutes` < 10                    | ERROR      | Iniciar troca manual de conta                 |
| 3+ `SESSION_SWITCHING` consecutivos em 10 min   | ERROR      | Verificar status do pool de contas            |
| `POOL_EXHAUSTED` detectado                      | CRITICAL   | Intervenção imediata: adicionar conta ao pool |
| Latência média > 30s por 5 minutos              | WARNING    | Verificar uso de VRAM no Colab                |
| Todas as retentativas falham em uma Wave        | CRITICAL   | Verificar logs do orquestrador                |

### 8.3 Health Check Periódico do Daemon

```typescript
// Exemplo ilustrativo: verificação de saúde periódica no Daemon

async function periodicHealthCheck(llm: ColabInfinityLLMClient): Promise<void> {
  const health = await llm.healthCheck();

  if (!health.ok) {
    console.error("[ColabInfinity] ALERTA: Servidor LLM não disponível!");
    // Notificar operador, pausar Waves que dependem de LLM, etc.
    return;
  }

  // Log de saúde periódico
  console.log(
    `[ColabInfinity] Saúde OK | ` +
    `Modelo: ${health.model} | ` +
    `Quota: ${health.quotaRemainingMin}min restantes`
  );

  // Alerta de quota baixa
  if (health.quotaRemainingMin > 0 && health.quotaRemainingMin < 30) {
    console.warn(
      `[ColabInfinity] ⚠️ Quota baixa: ${health.quotaRemainingMin}min. ` +
      `Troca de conta será iniciada em breve pelo orquestrador.`
    );
  }
}

// Executar a cada 5 minutos
// setInterval(() => periodicHealthCheck(getLLMClient()), 5 * 60 * 1000);
```

---

## 9. Exemplos de Waves com Colab Infinity

### 9.1 Wave Simples: Geração de Especificação (Architect)

```typescript
// Exemplo ilustrativo: Wave de geração de especificação técnica
// Demonstra o fluxo completo: prompt → LLM via Colab Infinity → Anti-Vibe Gate 1

async function specificationWave(featureRequest: string): Promise<string> {
  const llm = getLLMClient();

  // Verificar disponibilidade
  await llm.waitForReady(60_000);

  // Chamar o agente Architect
  const result = await llm.call({
    agentName: "Architect",
    messages: [
      {
        role: "system",
        content: AGENT_SYSTEM_PROMPTS["Architect"],
      },
      {
        role: "user",
        content: `
Crie uma especificação técnica completa para a seguinte funcionalidade:

${featureRequest}

A especificação deve incluir:
1. Visão geral e objetivos
2. Requisitos funcionais (RF01, RF02, ...)
3. Requisitos não funcionais
4. Diagrama de componentes (ASCII)
5. Critérios de aceite verificáveis
6. Riscos identificados
        `.trim(),
      },
    ],
    temperature: 0.4,
    maxTokens: 2048,
  });

  // Gate 1: Validar que a especificação tem os campos obrigatórios
  const content = result.content;
  const hasRF = content.includes("RF0") || content.includes("Requisito Funcional");
  const hasCriteria = content.toLowerCase().includes("critério") ||
                      content.toLowerCase().includes("aceite");

  if (!hasRF || !hasCriteria) {
    throw new Error(
      "Gate 1 falhou: Especificação incompleta. " +
      "Faltam requisitos funcionais ou critérios de aceite."
    );
  }

  console.log(
    `[Wave:Specification] Concluída | ` +
    `tokens=${result.usage.totalTokens} | ` +
    `ms=${result.inferenceMs}`
  );

  return content;
}
```

### 9.2 Wave com Múltiplos Agentes: Architect → Guardian

```typescript
// Exemplo ilustrativo: Wave sequencial com dois agentes do Conselho

async function architectAndGuardianWave(task: string): Promise<{
  specification: string;
  validationResult: string;
}> {
  const llm = getLLMClient();

  // Passo 1: Architect gera a especificação
  console.log("[Wave] Passo 1: Architect gerando especificação...");
  const specResult = await llm.call({
    agentName: "Architect",
    messages: [
      { role: "system",  content: AGENT_SYSTEM_PROMPTS["Architect"] },
      { role: "user",    content: task },
    ],
    temperature: 0.4,
    maxTokens: 2048,
  });

  // Passo 2: Guardian valida a especificação do Architect
  console.log("[Wave] Passo 2: Guardian validando especificação...");
  const validationResult = await llm.call({
    agentName: "Guardian",
    messages: [
      { role: "system",    content: AGENT_SYSTEM_PROMPTS["Guardian"] },
      {
        role: "user",
        content:
          `Valide a seguinte especificação técnica produzida pelo agente Architect:\n\n` +
          `---\n${specResult.content}\n---\n\n` +
          `Identifique falhas, ambiguidades, riscos de segurança e incompletudes.`,
      },
    ],
    temperature: 0.2,
    maxTokens: 512,
  });

  console.log(
    `[Wave:ArchitectGuardian] Concluída | ` +
    `total_tokens=${specResult.usage.totalTokens + validationResult.usage.totalTokens}`
  );

  return {
    specification:    specResult.content,
    validationResult: validationResult.content,
  };
}
```

---

## 10. Migração de Provider de LLM

### 10.1 Migrar do Colab Infinity para OpenAI (quando escalar)

Quando o projeto crescer e justificar o custo de APIs pagas, a migração é trivial:

```dotenv
# ANTES (Colab Infinity — desenvolvimento)
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=dummy
LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2

# DEPOIS (OpenAI — produção)
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LLM_MODEL=gpt-4o
```

**Nenhuma alteração de código é necessária no Daemon ou nos agentes.**

### 10.2 Tabela de Equivalência de Modelos

| Colab Infinity (Gratuito)                  | OpenAI Equivalente (Pago)  | Groq Equivalente (Pago)          |
|--------------------------------------------|----------------------------|----------------------------------|
| `mistralai/Mistral-7B-Instruct-v0.2`       | `gpt-3.5-turbo`            | `mixtral-8x7b-32768`             |
| `meta-llama/Meta-Llama-3-8B-Instruct`      | `gpt-4o-mini`              | `llama3-8b-8192`                 |
| `meta-llama/Meta-Llama-3-70B-Instruct`     | `gpt-4o`                   | `llama3-70b-8192`                |
| `mistralai/Mixtral-8x7B-Instruct-v0.1`     | `gpt-4-turbo`              | `mixtral-8x7b-32768`             |

### 10.3 Providers Alternativos Compatíveis

O mesmo módulo de cliente funciona com qualquer provider OpenAI-compatible:

```dotenv
# Ollama (local, sem GPU do Colab)
LLM_BASE_URL=http://127.0.0.1:11434/v1
LLM_API_KEY=ollama
LLM_MODEL=mistral:7b

# Groq (pago, muito rápido)
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LLM_MODEL=llama3-8b-8192

# Together AI (pago, bom custo-benefício)
LLM_BASE_URL=https://api.together.xyz/v1
LLM_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LLM_MODEL=meta-llama/Llama-3-8b-chat-hf
```

---

## 11. Troubleshooting da Integração

### 11.1 Problemas Comuns e Soluções

| Problema                                              | Diagnóstico                                                    | Solução                                                         |
|-------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------|
| `ECONNREFUSED 127.0.0.1:11434`                        | Proxy local do Colab Infinity não está rodando                  | Iniciar o orquestrador; verificar se o Colab está ativo         |
| `UNAUTHORIZED` (401) em todas as chamadas             | `LLM_API_KEY` incorreto ou `require_auth: true` com key errada | Verificar `.env`; comparar com `server.api_key` no config       |
| Resposta vazia (`content: ""`)                        | Modelo retornou sequência de parada imediatamente               | Verificar `stop` tokens; usar temperatura > 0                   |
| Latência > 60s consistentemente                       | Modelo rodando em CPU (GPU não alocada)                        | Verificar `runtime.device: "cuda"` no `/health`                 |
| `MODEL_LOADING` por mais de 10 minutos                | Download lento do modelo (~14 GB via HuggingFace)              | Aguardar; considerar modelo menor (Phi-3 Mini ~3GB)             |
| Respostas truncadas (finish_reason: "length")         | `max_tokens` insuficiente para a tarefa                        | Aumentar `max_tokens` para o agente específico                  |
| Texto em inglês em vez de português                   | Modelo sem instrução de idioma no system prompt                | Adicionar `"Sempre responda em português brasileiro."` ao prompt |
| Respostas inconsistentes entre sessões                | Troca de modelo entre sessões Colab                            | Fixar `MODEL_ID` no notebook e verificar que não muda           |
| `CheckpointDecodeError` ao reiniciar orquestrador     | Checkpoint do Colab Infinity corrompido                        | Ver Runbook seção 5.4; usar checkpoint anterior                  |

### 11.2 Script de Diagnóstico Completo

```bash
#!/bin/bash
# Execute no diretório do Ouroboros Runtime para verificar a integração com Colab Infinity

echo "=== Diagnóstico de Integração Colab Infinity × Ouroboros Runtime ==="
echo ""

# 1. Verificar variáveis de ambiente
echo "1. Variáveis de ambiente (.env):"
if [ -f .env ]; then
    grep "LLM_" .env | sed 's/LLM_API_KEY=.*/LLM_API_KEY=***REDACTED***/g'
else
    echo "   ERRO: Arquivo .env não encontrado!"
fi
echo ""

# 2. Verificar proxy local
echo "2. Proxy local (http://127.0.0.1:11434):"
HEALTH=$(curl -sf http://127.0.0.1:11434/health 2>&1)
if [ $? -eq 0 ]; then
    STATUS=$(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status'])" 2>/dev/null)
    MODEL=$(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('model',{}).get('id','?'))" 2>/dev/null)
    echo "   Status: $STATUS | Modelo: $MODEL"
else
    echo "   ERRO: Proxy não responde. Inicie o orquestrador do Colab Infinity."
fi
echo ""

# 3. Teste de inferência
echo "3. Teste de inferência:"
RESULT=$(curl -sf -X POST http://127.0.0.1:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"messages":[{"role":"user","content":"Responda apenas: OUROBOROS_OK"}],"max_tokens":10,"temperature":0.1}' \
    2>&1)
if [ $? -eq 0 ]; then
    CONTENT=$(echo "$RESULT" | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['choices'][0]['message']['content'])" 2>/dev/null)
    echo "   Resposta do modelo: $CONTENT"
else
    echo "   ERRO: Inferência falhou. Detalhes: $RESULT"
fi
echo ""

# 4. Verificar estado do pool
echo "4. Estado do pool de contas (via CLI do Colab Infinity):"
python3 -m colab_infinity.cli pool list 2>/dev/null || echo "   CLI do Colab Infinity não encontrado"
echo ""

echo "=== Diagnóstico concluído ==="
```

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*