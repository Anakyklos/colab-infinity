# Especificação da API (API Spec)

Este documento define a interface HTTP exposta pelo FastAPI no Google Colab. A interface obedece estritamente às especificações essenciais de inferência (Chat Completions) publicadas pela OpenAI para garantir "plug and play" com Ouroboros Runtime, Hermes Agent e OpenClaw.

## Informações Base
- **Endpoint principal:** `POST /v1/chat/completions`
- **Autenticação:** Baseada em Bearer Token (opcional, validado via Header `Authorization: Bearer <token_arbitrario>`).
- **Content-Type:** `application/json`

## Requisição (Request)

O corpo da requisição deve ser um objeto JSON.

### Campos Suportados:

- `model` (string, **obrigatório**): Ignorado logicamente pelo servidor (pois o modelo é carregado hardcoded no script do Colab, ex: Mistral-7B), mas necessário para evitar erros de validação do cliente. Ex: "mistral".
- `messages` (array de objetos, **obrigatório**): Contém o histórico da conversa. Objetos contêm as propriedades `role` (system, user, assistant) e `content` (string).
- `temperature` (float, opcional): Default `0.7`. Valores entre 0.0 e 2.0. Controla a aleatoriedade.
- `max_tokens` (integer, opcional): Default `1024`. Número máximo de tokens gerados na resposta.
- `stream` (boolean, opcional): Default `false`. Suporte a SSE (Server-Sent Events) ainda não totalmente estável na v1 da infraestrutura, recomendado `false` (síncrono) para Ouroboros.

### Exemplo de Payload JSON (Requisição)

```json
{
  "model": "mistral-7b-instruct",
  "messages": [
    {"role": "system", "content": "Você é a persona Core do Ouroboros. Responda apenas em JSON."},
    {"role": "user", "content": "Analise o input e gere o plano de ação."}
  ],
  "temperature": 0.1,
  "max_tokens": 512
}
```

## Resposta (Response)

A resposta reflete exatamente a estrutura da API da OpenAI.

### Exemplo de Payload JSON (Resposta 200 OK)

```json
{
  "id": "chatcmpl-colab-inf-8x9y0z",
  "object": "chat.completion",
  "created": 1709403194,
  "model": "mistral-7b-instruct-v0.2",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "{\"status\":\"success\", \"plan\": [\"Ler arquivo\", \"Criar AST\"]}"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 32,
    "completion_tokens": 18,
    "total_tokens": 50
  }
}
```

## Tratamento de Erros

O servidor retornará códigos HTTP apropriados em caso de falha, padronizados no objeto de "error" da OpenAI:

- `400 Bad Request`: JSON mal formatado ou requisição sem o campo `messages`.
- `401 Unauthorized`: Token de Bearer inválido (se a verificação for ativada no Colab).
- `429 Too Many Requests`: GPU sob excessiva contenção local ou enfileiramento (timeout de resposta excedido antes da geração).
- `500 Internal Server Error`: Erro de carregamento (OOM, Out of Memory CUDA) ou colapso do runtime FastAPI.

### Exemplo de Resposta de Erro (400 Bad Request)
```json
{
  "error": {
    "message": "Invalid prompt format or missing 'messages' array.",
    "type": "invalid_request_error",
    "param": "messages",
    "code": "missing_required_parameter"
  }
}
```

## Consumo da API

### Usando cURL

```bash
curl https://meu-tunel-cloudflare.trycloudflare.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer my-secret-key" \
  -d '{
    "model": "mistral",
    "messages": [{"role": "user", "content": "Teste de conexão!"}]
  }'
```

### Usando TypeScript (Contexto Ouroboros / SDK OpenAI)

Para utilizar esta API dentro do daemon do Ouroboros escrito em TypeScript, recomenda-se passar uma configuração customizada para o cliente oficial da OpenAI ou similar (ex: via fetch):

```typescript
import OpenAI from 'openai';

// Inicialização com a base URL fornecida pelo Orquestrador do Colab Infinity
const openai = new OpenAI({
  baseURL: "https://meu-tunel-cloudflare.trycloudflare.com/v1", // URL injetada via Env
  apiKey: "my-secret-key", // Chave simulada do Colab Infinity
});

async function main() {
  const completion = await openai.chat.completions.create({
    messages: [{ role: "system", content: "Você é o Core." }],
    model: "mistral-7b-instruct", // Será processado pelo modelo quantizado localmente
  });

  console.log(completion.choices[0].message.content);
}

main();
```
