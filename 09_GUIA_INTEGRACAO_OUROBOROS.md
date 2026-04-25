# Guia de Integração com Ouroboros Runtime

Este documento trata exclusivamente das modificações necessárias no daemon e ambiente do Ouroboros para acoplar a sua inteligência artificial geradora à infraestrutura contínua do Colab Infinity.

## 1. Configurando o Ambiente (.env) do Ouroboros

Localize o arquivo de configuração de variáveis no diretório do seu daemon Ouroboros. Edite (ou crie um mecanismo no seu arquivo Bash Profile se preferir):

```env
# No Ouroboros
LLM_PROVIDER=openai
OPENAI_API_KEY=colab-inf-default-key # O Token bearer estático escolhido
OPENAI_BASE_URL=https://<dinamico>-cloudflare.trycloudflare.com/v1
```

O orquestrador do Colab Infinity substituirá programaticamente a linha `OPENAI_BASE_URL` no `.env` do Ouroboros após cada troca de conta (failover).

## 2. Ajustes no Daemon do Ouroboros (TypeScript)

Para garantir resiliência aos "solavancos" que ocorrem no rodízio de Worker Accounts (aprox. 5 min de downtime na rede), as funções do RPC do Ouroboros que chamam as Completion APIs precisam de um middleware de retry de alta latência.

**Exemplo de Retry Inteligente na camada LLM do Daemon (`src/services/llm.ts`):**

```typescript
import OpenAI from "openai";

// Instancia lendo do process.env (ou recarregando env localmente)
const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_BASE_URL,
});

async function callColabInfinityWithRetry(messages: any[], maxRetries = 10): Promise<string> {
    let attempt = 0;
    while (attempt < maxRetries) {
        try {
            // Em cada tentativa, força recarregar a config local (se Orquestrador alterou .env)
            const currentURL = reloadEnvAndGetUrl();
            client.baseURL = currentURL;

            const response = await client.chat.completions.create({
                model: "mistral-7b-instruct", // Será ignorado se quantizado estático, mas obrigatório
                messages: messages,
            });
            return response.choices[0].message.content || "";
        } catch (error: any) {
            console.error(`Falha no Colab. Orquestrador em swap? Tentativa ${attempt + 1}/${maxRetries}`);
            if (error.status === 502 || error.message.includes('fetch failed')) {
                // Se 502 (Bad Gateway da Cloudflare), significa Worker caiu.
                // Aguarda 1 min para que o Orquestrador Python suba novo Worker.
                await new Promise(res => setTimeout(res, 60000));
                attempt++;
            } else {
                throw error; // Falhas de auth, ou request mal formado, não retentam infinitamente.
            }
        }
    }
    throw new Error("Timeout do Failover: O Colab Infinity não subiu um novo túnel a tempo.");
}
```

## 3. Considerações sobre o Protocolo Anti-Vibe

O Protocolo Anti-Vibe do Ouroboros é um "Quality Gate". Diferente da GPT-4o, um modelo 7B (Mistral) quantizado para 4-bits no Colab Infinity pode ser ocasionalmente inconsistente ou adicionar "textos de saudação" fora de um JSON requerido ("Certamente! Aqui está o seu plano... { ... }").

**Configuração recomendada:**
1. Altere o *System Prompt* de todas as chamadas do Ouroboros para forçar saída estrita.
2. Adicione no Anti-Vibe um `regex` limpo que extrai o primeiro e o último bracket `{` `}` caso o Colab Infinity devolva lixo no preâmbulo.
3. Se o Anti-Vibe barrar a requisição e solicitar correção, o retry também baterá na URL dinâmica. O Colab deve processar normal.
