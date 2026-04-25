# Colab Infinity — Especificação da API (API Spec)

**Versão:** 1.0.0  
**Data:** 2025-07-14  
**Status:** Aprovado  
**Referências:** `03_sad.md`, `02_srs.md`

---

## Índice

1. [Visão Geral](#1-visão-geral)
2. [Autenticação](#2-autenticação)
3. [Endpoint: Health Check — GET /health](#3-endpoint-health-check)
4. [Endpoint: Chat Completions — POST /v1/chat/completions](#4-endpoint-chat-completions)
5. [Endpoint: Status da Sessão — GET /v1/status](#5-endpoint-status-da-sessão)
6. [Endpoint: Forçar Checkpoint — POST /v1/checkpoint](#6-endpoint-forçar-checkpoint)
7. [Códigos de Erro](#7-códigos-de-erro)
8. [Rate Limiting](#8-rate-limiting)
9. [Exemplos com curl](#9-exemplos-com-curl)
10. [Exemplos em TypeScript (Ouroboros Runtime)](#10-exemplos-em-typescript-ouroboros-runtime)
11. [Compatibilidade com a API OpenAI](#11-compatibilidade-com-a-api-openai)
12. [Notas de Integração por Consumidor](#12-notas-de-integração-por-consumidor)

---

## 1. Visão Geral

O Colab Infinity expõe uma **API HTTP REST compatível com o padrão OpenAI Chat Completions**.
Qualquer agente ou cliente que suporte a especificação `openai` — incluindo o **Ouroboros Runtime**,
o **Hermes Agent** e o **OpenClaw** — pode apontar para este servidor sem modificações de código.

### 1.1 Camadas de Acesso

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CAMADAS DE ACESSO                            │
├──────────────────────┬──────────────────────────────────────────────┤
│  Camada              │  URL Base                                    │
├──────────────────────┼──────────────────────────────────────────────┤
│  Proxy Local         │  http://127.0.0.1:11434                      │
│  (ponto de entrada   │  → Orquestrador mantém esta URL estável      │
│   para os agentes)   │    independente de trocas de conta           │
├──────────────────────┼──────────────────────────────────────────────┤
│  Túnel Público       │  https://<hash>.ngrok-free.app               │
│  (URL ngrok)         │  → Muda a cada sessão Colab; não usar        │
│                      │    diretamente em produção                   │
├──────────────────────┼──────────────────────────────────────────────┤
│  Servidor Interno    │  http://127.0.0.1:8000                       │
│  (FastAPI no Colab)  │  → Apenas visível dentro do notebook         │
└──────────────────────┴──────────────────────────────────────────────┘
```

> **Regra de ouro:** Todos os agentes (Ouroboros, Hermes, OpenClaw) devem sempre apontar para
> `http://127.0.0.1:11434`. O orquestrador local é responsável por rotear essa requisição para
> a sessão Colab ativa no momento, de forma transparente.

### 1.2 Versionamento

A API usa versionamento por prefixo de rota (`/v1/`). Versões futuras utilizarão `/v2/`. Não há
negociação de versão via header Accept.

### 1.3 Formato Padrão

| Propriedade       | Valor                                              |
|-------------------|----------------------------------------------------|
| Content-Type      | `application/json; charset=utf-8`                  |
| Encoding          | UTF-8                                              |
| Protocolo HTTP    | HTTP/1.1 e HTTP/2                                  |
| Streaming         | Server-Sent Events (SSE) via `Transfer-Encoding: chunked` |
| Fuso horário      | Todos os timestamps em ISO 8601 UTC                |

---

## 2. Autenticação

A autenticação é **desabilitada por padrão** para uso local. Pode ser habilitada via
`colab_infinity.config.yaml` para ambientes com exposição pública.

### 2.1 Header de Autenticação

Quando habilitada (`require_auth: true`), toda requisição deve incluir:

```
Authorization: Bearer <COLAB_INFINITY_API_KEY>
```

### 2.2 Configuração

No arquivo `~/.colab_infinity/colab_infinity.config.yaml`:

```yaml
server:
  require_auth: false                          # true para habilitar
  api_key: "ci-sk-xxxxxxxxxxxxxxxxxxxxxxxx"   # ignorado se require_auth: false
```

Variável de ambiente alternativa (tem precedência sobre o YAML):

```bash
export COLAB_INFINITY_API_KEY="ci-sk-xxxxxxxxxxxxxxxxxxxxxxxx"
export COLAB_INFINITY_REQUIRE_AUTH="true"
```

### 2.3 Comportamento por Estado de Autenticação

| `require_auth` | Header presente | Resultado              |
|----------------|-----------------|------------------------|
| `false`        | Qualquer valor  | Requisição aceita      |
| `true`         | Correto         | Requisição aceita      |
| `true`         | Incorreto       | `401 UNAUTHORIZED`     |
| `true`         | Ausente         | `401 UNAUTHORIZED`     |

---

## 3. Endpoint: Health Check

### `GET /health`

Verifica o estado operacional do servidor. Usado pelo orquestrador local para monitorar
vivacidade da sessão Colab ativa.

#### Request

Sem corpo. Sem parâmetros de query.

```
GET /health HTTP/1.1
Host: 127.0.0.1:11434
```

#### Response — 200 OK (servidor pronto)

```json
{
  "status": "ok",
  "timestamp": "2025-07-14T14:22:10Z",
  "uptime_seconds": 18430,
  "model": {
    "id": "mistralai/Mistral-7B-Instruct-v0.2",
    "loaded": true,
    "quantization": "4bit",
    "context_length": 4096
  },
  "runtime": {
    "device": "cuda",
    "gpu_name": "Tesla T4",
    "vram_used_mb": 8421,
    "vram_total_mb": 15360,
    "vram_free_mb": 6939
  },
  "session": {
    "id": "ci_sess_20250714_121500",
    "account_index": 1,
    "requests_served": 247,
    "tokens_generated": 98340,
    "estimated_quota_remaining_minutes": 312
  },
  "orchestrator": {
    "pool_size": 4,
    "accounts_available": 3,
    "checkpoint_last_saved": "2025-07-14T14:20:00Z"
  }
}
```

#### Response — 503 Service Unavailable (modelo ainda carregando)

```json
{
  "status": "loading",
  "timestamp": "2025-07-14T14:10:01Z",
  "uptime_seconds": 61,
  "model": {
    "id": "mistralai/Mistral-7B-Instruct-v0.2",
    "loaded": false,
    "load_progress_pct": 52
  }
}
```

#### Campos da Resposta

| Campo                                     | Tipo    | Descrição                                                     |
|-------------------------------------------|---------|---------------------------------------------------------------|
| `status`                                  | string  | `"ok"`, `"loading"`, `"error"`, `"shutting_down"`            |
| `uptime_seconds`                          | integer | Segundos desde o início da sessão Colab                       |
| `model.id`                                | string  | Identificador HuggingFace do modelo                           |
| `model.loaded`                            | boolean | `true` se o modelo está pronto para inferência                |
| `model.quantization`                      | string  | `"4bit"`, `"8bit"` ou `"none"`                               |
| `runtime.device`                          | string  | `"cuda"` ou `"cpu"`                                           |
| `runtime.gpu_name`                        | string  | Nome da GPU alocada pelo Colab                                |
| `session.id`                              | string  | Identificador único desta sessão                              |
| `session.account_index`                   | integer | Índice da conta ativa no pool (0-based)                       |
| `session.estimated_quota_remaining_minutes`| integer | Minutos estimados antes do esgotamento da cota               |
| `orchestrator.pool_size`                  | integer | Total de contas no pool                                       |
| `orchestrator.accounts_available`         | integer | Contas disponíveis (não exauridas)                            |
| `orchestrator.checkpoint_last_saved`      | string  | ISO 8601 do último checkpoint bem-sucedido                    |

---

## 4. Endpoint: Chat Completions

### `POST /v1/chat/completions`

Endpoint principal de inferência. **Totalmente compatível com a API `chat/completions` da OpenAI.**

### 4.1 Request

#### Headers

```
Content-Type: application/json
Authorization: Bearer <API_KEY>    ← apenas se require_auth: true
```

#### Corpo da Requisição (JSON Completo)

```json
{
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "messages": [
    {
      "role": "system",
      "content": "Você é um assistente de engenharia de software especializado em TypeScript e sistemas distribuídos."
    },
    {
      "role": "user",
      "content": "Explique o padrão de design Observer em 3 parágrafos concisos."
    },
    {
      "role": "assistant",
      "content": "O padrão Observer define uma dependência um-para-muitos entre objetos..."
    },
    {
      "role": "user",
      "content": "Como implementar isso em TypeScript com tipagem forte?"
    }
  ],
  "temperature": 0.7,
  "top_p": 0.95,
  "top_k": 50,
  "max_tokens": 1024,
  "stream": false,
  "stop": ["\n###", "<|endoftext|>"],
  "presence_penalty": 0.0,
  "frequency_penalty": 0.0,
  "n": 1,
  "user": "ouroboros-daemon-wave-42"
}
```

#### Descrição dos Campos do Request

| Campo               | Tipo              | Obrigatório | Padrão   | Descrição                                                                    |
|---------------------|-------------------|-------------|----------|------------------------------------------------------------------------------|
| `model`             | string            | Não         | (config) | ID do modelo. Se omitido, usa o modelo carregado na sessão ativa.            |
| `messages`          | array             | **Sim**     | —        | Histórico de mensagens. Mínimo: 1 mensagem.                                  |
| `messages[].role`   | string            | **Sim**     | —        | `"system"`, `"user"` ou `"assistant"`.                                       |
| `messages[].content`| string            | **Sim**     | —        | Conteúdo textual da mensagem.                                                |
| `temperature`       | float             | Não         | `0.7`    | Aleatoriedade na geração. Range: `[0.0, 2.0]`.                               |
| `top_p`             | float             | Não         | `0.95`   | Nucleus sampling. Range: `(0.0, 1.0]`.                                       |
| `top_k`             | integer           | Não         | `50`     | Top-K sampling. `0` desabilita. Extensão proprietária — ausente na OpenAI.   |
| `max_tokens`        | integer           | Não         | `512`    | Máximo de tokens a gerar. Range: `[1, 4096]`.                                |
| `stream`            | boolean           | Não         | `false`  | Se `true`, retorna SSE em vez de JSON completo.                              |
| `stop`              | string \| array   | Não         | `null`   | Sequências de parada. Máximo 4 itens.                                        |
| `presence_penalty`  | float             | Não         | `0.0`    | Penaliza tokens já presentes. Range: `[-2.0, 2.0]`.                          |
| `frequency_penalty` | float             | Não         | `0.0`    | Penaliza tokens frequentes. Range: `[-2.0, 2.0]`.                            |
| `n`                 | integer           | Não         | `1`      | Número de completions. **Máximo suportado: 1.**                              |
| `user`              | string            | Não         | `null`   | Identificador do agente consumidor. Registrado em logs e métricas.           |

---

### 4.2 Response — Modo Síncrono (`stream: false`)

**HTTP 200 OK**

```json
{
  "id": "chatcmpl-ci_sess_20250714_121500-00247",
  "object": "chat.completion",
  "created": 1720965730,
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Para implementar o padrão Observer em TypeScript com tipagem forte, você pode usar interfaces genéricas...\n\n```typescript\ninterface Observer<T> {\n  update(data: T): void;\n}\n\ninterface Observable<T> {\n  subscribe(observer: Observer<T>): void;\n  unsubscribe(observer: Observer<T>): void;\n  notify(data: T): void;\n}\n```\n\nEssa abordagem garante que tanto os observadores quanto o sujeito sejam tipados..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 187,
    "completion_tokens": 312,
    "total_tokens": 499
  },
  "system_fingerprint": "ci-v1.0-t4-4bit",
  "x_colab_infinity": {
    "session_id": "ci_sess_20250714_121500",
    "account_index": 1,
    "inference_ms": 4210,
    "quota_remaining_minutes": 311
  }
}
```

#### Campos da Resposta Síncrona

| Campo                          | Tipo    | Descrição                                                        |
|--------------------------------|---------|------------------------------------------------------------------|
| `id`                           | string  | ID único da completion, prefixado com `chatcmpl-`.               |
| `object`                       | string  | Sempre `"chat.completion"`.                                      |
| `created`                      | integer | Unix timestamp UTC da geração.                                   |
| `model`                        | string  | Modelo que gerou a resposta.                                     |
| `choices[].index`              | integer | Sempre `0` (pois `n=1`).                                         |
| `choices[].message.role`       | string  | Sempre `"assistant"`.                                            |
| `choices[].message.content`    | string  | Texto gerado.                                                    |
| `choices[].finish_reason`      | string  | `"stop"`, `"length"` ou `"error"`.                              |
| `usage.prompt_tokens`          | integer | Tokens contados no prompt de entrada.                            |
| `usage.completion_tokens`      | integer | Tokens gerados na resposta.                                      |
| `usage.total_tokens`           | integer | Soma dos dois campos anteriores.                                 |
| `system_fingerprint`           | string  | Identificador do ambiente de execução (hardware + versão).       |
| `x_colab_infinity`             | object  | **Extensão proprietária.** Metadados da sessão Colab Infinity.   |
| `x_colab_infinity.session_id`  | string  | ID da sessão Colab ativa.                                        |
| `x_colab_infinity.account_index`| integer| Índice da conta Google ativa no pool.                           |
| `x_colab_infinity.inference_ms`| integer | Tempo de inferência em milissegundos.                            |
| `x_colab_infinity.quota_remaining_minutes` | integer | Minutos estimados de cota restante.             |

---

### 4.3 Response — Modo Streaming (`stream: true`)

**HTTP 200 OK**

```
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked
```

Cada chunk tem o formato `data: <JSON>\n\n`. O cliente deve processar linha a linha:

```
data: {"id":"chatcmpl-ci_sess_20250714_121500-00247","object":"chat.completion.chunk","created":1720965730,"model":"mistralai/Mistral-7B-Instruct-v0.2","choices":[{"index":0,"delta":{"role":"assistant","content":"Para"},"finish_reason":null}]}

data: {"id":"chatcmpl-ci_sess_20250714_121500-00247","object":"chat.completion.chunk","created":1720965730,"model":"mistralai/Mistral-7B-Instruct-v0.2","choices":[{"index":0,"delta":{"content":" implementar"},"finish_reason":null}]}

data: {"id":"chatcmpl-ci_sess_20250714_121500-00247","object":"chat.completion.chunk","created":1720965730,"model":"mistralai/Mistral-7B-Instruct-v0.2","choices":[{"index":0,"delta":{"content":" o padrão"},"finish_reason":null}]}

data: {"id":"chatcmpl-ci_sess_20250714_121500-00247","object":"chat.completion.chunk","created":1720965730,"model":"mistralai/Mistral-7B-Instruct-v0.2","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

**Regras do protocolo SSE nesta implementação:**

- O primeiro chunk sempre contém `"role": "assistant"` no `delta`.
- O chunk final antes de `[DONE]` tem `delta: {}` e `finish_reason` preenchido.
- `data: [DONE]` encerra o stream; o cliente deve fechar a conexão após recebê-lo.
- Chunks de `x_colab_infinity` são enviados apenas no **último chunk** antes de `[DONE]`.

---

## 5. Endpoint: Status da Sessão

### `GET /v1/status`

Retorna informações detalhadas sobre o estado operacional completo: sessão Colab, pool de contas,
modelo, checkpoint e túnel ngrok.

#### Response — 200 OK

```json
{
  "session": {
    "id": "ci_sess_20250714_121500",
    "started_at": "2025-07-14T12:15:00Z",
    "uptime_seconds": 18430,
    "colab_runtime_type": "GPU (T4)",
    "estimated_expiry_at": "2025-07-15T00:15:00Z",
    "estimated_remaining_seconds": 42170
  },
  "model": {
    "id": "mistralai/Mistral-7B-Instruct-v0.2",
    "quantization": "4bit",
    "context_length": 4096,
    "loaded": true,
    "load_duration_seconds": 193
  },
  "account": {
    "index": 1,
    "email_masked": "col***@gmail.com",
    "session_requests": 247,
    "session_tokens_generated": 98340,
    "quota_used_minutes": 307,
    "quota_limit_minutes": 720
  },
  "pool": {
    "total": 4,
    "available": 3,
    "exhausted": 1,
    "banned": 0,
    "in_cooldown": 0,
    "next_account_index": 2
  },
  "checkpoint": {
    "last_saved_at": "2025-07-14T14:20:00Z",
    "total_saved": 18,
    "drive_path": "hermes_infinito/checkpoints/ci_ckpt_20250714_142000.json",
    "interval_seconds": 300
  },
  "tunnel": {
    "provider": "ngrok",
    "public_url": "https://a1b2c3d4e5f6.ngrok-free.app",
    "region": "us",
    "active_connections": 1
  }
}
```

---

## 6. Endpoint: Forçar Checkpoint

### `POST /v1/checkpoint`

Força o salvamento imediato do estado da sessão no Google Drive (conta armazém). Em operação
normal, checkpoints são salvos automaticamente a cada `checkpoint.interval_seconds` (padrão: 300s)
ou quando a cota restante cai abaixo de `checkpoint.quota_threshold_minutes` (padrão: 30min).

#### Request (corpo opcional)

```json
{
  "reason": "pre_account_switch",
  "include_conversation_cache": false
}
```

| Campo                       | Tipo    | Padrão  | Descrição                                                       |
|-----------------------------|---------|---------|-----------------------------------------------------------------|
| `reason`                    | string  | `null`  | Motivo registrado no log e no nome do arquivo de checkpoint.    |
| `include_conversation_cache`| boolean | `false` | Inclui cache de conversas recentes no checkpoint (aumenta tamanho). |

#### Response — 200 OK

```json
{
  "status": "saved",
  "checkpoint_id": "ci_ckpt_20250714_160000_pre_account_switch",
  "drive_path": "colab_infinity/checkpoints/ci_ckpt_20250714_160000_pre_account_switch.json",
  "size_bytes": 5120,
  "saved_at": "2025-07-14T16:00:00Z",
  "duration_ms": 1840
}
```

#### Response — 409 Conflict (checkpoint em progresso)

```json
{
  "error": {
    "code": "CHECKPOINT_IN_PROGRESS",
    "message": "Um checkpoint já está sendo salvo. Aguarde e tente novamente.",
    "retry_after_seconds": 8
  }
}
```

---

## 7. Códigos de Erro

### 7.1 Envelope Padrão de Erro

```json
{
  "error": {
    "code": "CODIGO_INTERNO",
    "message": "Descrição legível por humanos.",
    "type": "categoria_do_erro",
    "param": "nome_do_campo_problematico_ou_null",
    "request_id": "req_20250714_a7f3b2"
  }
}
```

### 7.2 Tabela Completa de Erros

| HTTP | Código Interno               | Tipo               | Causa                                               | Ação Recomendada                                          |
|------|------------------------------|--------------------|-----------------------------------------------------|-----------------------------------------------------------|
| 400  | `INVALID_REQUEST`            | `invalid_request`  | JSON malformado ou campo obrigatório ausente.        | Corrigir o corpo da requisição.                           |
| 400  | `INVALID_MESSAGES_FORMAT`    | `invalid_request`  | `messages` vazio ou com `role` inválido.            | Verificar estrutura e valores de `messages`.              |
| 400  | `MAX_TOKENS_EXCEEDED`        | `invalid_request`  | `max_tokens` > limite do modelo (4096).             | Reduzir `max_tokens` para no máximo 4096.                 |
| 400  | `UNSUPPORTED_N`              | `invalid_request`  | `n` > 1 solicitado (não suportado).                 | Usar `n: 1` ou omitir o campo.                            |
| 400  | `PARAM_OUT_OF_RANGE`         | `invalid_request`  | Parâmetro numérico fora do intervalo válido.        | Verificar o campo indicado em `param` e seu range.        |
| 401  | `UNAUTHORIZED`               | `auth_error`       | Header `Authorization` ausente ou inválido.         | Incluir `Bearer <API_KEY>` correto.                       |
| 429  | `RATE_LIMIT_EXCEEDED`        | `rate_limit`       | Muitas requisições por segundo ou por minuto.       | Aguardar `Retry-After` segundos antes de retentar.        |
| 500  | `MODEL_INFERENCE_ERROR`      | `server_error`     | Exceção durante a inferência do modelo.             | Retentar 1 vez; reportar se persistir.                    |
| 503  | `MODEL_LOADING`              | `server_error`     | Modelo ainda carregando; servidor não está pronto.  | Aguardar 20–30s e retentar (ver `/health`).               |
| 503  | `SESSION_SWITCHING`          | `server_error`     | Orquestrador trocando de conta Google.              | Aguardar 60–120s e retentar automaticamente.              |
| 503  | `GPU_UNAVAILABLE`            | `server_error`     | Colab não alocou GPU; runtime em CPU.               | Verificar disponibilidade de GPU e reiniciar sessão.      |
| 503  | `POOL_EXHAUSTED`             | `server_error`     | Todas as contas do pool estão exauridas ou em ban.  | Aguardar cooldown (24h) ou adicionar nova conta ao pool.  |
| 504  | `INFERENCE_TIMEOUT`          | `server_error`     | Inferência excedeu `inference_timeout_seconds`.     | Reduzir `max_tokens` ou simplificar o prompt.             |

### 7.3 Exemplo de Resposta de Erro Tipada

```json
{
  "error": {
    "code": "PARAM_OUT_OF_RANGE",
    "message": "O campo 'temperature' deve estar entre 0.0 e 2.0. Recebido: 3.5",
    "type": "invalid_request",
    "param": "temperature",
    "request_id": "req_20250714_d9f3a1"
  }
}
```

---

## 8. Rate Limiting

### 8.1 Limites Padrão

| Janela      | Limite               | Escopo              |
|-------------|----------------------|---------------------|
| Por segundo | 2 requisições        | Por IP de origem    |
| Por minuto  | 30 requisições       | Por IP de origem    |
| Por hora    | 300 requisições      | Global (sessão)     |

> Os limites são configuráveis em `colab_infinity.config.yaml` → `server.rate_limit`.

### 8.2 Headers de Rate Limit

Presentes em **todas** as respostas:

```
X-RateLimit-Limit-Second: 2
X-RateLimit-Remaining-Second: 1
X-RateLimit-Limit-Minute: 30
X-RateLimit-Remaining-Minute: 27
X-RateLimit-Reset: 1720965790
```

### 8.3 Resposta ao Exceder o Limite

```
HTTP/1.1 429 Too Many Requests
Retry-After: 1
```

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Limite de 2 requisições por segundo excedido. Aguarde 1 segundo.",
    "type": "rate_limit",
    "param": null,
    "request_id": "req_20250714_e7b2c9"
  }
}
```

---

## 9. Exemplos com curl

### 9.1 Health Check Básico

```bash
curl -s http://127.0.0.1:11434/health | python3 -m json.tool
```

### 9.2 Chat Completion Síncrono

```bash
curl -s http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {
        "role": "system",
        "content": "Você é um assistente técnico especializado em TypeScript."
      },
      {
        "role": "user",
        "content": "O que é um tipo discriminado (discriminated union) em TypeScript?"
      }
    ],
    "temperature": 0.5,
    "max_tokens": 512,
    "stream": false,
    "user": "cli-test"
  }' | python3 -m json.tool
```

### 9.3 Chat Completion com Streaming

```bash
curl -s http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Escreva um haiku sobre sistemas distribuídos."}],
    "stream": true,
    "max_tokens": 100
  }' \
  --no-buffer \
  | grep "^data:" \
  | grep -v "\[DONE\]" \
  | sed 's/^data: //' \
  | python3 -c "
import sys, json
for line in sys.stdin:
    chunk = json.loads(line.strip())
    delta = chunk['choices'][0]['delta']
    if 'content' in delta:
        print(delta['content'], end='', flush=True)
print()
"
```

### 9.4 Forçar Checkpoint

```bash
curl -s -X POST http://127.0.0.1:11434/v1/checkpoint \
  -H "Content-Type: application/json" \
  -d '{"reason": "manual_backup", "include_conversation_cache": false}' \
  | python3 -m json.tool
```

### 9.5 Consultar Status Completo da Sessão

```bash
curl -s http://127.0.0.1:11434/v1/status \
  | python3 -c "
import sys, json
s = json.load(sys.stdin)
print(f\"Sessão : {s['session']['id']}\")
print(f\"Conta  : {s['account']['index']} ({s['account']['email_masked']})\")
print(f\"Modelo : {s['model']['id']} ({s['model']['quantization']})\")
print(f\"Cota   : {s['account']['quota_used_minutes']} / {s['account']['quota_limit_minutes']} min\")
print(f\"Pool   : {s['pool']['available']}/{s['pool']['total']} contas disponíveis\")
"
```

### 9.6 Teste com Autenticação Habilitada

```bash
export CI_API_KEY="ci-sk-xxxxxxxxxxxxxxxxxxxxxxxx"

curl -s http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${CI_API_KEY}" \
  -d '{
    "messages": [{"role": "user", "content": "ping"}],
    "max_tokens": 5
  }'
```

---

## 10. Exemplos em TypeScript (Ouroboros Runtime)

### 10.1 Chamada Direta via `fetch` (Daemon RPC Handler)

Este snippet ilustra como um handler RPC do Ouroboros Daemon faria uma chamada ao Colab Infinity.
Trata-se de um exemplo conceitual do padrão de integração — não é código de produção finalizado.

```typescript
// Exemplo ilustrativo: handler de RPC no Ouroboros Daemon
// Arquivo de referência conceitual: src/rpc/llm-handler.ts

interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

interface ColabInfinityRequest {
  messages: ChatMessage[];
  model?: string;
  temperature?: number;
  max_tokens?: number;
  stream?: boolean;
  user?: string;
}

interface ColabInfinityResponse {
  id: string;
  object: "chat.completion";
  created: number;
  model: string;
  choices: Array<{
    index: number;
    message: ChatMessage;
    finish_reason: "stop" | "length" | "error";
  }>;
  usage: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
  };
  x_colab_infinity: {
    session_id: string;
    account_index: number;
    inference_ms: number;
    quota_remaining_minutes: number;
  };
}

// Configuração do endpoint — lida de variável de ambiente
const COLAB_INFINITY_BASE_URL =
  process.env.COLAB_INFINITY_BASE_URL ?? "http://127.0.0.1:11434";
const COLAB_INFINITY_API_KEY =
  process.env.COLAB_INFINITY_API_KEY ?? null;

async function callColabInfinity(
  messages: ChatMessage[],
  options: Partial<ColabInfinityRequest> = {}
): Promise<ColabInfinityResponse> {
  const payload: ColabInfinityRequest = {
    messages,
    model: "mistralai/Mistral-7B-Instruct-v0.2",
    temperature: 0.7,
    max_tokens: 1024,
    stream: false,
    user: "ouroboros-daemon",
    ...options,
  };

  const headers: Record<string, string> = {
    "Content-Type": "application/json",
  };
  if (COLAB_INFINITY_API_KEY) {
    headers["Authorization"] = `Bearer ${COLAB_INFINITY_API_KEY}`;
  }

  const response = await fetch(
    `${COLAB_INFINITY_BASE_URL}/v1/chat/completions`,
    {
      method: "POST",
      headers,
      body: JSON.stringify(payload),
      signal: AbortSignal.timeout(120_000), // 120s timeout
    }
  );

  if (!response.ok) {
    const errorBody = await response.json().catch(() => ({}));
    throw new Error(
      `ColabInfinity error ${response.status}: ${errorBody?.error?.message ?? "Unknown error"}`
    );
  }

  return response.json() as Promise<ColabInfinityResponse>;
}
```

### 10.2 Uso no Contexto de um Agente Ouroboros (Exemplo Conceitual)

```typescript
// Exemplo ilustrativo: invocação de LLM em um agente do conselho
// Padrão de uso dentro de um Wave handler do Ouroboros Runtime

async function runArchitectAgent(
  task: string,
  context: string[]
): Promise<string> {
  const messages: ChatMessage[] = [
    {
      role: "system",
      content:
        "Você é o agente Architect do Ouroboros Runtime. " +
        "Sua especialidade é design de sistemas, arquitetura de software " +
        "e criação de especificações técnicas detalhadas. " +
        "Seja preciso, estruturado e use diagramas ASCII quando relevante.",
    },
    // Injetar contexto de memória do SQLite (histórico relevante)
    ...context.map((c) => ({ role: "user" as const, content: c })),
    {
      role: "user",
      content: task,
    },
  ];

  const result = await callColabInfinity(messages, {
    temperature: 0.4,    // Mais determinístico para tarefas de arquitetura
    max_tokens: 2048,
    user: "ouroboros-architect-agent",
  });

  const content = result.choices[0]?.message?.content;
  if (!content) {
    throw new Error("Resposta vazia do modelo LLM.");
  }

  // Log de métricas para o sistema de saúde do Ouroboros
  console.log(
    `[LLM] tokens=${result.usage.total_tokens} ` +
    `ms=${result.x_colab_infinity.inference_ms} ` +
    `quota_left=${result.x_colab_infinity.quota_remaining_minutes}min`
  );

  return content;
}
```

### 10.3 Verificação de Disponibilidade antes de Usar (Health Gate)

```typescript
// Exemplo ilustrativo: gate de verificação de saúde
// Conceito: verificar antes de disparar uma Wave que depende de LLM

async function assertColabInfinityReady(
  maxWaitMs = 300_000
): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < maxWaitMs) {
    try {
      const res = await fetch(`${COLAB_INFINITY_BASE_URL}/health`);
      const body = await res.json();
      if (body.status === "ok" && body.model?.loaded === true) {
        console.log(`[ColabInfinity] Pronto. Modelo: ${body.model.id}`);
        return;
      }
      if (body.status === "loading") {
        console.log(`[ColabInfinity] Carregando modelo... ${body.model?.load_progress_pct ?? "?"}%`);
      }
    } catch {
      console.log("[ColabInfinity] Servidor não disponível ainda. Aguardando...");
    }
    await new Promise((r) => setTimeout(r, 15_000)); // aguardar 15s entre tentativas
  }
  throw new Error(
    `ColabInfinity não ficou pronto em ${maxWaitMs / 1000}s.`
  );
}
```

---

## 11. Compatibilidade com a API OpenAI

A tabela abaixo compara o suporte do Colab Infinity com a API oficial `openai` (GPT-4 / GPT-3.5).

| Campo / Feature                | OpenAI   | Colab Infinity  | Observação                                                     |
|--------------------------------|----------|-----------------|----------------------------------------------------------------|
| `model`                        | ✅        | ✅ Parcial       | Aceito, mas o modelo em execução não muda dinamicamente.       |
| `messages` (text)              | ✅        | ✅               | Suporte completo: `system`, `user`, `assistant`.               |
| `messages` (array de content)  | ✅        | ❌               | Apenas string. Multimodal/vision não suportado.                |
| `temperature`                  | ✅        | ✅               | Range idêntico `[0.0, 2.0]`.                                   |
| `top_p`                        | ✅        | ✅               | Suporte total.                                                 |
| `top_k`                        | ❌        | ✅ Extensão      | Parâmetro proprietário, ausente na OpenAI.                     |
| `max_tokens`                   | ✅        | ✅               | Limite máximo: 4096.                                           |
| `stream`                       | ✅        | ✅               | SSE compatível com openai SDK.                                 |
| `stop`                         | ✅        | ✅               | Máximo de 4 sequências.                                        |
| `n`                            | ✅        | ⚠️ Parcial       | Apenas `n=1`. Retorna erro para `n>1`.                         |
| `presence_penalty`             | ✅        | ✅               | Suporte total.                                                 |
| `frequency_penalty`            | ✅        | ✅               | Suporte total.                                                 |
| `logit_bias`                   | ✅        | ❌               | Não implementado.                                              |
| `user`                         | ✅        | ✅               | Registrado em logs; não afeta geração.                         |
| `tools` / `function_calling`   | ✅        | ❌               | Não suportado nesta versão.                                    |
| `response_format` (JSON mode)  | ✅        | ❌               | Não implementado.                                              |
| `seed`                         | ✅        | ⚠️ Parcial       | Aceito, sem garantia de reprodutibilidade perfeita.            |
| `stream_options`               | ✅        | ❌               | Não implementado.                                              |
| `messages[].name`              | ✅        | ⚠️ Ignorado      | Aceito sem erro, mas descartado silenciosamente.               |
| Imagens / Vision               | ✅ (V)    | ❌               | Fora do escopo desta versão.                                   |
| `x_colab_infinity` (extensão)  | ❌        | ✅               | Campo proprietário com metadados da sessão Colab.              |

**Legenda:** ✅ Suportado · ⚠️ Parcialmente suportado · ❌ Não suportado

---

## 12. Notas de Integração por Consumidor

### 12.1 Ouroboros Runtime

| Configuração                          | Valor                                  |
|---------------------------------------|----------------------------------------|
| Variável de ambiente                  | `COLAB_INFINITY_BASE_URL`              |
| Valor padrão esperado                 | `http://127.0.0.1:11434`              |
| Variável de API Key                   | `COLAB_INFINITY_API_KEY`               |
| Timeout recomendado                   | `120000` ms (120s)                     |
| Comportamento em `503 SESSION_SWITCHING` | Retry com backoff de 90s, max 5x   |
| Campo `user` recomendado              | `"ouroboros-<agent_name>-wave-<id>"`  |

O Ouroboros Runtime deve verificar `/health` antes de iniciar qualquer Wave que dependa de inferência
de LLM. O campo `x_colab_infinity.quota_remaining_minutes` pode ser usado para acionar alertas
preventivos quando a cota estiver baixa.

### 12.2 Hermes Agent

| Configuração                | Valor                                            |
|-----------------------------|--------------------------------------------------|
| Chave de config             | `llm.base_url`                                   |
| Valor                       | `http://127.0.0.1:11434/v1`                      |
| Chave de modelo             | `llm.model`                                      |
| Valor                       | `mistralai/Mistral-7B-Instruct-v0.2`            |
| Provider                    | `openai_compatible`                              |

```yaml
# Trecho ilustrativo de ~/.config/hermes-agent/config.yaml
llm:
  provider: openai_compatible
  base_url: "http://127.0.0.1:11434/v1"
  api_key: null
  model: "mistralai/Mistral-7B-Instruct-v0.2"
  timeout_seconds: 120
  max_retries: 3
```

### 12.3 OpenClaw

O OpenClaw, sendo compatível com a especificação OpenAI, deve ser configurado com:

```
OPENAI_API_BASE=http://127.0.0.1:11434/v1
OPENAI_API_KEY=dummy   # qualquer valor se require_auth: false
```

Nenhuma modificação de código é necessária no OpenClaw.

### 12.4 Qualquer Client openai SDK (Python)

```python
# Exemplo ilustrativo de uso com openai SDK oficial
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:11434/v1",
    api_key="dummy",  # ignorado se require_auth: false
)

response = client.chat.completions.create(
    model="mistralai/Mistral-7B-Instruct-v0.2",
    messages=[{"role": "user", "content": "Olá, Colab Infinity!"}],
    max_tokens=50,
)
print(response.choices[0].message.content)
```

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*