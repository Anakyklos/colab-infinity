# Guia de Integração

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `04_api_spec.md`

Este documento orienta os desenvolvedores a integrarem seus sistemas (Agentes Autônomos, como o Ouroboros Runtime) de forma passiva à infraestrutura do Colab Infinity, usufruindo da API OpenAI compatível exposta no Proxy local via Ngrok.

---

## 1. Abordagem "Drop-in Replacement"

O Colab Infinity opera através da porta TCP `11434` (completamente similar à abordagem feita pelo `Ollama`). No entanto, em vez de carregar o modelo localmente sobrecarregando a máquina do desenvolvedor, o proxy local despacha as rotas pelo túnel Ngrok para a VM da Google.

A única alteração requerida nos agentes clientes é a **substituição da Base URL e da Key**.

### 1.1 Configuração via `.env` (Padrão de Mercado)

Para repositórios que seguem o padrão LangChain / OpenAI, ajuste o arquivo de variáveis de ambiente:

```dotenv
# Desative o roteamento para a OpenAI verdadeira
OPENAI_API_BASE=http://127.0.0.1:11434/v1
OPENAI_API_KEY=sk-colab-infinity-dummy-key
# Especificar o modelo conforme configurado no Jupyter Notebook
OPENAI_MODEL=mistralai/Mistral-7B-Instruct-v0.2
```

---

## 2. Exemplos de Integração em Código

### 2.1 Python (OpenAI SDK)

```python
from openai import OpenAI

# A chave 'api_key' é descartada pelo proxy, mas a biblioteca requer seu preenchimento.
client = OpenAI(
    base_url="http://127.0.0.1:11434/v1",
    api_key="infinity-123"
)

response = client.chat.completions.create(
    model="mistralai/Mistral-7B-Instruct-v0.2",
    messages=[
        {"role": "system", "content": "Você é o núcleo cognitivo de um Agente."},
        {"role": "user", "content": "Plano de execução aprovado. Avance."}
    ]
)

print(response.choices[0].message.content)
```

### 2.2 TypeScript / Bun (Ouroboros Runtime / LangChain JS)

A integração com o ecossistema Typescript do `Ouroboros Runtime` é fluída com o pacote `@langchain/openai`.

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  configuration: {
    baseURL: "http://127.0.0.1:11434/v1",
  },
  openAIApiKey: "dummy-key-not-used",
  modelName: "mistralai/Mistral-7B-Instruct-v0.2",
  temperature: 0.7,
  maxTokens: 1024,
  maxRetries: 3,     // O proxy cuidará de retries profundos, aqui é uma segurança local.
  timeout: 120000    // Timeout elevado é CRUCIAL (Ngrok + Inferência pode tomar tempo).
});

async function run() {
  const res = await model.invoke([
    { role: "user", content: "Analise o arquivo AGENTS.md no diretório atual." }
  ]);
  console.log(res.content);
}
run();
```

---

## 3. Lógica de Tolerância a Falhas no Cliente

Embora o proxy do **Colab Infinity** oculte a maior parte das complexidades de Failover e recarregue silenciosamente as máquinas Colab nos bastidores, recomenda-se que os agentes clientes implementem as seguintes premissas na camada de rede para operarem perfeitamente neste túnel Ngrok:

1. **Max Retries (Tratamento de 503):** Se uma requisição chegar bem no meio de uma rotação pesada de VM e o proxy decidir que não aguardará (devolvendo *503 Service Unavailable*), o Agente deve implementar bibliotecas de Retry Exponencial ou acatar o cabeçalho `Retry-After: 60`, aguardando a nova máquina no Colab acordar, religar o modelo de 4-bits na T4 e repassar para um novo link do Ngrok.
2. **Timeouts Elevados:** Modelos 7B inferindo grandes contextos de documentação de código demoram para retornar o primeiro token. O timeout do Axios/Fetch/Requests do agente **nunca deve ser menor que 60 segundos** ao usar o Infinity.
