# Especificação da API (API Spec)

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `03_architecture.md`, `02_srs.md`

---

## 1. Visão Geral

O Colab Infinity atua como um *Drop-in Replacement* para a API da OpenAI. Isso significa que a interface de contrato HTTP segue estritamente a documentação oficial da plataforma OpenAI. O endpoint exposto na máquina local será invocado através da *Base URL* do Orquestrador, geralmente rodando em `http://127.0.0.1:11434`.

Os agentes que consomem o serviço enviarão suas requisições localmente, e o orquestrador efetuará o proxy direto para a rede da instância no Google Colab, garantida por um túnel **Ngrok**.

---

## 2. Endpoints Principais

### 2.1 Chat Completions (Inferência)

Gera respostas de chat seguindo o fluxo contínuo de conversas.

- **URL:** `/v1/chat/completions`
- **Método HTTP:** `POST`
- **Headers:**
  - `Content-Type: application/json`
  - `Authorization: Bearer <qualquer_coisa>` *(Token é ignorado e repassado pro túnel como dummy auth, já que é gratuito)*

#### Payload de Requisição (Request Body)

```json
{
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "messages": [
    {"role": "system", "content": "Você é um assistente útil."},
    {"role": "user", "content": "Descreva a arquitetura do universo em 2 linhas."}
  ],
  "temperature": 0.7,
  "max_tokens": 512,
  "stream": false
}
```
*Suporta também `top_p`, `presence_penalty` e `frequency_penalty` processados internamente.*

#### Respostas (Response Body - Non-stream)
```json
{
  "id": "chatcmpl-colabinf1234",
  "object": "chat.completion",
  "created": 1690000000,
  "model": "mistralai/Mistral-7B-Instruct-v0.2",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "O universo é uma vasta e complexa expansão de espaço-tempo. Abriga bilhões de galáxias regidas pelas leis fundamentais da física."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 30,
    "total_tokens": 55
  }
}
```

#### Comportamento de Streaming (`"stream": true`)
Se `stream: true` for passado, o Colab Infinity via Ngrok retornará chunks HTTP contínuos através de Server-Sent Events (SSE), obedecendo ao padrão estrito OpenAI (`data: {...} \n\n`).

---

### 2.2 Status / Health Check e Métricas

Endpoint exclusivo e estendido do Colab Infinity para leitura de metadados da sessão, usado intensivamente pelo Orquestrador para decisões de *failover*.

- **URL:** `/v1/status`
- **Método HTTP:** `GET`
- **Headers:** Não requerido

#### Resposta Esperada (200 OK)
```json
{
  "status": "healthy",
  "environment": "Google Colab",
  "ngrok_tunnel": "https://a1b2c3d4.ngrok-free.app",
  "model_loaded": "mistralai/Mistral-7B-Instruct-v0.2",
  "vram": {
    "total_mb": 15109,
    "allocated_mb": 8450,
    "utilization_percent": 55.9
  },
  "uptime_seconds": 12450
}
```

---

## 3. Tratamento de Erros no Proxy Local

O Orquestrador Local intercepta as falhas de rede antes que o cliente (Agente Autônomo) trave. Os *Status Codes* possíveis percebidos pelo cliente em caso de erro são:

| HTTP Status | Descrição e Comportamento do Orquestrador |
|---|---|
| **502 Bad Gateway** | Ngrok está inativo ou o Colab travou abruptamente. O Proxy tentará reconectar, pondo a requisição na Fila e iniciando *Failover*. Se não for resolvido após 3 *retries* (em média de 10 min), devolve 502 ao cliente. |
| **503 Service Unavailable** | Rotação de conta ativamente em progresso. O Orquestrador instrui o proxy a retornar 503 com cabeçalho `Retry-After: 60`. |
| **400 Bad Request** | Erro de validação de Schema no JSON das requisições (exemplo, campo `messages` vazio). O Orquestrador apenas repassa os erros originários do modelo/FastAPI. |

## 4. Limitações de Timeout e Rate Limits

1. Devido ao roteamento via **Ngrok Free Tier**, existe um limite maciço intrínseco de conexões simultâneas que, se ultrapassado rapidamente (flood), pode engatilhar um bloqueio `429 Too Many Requests` proveniente do Ngrok (e não do FastAPI).
2. O servidor proxy local possui um *Timeout* prolongado configurado de fábrica (Padrão: 120.000 ms ou 2 minutos) para lidar com cargas de contexto pesado no LLM, assegurando que o Ngrok não corte a conexão cedo demais no caso de longos *prompts*.
