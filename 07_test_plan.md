# Colab Infinity — Plano de Testes (Test Plan)

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Referências:** `02_srs.md`, `03_sad.md`, `04_api_spec.md`

---

## Índice

1. [Visão Geral e Objetivos](#1-visão-geral-e-objetivos)
2. [Escopo dos Testes](#2-escopo-dos-testes)
3. [Estratégia de Testes](#3-estratégia-de-testes)
4. [Ambiente de Testes](#4-ambiente-de-testes)
5. [Casos de Teste — Endpoint da API](#5-casos-de-teste--endpoint-da-api)
6. [Casos de Teste — Compatibilidade com Ouroboros Runtime](#6-casos-de-teste--compatibilidade-com-ouroboros-runtime)
7. [Casos de Teste — Compatibilidade com Hermes Agent](#7-casos-de-teste--compatibilidade-com-hermes-agent)
8. [Casos de Teste — Checkpoint (Salvar/Carregar)](#8-casos-de-teste--checkpoint-salvarcarregar)
9. [Casos de Teste — Troca de Conta (Simulação)](#9-casos-de-teste--troca-de-conta-simulação)
10. [Casos de Teste — Estresse e Carga](#10-casos-de-teste--estresse-e-carga)
11. [Critérios de Aceite](#11-critérios-de-aceite)
12. [Métricas e Relatórios](#12-métricas-e-relatórios)

---

## 1. Visão Geral e Objetivos

Este plano define a estratégia, casos de teste e critérios de aceite para validação do sistema
**Colab Infinity** antes de sua entrada em operação contínua como infraestrutura de LLM para o
ecossistema **Ouroboros Runtime / Hermes Agent / OpenClaw**.

### 1.1 Objetivos de Teste

| Objetivo                                                                                    | Prioridade |
|---------------------------------------------------------------------------------------------|------------|
| Validar compatibilidade total do endpoint `/v1/chat/completions` com a API OpenAI           | Alta       |
| Confirmar que o Ouroboros Runtime opera sem modificações usando o Colab Infinity como LLM   | Alta       |
| Garantir que o Hermes Agent completa conversas via o proxy local sem erros                  | Alta       |
| Verificar que checkpoints salvam e restauram estado sem perda de dados                      | Alta       |
| Confirmar que a troca automática de conta ocorre dentro do MTTR alvo (≤ 8 minutos)          | Alta       |
| Determinar os limites de throughput e latência do sistema sob carga                         | Média      |

### 1.2 Documentos de Referência

| Documento              | Propósito                                       |
|------------------------|-------------------------------------------------|
| `02_srs.md`            | Fonte dos requisitos funcionais (RF01–RF28)     |
| `03_sad.md`            | Arquitetura, fluxos e diagramas de sequência    |
| `04_api_spec.md`       | Contratos de request/response da API            |
| `05_setup_guide.md`    | Ambiente de referência para os testes           |
| `09_integration_guide.md` | Integração específica com o Ouroboros Runtime |

---

## 2. Escopo dos Testes

### 2.1 In Scope (O que será testado)

- API REST do servidor Colab via proxy local (`http://127.0.0.1:11434`)
- Proxy local do orquestrador (roteamento, retry, timeout, rate limiting)
- Mecanismo de checkpoint (serialização, escrita atômica, restauração)
- Lógica de troca de conta (detecção de falha, seleção da próxima conta, atualização do proxy)
- Integração Ouroboros Runtime ↔ Proxy local
- Integração Hermes Agent ↔ Proxy local
- Compatibilidade com o `openai` Python SDK e TypeScript SDK
- Comportamento sob carga (throughput, latência, estabilidade)
- Tratamento correto de erros e respostas HTTP

### 2.2 Out of Scope (O que NÃO será testado)

- Qualidade semântica das respostas geradas pelo LLM (avaliação subjetiva)
- Interface do usuário do Google Colab
- Infraestrutura interna do ngrok (reliability do relay server)
- Confiabilidade da rede do Google Drive
- Segurança das contas Google (responsabilidade do operador)
- Performance do modelo LLM em si (fora do controle do Colab Infinity)

---

## 3. Estratégia de Testes

### 3.1 Pirâmide de Testes

```
                    /\
                   /  \
                  / E2E \          ←  5% — Sistema completo real
                 /--------\
                / Integração\      ← 30% — Componentes integrados com mocks
               /--------------\
              /   Unitários    \   ← 65% — Funções e classes isoladas
             /------------------\
```

### 3.2 Níveis, Ferramentas e Diretórios

| Nível          | Ferramenta Principal       | Diretório              | Comando de Execução                          |
|----------------|----------------------------|------------------------|----------------------------------------------|
| Unitário       | `pytest`                   | `tests/unit/`          | `pytest tests/unit/ -v`                      |
| Integração     | `pytest` + `httpx`         | `tests/integration/`   | `pytest tests/integration/ -v`               |
| Sistema (E2E)  | `pytest` + `curl`          | `tests/e2e/`           | `pytest tests/e2e/ -v --slow --real-colab`   |
| Estresse       | `locust`                   | `tests/stress/`        | `locust -f tests/stress/locustfile.py`       |

### 3.3 Classificação de Prioridade dos Casos de Teste

| Prioridade | Critério                                                 |
|------------|----------------------------------------------------------|
| **P0**     | Bloqueador — impede uso básico do sistema pelo Ouroboros |
| **P1**     | Crítico — impacta funcionalidade principal               |
| **P2**     | Importante — reduz qualidade mas não bloqueia operação   |
| **P3**     | Baixo — edge case ou melhoria futura                     |

---

## 4. Ambiente de Testes

### 4.1 Ambiente de Desenvolvimento (Unitários + Integração com Mocks)

```
Máquina Local
├── Python 3.10+ com virtualenv
├── pytest 8.2.2
├── Mock do servidor Colab (FastAPI stub rodando em porta 8000)
│   └── Retorna respostas pré-definidas sem GPU ou LLM real
├── Mock do Google Drive (pasta local /tmp/ci_drive_mock/)
│   └── Simula leitura/escrita de checkpoint.json e pool_state.json
└── Mock do ngrok (URL fixa: http://localhost:8000)
```

### 4.2 Ambiente de Staging (Sistema Completo Real)

```
Máquina Local (Orquestrador)            Google Colab (Servidor Real)
├── orchestrator.py ativo               ├── GPU T4 alocada
├── Proxy local 127.0.0.1:11434         ├── Modelo: Mistral-7B-Instruct 4bit
├── Ouroboros Runtime configurado       ├── FastAPI + uvicorn :8000
├── Hermes Agent configurado            └── Túnel ngrok real
└── Drive real (conta de teste)
```

### 4.3 Variáveis de Ambiente para Testes

```bash
# Arquivo: tests/.env.test
CI_TEST_BASE_URL="http://127.0.0.1:11434/v1"
CI_TEST_MOCK_COLAB="true"          # false para testes E2E reais
CI_TEST_MOCK_DRIVE="true"
CI_TEST_MOCK_NGROK="true"
CI_TEST_TIMEOUT=30
CI_TEST_ACCOUNT_COUNT=3
CI_TEST_DRIVE_MOCK_PATH="/tmp/ci_drive_mock"
```

### 4.4 Setup do Ambiente de Testes

```bash
cd ~/colab-infinity
source .venv/bin/activate
pip install -r requirements-dev.txt
cp tests/.env.test.example tests/.env.test
# Edite tests/.env.test conforme necessário
pytest --co -q   # Lista todos os testes sem executar
```

---

## 5. Casos de Teste — Endpoint da API

### TC-API-001 — Health Check retorna 200 com campos obrigatórios

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-001                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF03 — Endpoint de health check                              |

**Pré-condições:**
- Proxy local rodando em `127.0.0.1:11434`
- Mock do servidor Colab retornando `{"status": "ok", "model": {"loaded": true}}`

**Passos:**
1. Executar `GET http://127.0.0.1:11434/health`
2. Capturar e validar resposta JSON

**Resultado Esperado:**
- Status HTTP: `200`
- `response.status` == `"ok"`
- `response.model.loaded` == `true`
- `response.model.id` é uma string não vazia
- `response.uptime_seconds` >= 0
- `response.session.id` é uma string não vazia
- `response.timestamp` é uma string ISO 8601 válida

**Resultado Obtido:** _(a preencher durante execução)_
**Status:** _(PASS / FAIL)_

---

### TC-API-002 — Chat completion retorna resposta válida no formato OpenAI

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-002                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF01, RF02 — Carregamento de modelo e endpoint de inferência |

**Pré-condições:**
- Servidor com modelo carregado (`model.loaded: true`)
- Proxy local ativo

**Passos:**
1. Enviar `POST /v1/chat/completions` com:
   ```json
   {
     "messages": [{"role": "user", "content": "Responda apenas: OK"}],
     "max_tokens": 10,
     "temperature": 0.1,
     "stream": false
   }
   ```
2. Capturar e validar resposta

**Resultado Esperado:**
- Status HTTP: `200`
- `response.choices` é array com 1 elemento
- `response.choices[0].message.role` == `"assistant"`
- `response.choices[0].message.content` é string não vazia
- `response.choices[0].finish_reason` em `["stop", "length"]`
- `response.usage.total_tokens` > 0
- `response.id` começa com `"chatcmpl-"`
- `response.x_colab_infinity.session_id` é string não vazia

**Status:** _(PASS / FAIL)_

---

### TC-API-003 — Requisição com campos inválidos retorna HTTP 400

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-003                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF10 — Validação de entrada da API                           |

**Passos (4 sub-casos):**
1. Enviar `POST /v1/chat/completions` com `"messages": []` (array vazio)
2. Enviar sem o campo `messages` (campo obrigatório ausente)
3. Enviar com `"temperature": 5.0` (fora do range `[0.0, 2.0]`)
4. Enviar com `"max_tokens": 99999` (acima do limite de 4096)
5. Enviar com `"n": 3` (não suportado — máximo 1)

**Resultado Esperado (para cada sub-caso):**
- Status HTTP: `400`
- `response.error.code` em `["INVALID_MESSAGES_FORMAT", "INVALID_REQUEST", "PARAM_OUT_OF_RANGE", "MAX_TOKENS_EXCEEDED", "UNSUPPORTED_N"]`
- `response.error.message` é string descritiva não vazia
- `response.error.type` == `"invalid_request"`

**Status:** _(PASS / FAIL)_

---

### TC-API-004 — Autenticação rejeitada quando API Key incorreta

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-004                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF11 — Autenticação opcional por API Key                     |

**Pré-condições:**
- `require_auth: true` em `colab_infinity_config.yaml`
- `api_key: "ci-sk-testkey123456789"` configurado

**Passos:**
1. Enviar `GET /health` sem header `Authorization`
2. Enviar `GET /health` com `Authorization: Bearer wrongkey`
3. Enviar `GET /health` com `Authorization: Bearer ci-sk-testkey123456789`

**Resultado Esperado:**
- Casos 1 e 2: Status HTTP `401`, `error.code` == `"UNAUTHORIZED"`
- Caso 3: Status HTTP `200` com body normal

**Status:** _(PASS / FAIL)_

---

### TC-API-005 — Streaming retorna chunks SSE válidos e completos

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-005                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF05 — Suporte a streaming (SSE)                             |

**Passos:**
1. Enviar `POST /v1/chat/completions` com `"stream": true` e `"max_tokens": 50`
2. Ler a resposta linha a linha até `data: [DONE]`
3. Parsear cada linha como JSON

**Resultado Esperado:**
- `Content-Type` contém `text/event-stream`
- Cada linha que começa com `data:` (exceto `[DONE]`) é JSON válido
- Cada chunk JSON tem `object` == `"chat.completion.chunk"`
- Primeiro chunk tem `delta.role` == `"assistant"`
- Último chunk antes de `[DONE]` tem `finish_reason` != `null`
- `data: [DONE]` é o último evento do stream
- Concatenação de todos `delta.content` forma texto coerente

**Status:** _(PASS / FAIL)_

---

### TC-API-006 — Rate limiting é aplicado corretamente

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-006                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RNF-P05 — Rate limiting                                      |

**Passos:**
1. Enviar 5 requisições `GET /health` em < 1 segundo entre cada
2. Registrar o status HTTP de cada resposta

**Resultado Esperado:**
- Primeiras 2 requisições: `200`
- A partir da 3ª dentro do mesmo segundo: `429`
- Header `Retry-After` presente nas respostas `429`
- Header `X-RateLimit-Remaining-Second` decrementado nas primeiras respostas
- Após aguardar `Retry-After` segundos, próxima requisição retorna `200`

**Status:** _(PASS / FAIL)_

---

### TC-API-007 — Endpoint /v1/status retorna estrutura completa

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-007                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF04 — Endpoint de status da sessão                          |

**Passos:**
1. Executar `GET /v1/status` com servidor ativo
2. Validar estrutura da resposta

**Resultado Esperado:**
- Status `200`
- Campos de nível superior presentes: `session`, `model`, `account`, `pool`, `checkpoint`, `tunnel`
- `session.uptime_seconds` >= 0
- `pool.total` >= 1
- `pool.available` >= 0
- `model.loaded` == `true`
- `checkpoint.last_saved_at` é ISO 8601 válido

**Status:** _(PASS / FAIL)_

---

### TC-API-008 — Compatibilidade com openai Python SDK

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-API-008                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF26–RF28 — Compatibilidade com agentes consumidores         |

**Pré-condições:**
- `pip install openai>=1.0.0`
- Proxy local ativo

**Passos:**

```python
# Exemplo ilustrativo do caso de teste TC-API-008
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:11434/v1",
    api_key="dummy"
)
response = client.chat.completions.create(
    model="mistralai/Mistral-7B-Instruct-v0.2",
    messages=[{"role": "user", "content": "ping"}],
    max_tokens=5,
)
# Verificar que nenhuma exceção é lançada
# Verificar response.choices[0].message.content
```

**Resultado Esperado:**
- Nenhuma exceção `ValidationError`, `AttributeError` ou `OpenAIError`
- `response.choices[0].message.content` é string não vazia
- `response.usage.total_tokens` > 0
- `response.model` é string não vazia

**Status:** _(PASS / FAIL)_

---

## 6. Casos de Teste — Compatibilidade com Ouroboros Runtime

### TC-OUR-001 — Ouroboros Runtime conecta ao proxy local sem modificações

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-OUR-001                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Sistema (E2E)                                                |
| **RF**         | RF26 — Compatibilidade com Ouroboros Runtime                 |

**Pré-condições:**
- Ouroboros Runtime clonado e instalado (`bun install`)
- Arquivo `.env` do Ouroboros configurado com:
  ```
  LLM_BASE_URL=http://127.0.0.1:11434/v1
  LLM_API_KEY=dummy
  ```
- Proxy local do Colab Infinity ativo (com mock do servidor Colab)

**Passos:**
1. Iniciar o Ouroboros Runtime Daemon: `bun run start`
2. Disparar uma Wave simples via TUI ou CLI do Ouroboros
3. Monitorar os logs do Daemon e do orquestrador

**Resultado Esperado:**
- Daemon inicia sem erros de conexão ao LLM
- Wave completa com sucesso (passa pelo menos pelo Gate 1 do Protocolo Anti-Vibe)
- Log do orquestrador registra pelo menos 1 requisição de `chat/completions` recebida
- Nenhum erro `CONNECTION_REFUSED` ou `ECONNREFUSED` nos logs do Daemon

**Status:** _(PASS / FAIL)_

---

### TC-OUR-002 — Ouroboros executa Wave com múltiplos agentes do Conselho

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-OUR-002                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Sistema (E2E)                                                |
| **RF**         | RF26 — Compatibilidade com múltiplos agentes simultâneos     |

**Pré-condições:**
- Mesmo ambiente que TC-OUR-001
- Wave configurada para envolver pelo menos 2 agentes (ex: Architect + Guardian)

**Passos:**
1. Disparar Wave que envolve Architect + Guardian
2. Aguardar conclusão da Wave
3. Verificar que ambos os agentes receberam respostas

**Resultado Esperado:**
- Ambos os agentes recebem respostas coerentes
- Nenhuma requisição é perdida ou fica sem resposta
- Não há condição de corrida no proxy local (respostas chegam ao agente correto)
- O SQLite do Ouroboros registra os resultados de ambos os agentes

**Status:** _(PASS / FAIL)_

---

### TC-OUR-003 — Ouroboros recebe 503 e retenta corretamente

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-OUR-003                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF19 — Troca automática de conta                             |

**Pré-condições:**
- Mock do proxy configurado para retornar `503 SESSION_SWITCHING` nas 2 primeiras chamadas
- Terceira chamada retorna `200` normalmente

**Passos:**
1. Configurar mock com comportamento descrito
2. Disparar uma chamada de agente via Ouroboros
3. Monitorar logs

**Resultado Esperado:**
- O Ouroboros Runtime retenta a chamada (conforme `LLM_MAX_RETRIES=3`)
- A terceira tentativa retorna `200` e a Wave continua
- Nenhuma exceção não tratada nos logs do Daemon
- Comportamento de retry registrado nos logs de nível `WARN` ou `INFO`

**Status:** _(PASS / FAIL)_

---

## 7. Casos de Teste — Compatibilidade com Hermes Agent

### TC-HRM-001 — Hermes Agent conecta ao proxy local e recebe resposta

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-HRM-001                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF27 — Compatibilidade com Hermes Agent                      |

**Pré-condições:**
- Hermes Agent configurado com `base_url: "http://127.0.0.1:11434/v1"`
- Proxy local ativo com mock do servidor Colab

**Passos:**
1. Executar `hermes-agent test-connection`
2. Capturar saída do comando e exit code

**Resultado Esperado:**
- Saída contém `✓ Health check: OK`
- Saída contém `✓ Chat completion: OK`
- Exit code `0`
- Nenhuma mensagem de erro ou exceção

**Status:** _(PASS / FAIL)_

---

### TC-HRM-002 — Hermes Agent completa conversa de 5 turnos sem erros

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-HRM-002                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Sistema (E2E)                                                |
| **RF**         | RF27 — Compatibilidade com Hermes Agent                      |

**Passos:**
1. Iniciar sessão de chat com o Hermes Agent (modo interativo ou scripted)
2. Enviar 5 mensagens sequenciais com respostas adequadas
3. Verificar que o histórico de contexto é mantido

**Resultado Esperado:**
- Hermes Agent processa os 5 turnos sem erros
- As respostas demonstram continuidade de contexto (o agente "lembra" mensagens anteriores)
- Nenhum timeout ou `ConnectionError` durante a sessão
- Todos os tokens de resposta são decodificados corretamente (sem caracteres garbled)

**Status:** _(PASS / FAIL)_

---

### TC-HRM-003 — Hermes Agent usa streaming e exibe tokens em tempo real

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-HRM-003                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF05 — Suporte a streaming                                   |

**Passos:**
1. Configurar Hermes Agent com `stream: true`
2. Enviar um prompt de ~200 tokens de resposta esperada
3. Medir o tempo até o primeiro token aparecer vs. o tempo total

**Resultado Esperado:**
- Primeiro token aparece em < 3 segundos (Time to First Token)
- Tokens aparecem progressivamente na tela (não tudo de uma vez no final)
- Resposta final é idêntica ao modo não-streaming para o mesmo seed/temperatura

**Status:** _(PASS / FAIL)_

---

## 8. Casos de Teste — Checkpoint (Salvar/Carregar)

### TC-CKP-001 — Checkpoint salvo automaticamente no intervalo configurado

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-001                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF12 — Checkpoint automático periódico                       |

**Pré-condições:**
- `checkpoint.interval_seconds: 60` (reduzido para teste)
- Mock do Drive habilitado com path local `/tmp/ci_drive_mock/`

**Passos:**
1. Iniciar orquestrador com `CHECKPOINT_INTERVAL_SECONDS=60`
2. Aguardar 65 segundos
3. Listar arquivos em `/tmp/ci_drive_mock/colab_infinity/checkpoints/`

**Resultado Esperado:**
- Pelo menos 1 arquivo `ci_ckpt_YYYYMMDD_HHMMSS.json` criado
- O arquivo é JSON válido (parseável sem exceção)
- Campos obrigatórios presentes: `schema_version`, `saved_at`, `save_reason`, `session`, `pool`
- `save_reason` == `"periodic"`
- `session.requests_served` >= 0

**Status:** _(PASS / FAIL)_

---

### TC-CKP-002 — Checkpoint forçado via POST /v1/checkpoint retorna confirmação

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-002                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF13 — Checkpoint forçado via API                            |

**Passos:**
1. Executar:
   ```bash
   curl -s -X POST http://127.0.0.1:11434/v1/checkpoint \
     -H "Content-Type: application/json" \
     -d '{"reason": "tc_ckp_002_test"}'
   ```
2. Verificar resposta e arquivo criado no Drive mock

**Resultado Esperado:**
- Status HTTP: `200`
- `response.status` == `"saved"`
- `response.checkpoint_id` contém `"tc_ckp_002_test"` ou string derivada
- Arquivo correspondente existe em `/tmp/ci_drive_mock/colab_infinity/checkpoints/`
- `duration_ms` > 0

**Status:** _(PASS / FAIL)_

---

### TC-CKP-003 — Restauração de estado a partir de checkpoint válido

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-003                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF14 — Restauração a partir de checkpoint                    |

**Pré-condições:**
- Arquivo de fixture `tests/fixtures/checkpoint_valid.json`:
  ```json
  {
    "schema_version": "1.1",
    "saved_at": "2025-07-14T03:00:00Z",
    "save_reason": "test_fixture",
    "session": {
      "id": "ci_sess_test_restore",
      "account_index": 2,
      "ngrok_url": "https://test-fixture.ngrok-free.app",
      "model_id": "mistralai/Mistral-7B-Instruct-v0.2",
      "requests_served": 150,
      "tokens_generated": 45000
    },
    "pool": {
      "total_accounts": 4,
      "accounts": [
        {"index": 0, "status": "exhausted"},
        {"index": 1, "status": "exhausted"},
        {"index": 2, "status": "active"},
        {"index": 3, "status": "available"}
      ],
      "next_account_index": 3
    },
    "metrics": {
      "total_requests_all_sessions": 300,
      "total_account_switches": 2
    }
  }
  ```

**Passos:**
1. Iniciar orquestrador com flag `--restore-from tests/fixtures/checkpoint_valid.json`
2. Verificar estado inicial após restauração

**Resultado Esperado:**
- `active_account_index` == `2`
- Pool reflete: conta 0 e 1 como `exhausted`, conta 3 como `available`
- Contadores acumulados `total_requests_all_sessions` iniciam em `300`
- `total_account_switches` inicia em `2`
- Log registra evento `checkpoint_restored` com o path do arquivo

**Status:** _(PASS / FAIL)_

---

### TC-CKP-004 — Checkpoint corrompido aciona recuperação de fallback

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-004                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF14 — Tolerância a checkpoint corrompido                    |

**Pré-condições:**
- Arquivo `tests/fixtures/checkpoint_corrupt.json` com conteúdo: `{"broken": true, "schema_version":` (JSON truncado/inválido)

**Passos:**
1. Tentar carregar o checkpoint corrompido via `CheckpointManager.load("tests/fixtures/checkpoint_corrupt.json")`
2. Verificar o comportamento do orquestrador

**Resultado Esperado:**
- `CheckpointCorruptError` é lançado (não `JSONDecodeError` não tratado)
- Log de nível `ERROR` registra o erro com o path do arquivo
- Orquestrador inicia em estado limpo (sem travar ou crashar)
- Nenhuma exceção não tratada propaga para o nível superior

**Status:** _(PASS / FAIL)_

---

### TC-CKP-005 — Escrita atômica impede checkpoint parcialmente escrito

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-005                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF16 — Escrita atômica de checkpoint                         |

**Passos:**
1. Usar mock que lança exceção durante a escrita do arquivo temporário `.tmp`
2. Verificar o estado dos arquivos após a exceção

**Resultado Esperado:**
- Arquivo de checkpoint anterior permanece intacto e válido
- Nenhum arquivo `.tmp` persiste no diretório após a falha
- O mecanismo usa `rename()` atômico (temp → final)
- Log registra `checkpoint_write_failed` com o erro específico

**Status:** _(PASS / FAIL)_

---

### TC-CKP-006 — Limpeza automática mantém apenas os N checkpoints mais recentes

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-CKP-006                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF15 — Limpeza de checkpoints antigos                        |

**Pré-condições:**
- `checkpoint.max_files: 3` (reduzido para teste)
- 3 arquivos de checkpoint já existentes no Drive mock

**Passos:**
1. Forçar salvamento de um 4º checkpoint via `POST /v1/checkpoint`
2. Listar arquivos no diretório de checkpoints

**Resultado Esperado:**
- Exatamente 3 arquivos de checkpoint presentes (o mais antigo foi removido)
- O arquivo mais antigo (pelo timestamp no nome) foi deletado
- Os 3 mais recentes estão intactos e são JSON válidos

**Status:** _(PASS / FAIL)_

---

## 9. Casos de Teste — Troca de Conta (Simulação)

### TC-SWT-001 — Troca automática acionada por falhas consecutivas de health check

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-SWT-001                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF18, RF19 — Monitoramento e troca automática de conta       |

**Pré-condições:**
- Mock do servidor Colab configurado para retornar `503` nas próximas 3 chamadas de health check
- Pool com 3 contas: conta 0 ativa, contas 1 e 2 disponíveis
- `health_check_fail_threshold: 3` no config

**Passos:**
1. Iniciar orquestrador com mocks
2. Aguardar 3 health checks falharem (3 × 30s = ~90s)
3. Monitorar logs e estado do proxy

**Resultado Esperado:**
- Após 3 falhas consecutivas, estado transita `ACTIVE` → `FAILING` → `SWITCHING`
- Log registra: `account_switch_triggered reason=health_check_failures count=3`
- Conta 0 é marcada como `exhausted` em `pool_state.json`
- Conta 1 é selecionada como nova conta ativa
- Estado retorna a `ACTIVE` após nova sessão estar pronta
- Proxy local atualiza URL de destino para a nova sessão
- Tempo total de troca (FAILING → ACTIVE) ≤ 8 minutos

**Status:** _(PASS / FAIL)_

---

### TC-SWT-002 — Troca NÃO ocorre com falhas abaixo do threshold

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-SWT-002                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Unitário                                                     |
| **RF**         | RF18 — Monitoramento de saúde contínuo                       |

**Pré-condições:**
- Mock configurado para retornar `503` em 2 chamadas consecutivas, depois `200`
- `health_check_fail_threshold: 3`

**Passos:**
1. Simular 2 falhas seguidas de 1 sucesso
2. Verificar estado do orquestrador

**Resultado Esperado:**
- Estado permanece `ACTIVE` (sem troca iniciada)
- Contador de falhas consecutivas é zerado após o sucesso
- Log registra `health_check_recovery` após o sucesso
- `pool_state.json` não é modificado

**Status:** _(PASS / FAIL)_

---

### TC-SWT-003 — Pool exausto coloca sistema em espera automática

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-SWT-003                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF20, RF21 — Seleção de conta e gerenciamento de cooldown    |

**Pré-condições:**
- Pool configurado com apenas 1 conta disponível
- Essa conta falha nos health checks

**Passos:**
1. Simular falha da única conta disponível
2. Verificar comportamento do sistema

**Resultado Esperado:**
- Orquestrador entra no estado `HALTED` (ou equivalente de pool exausto)
- Log de nível `CRITICAL` registra: `pool_exhausted available_accounts=0`
- Proxy local retorna `503 POOL_EXHAUSTED` para todas as novas requisições
- Se `notifications.enabled: true`, webhook é disparado com `event: "pool_exhausted"`
- Sistema retoma automaticamente quando uma conta passa pelo cooldown de 24h

**Status:** _(PASS / FAIL)_

---

### TC-SWT-004 — Troca manual via CLI funciona corretamente

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-SWT-004                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Sistema                                                      |
| **RF**         | RF23 — Troca manual de conta via CLI                         |

**Passos:**
1. Com o sistema ativo na conta 0, executar:
   ```bash
   python3 -m colab_infinity.cli switch-account --target-index 1
   ```
2. Monitorar logs e endpoint de status

**Resultado Esperado:**
- Orquestrador inicia processo de troca imediatamente
- Estado transita: `ACTIVE` → `SWITCHING` → `ACTIVE`
- `GET /v1/status` reflete `account.index: 1` após a troca
- Requisições durante a troca retornam `503 SESSION_SWITCHING`
- Log registra `manual_switch_triggered target_index=1`

**Status:** _(PASS / FAIL)_

---

### TC-SWT-005 — Checkpoint salvo antes da troca preserva contadores

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-SWT-005                                                   |
| **Prioridade** | P0                                                           |
| **Tipo**       | Integração                                                   |
| **RF**         | RF12, RF19 — Checkpoint pré-troca e tolerância a falhas      |

**Passos:**
1. Processar 50 requisições de chat, anotar `requests_served` e `tokens_generated`
2. Forçar uma troca de conta
3. Após nova sessão ativa, verificar `GET /v1/status` e último checkpoint

**Resultado Esperado:**
- Checkpoint salvo **antes** da troca tem `save_reason: "pre_switch"`
- Checkpoint contém `requests_served` = 50 (ou mais se houve mais)
- `pool_state.json` no Drive reflete o novo `active_account_index`
- Contadores acumulados não são zerados após a troca
- `GET /v1/status` retorna `pool.accounts[0].status: "exhausted"`

**Status:** _(PASS / FAIL)_

---

## 10. Casos de Teste — Estresse e Carga

### TC-STR-001 — Throughput máximo sustentável por 10 minutos

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-STR-001                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Estresse                                                     |
| **RNF**        | RNF-P05 — Throughput ≥ 2 req/min                            |

**Ferramenta:** `locust`

**Configuração do teste (em `tests/stress/locustfile.py`):**

```python
# Exemplo ilustrativo da configuração do Locust para TC-STR-001
# from locust import HttpUser, task, between
#
# class ColabInfinityUser(HttpUser):
#     host = "http://127.0.0.1:11434"
#     wait_time = between(5, 15)  # segundos entre requisições
#
#     @task
#     def chat_completion(self):
#         payload = {
#             "messages": [{"role": "user", "content": "Olá, qual é a capital da França?"}],
#             "max_tokens": 50,
#             "temperature": 0.7
#         }
#         with self.client.post("/v1/chat/completions",
#                               json=payload,
#                               timeout=120,
#                               catch_response=True) as resp:
#             if resp.status_code == 200:
#                 resp.success()
#             elif resp.status_code in (503, 429):
#                 resp.failure(f"Expected error: {resp.status_code}")
#             else:
#                 resp.failure(f"Unexpected: {resp.status_code}")
```

**Passos:**
1. Executar ramp-up de 1 → 3 usuários concorrentes por 10 minutos
2. Coletar métricas de latência, taxa de erro e throughput

**Resultado Esperado:**
- Taxa de erro < 5% (excluindo `503` durante trocas de conta)
- P50 de latência < 8 segundos para `max_tokens=50`
- P95 de latência < 20 segundos para `max_tokens=50`
- Sistema não trava, não reinicia e não perde requisições inesperadamente
- VRAM não excede 90% durante o teste

**Status:** _(PASS / FAIL)_

---

### TC-STR-002 — Estabilidade após 2 horas de operação contínua (Soak Test)

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-STR-002                                                   |
| **Prioridade** | P1                                                           |
| **Tipo**       | Estresse / Soak                                              |
| **RNF**        | RNF-R01 — Uptime ≥ 95%                                       |

**Passos:**
1. Iniciar sistema completo (orquestrador + Colab real + Hermes Agent)
2. Executar 1 requisição de chat a cada 60 segundos por 2 horas
3. Registrar: uptime, erros, latência média, checkpoints salvos

**Resultado Esperado:**
- Uptime ≥ 95% (máximo 6 minutos de indisponibilidade em 2 horas)
- Sem vazamento de memória: RSS do `orchestrator.py` não cresce > 50 MB ao longo do teste
- Pelo menos `120min / 5min = 24` checkpoints salvos com sucesso
- VRAM do Colab permanece estável (sem crescimento progressivo de fragmentação)
- Log do orquestrador não apresenta exceções não tratadas

**Status:** _(PASS / FAIL)_

---

### TC-STR-003 — Comportamento sob requisições com contexto longo (payload máximo)

| Atributo       | Valor                                                        |
|----------------|--------------------------------------------------------------|
| **ID**         | TC-STR-003                                                   |
| **Prioridade** | P2                                                           |
| **Tipo**       | Estresse                                                     |
| **RNF**        | RNF-P03 — Latência P99                                       |

**Passos:**
1. Enviar 5 requisições sequenciais com:
   - `max_tokens: 4096`
   - Contexto de entrada com ~2000 tokens (prompt longo)
2. Medir latência e uso de VRAM para cada requisição

**Resultado Esperado:**
- Nenhuma requisição retorna `500 MODEL_INFERENCE_ERROR` por OOM
- VRAM não excede 95% da capacidade total
- Latência P95 < 120 segundos para contexto máximo
- Modelo permanece carregado na VRAM entre as 5 requisições (sem reload)

**Status:** _(PASS / FAIL)_

---

## 11. Critérios de Aceite

### 11.1 Critérios Mandatórios para Entrada em Produção (Go/No-Go)

Todos os itens abaixo devem ser satisfeitos **antes** de iniciar a operação contínua:

| Critério                                                   | Condição                                                              | Obrigatório |
|------------------------------------------------------------|-----------------------------------------------------------------------|-------------|
| Todos os testes P0 passando                                | 0 falhas em: TC-API-001, TC-API-002, TC-API-008, TC-OUR-001, TC-CKP-001, TC-CKP-003, TC-SWT-001, TC-SWT-005 | **Sim** |
| Taxa de aprovação de testes P1                             | ≥ 90% dos testes P1 com status PASS                                   | **Sim**     |
| Nenhuma regressão na suite de API                          | TC-API-001 a TC-API-008: todos PASS                                   | **Sim**     |
| Checkpoint salva e restaura sem perda de estado            | TC-CKP-001, TC-CKP-002, TC-CKP-003: todos PASS                       | **Sim**     |
| Troca de conta dentro do MTTR alvo                         | TC-SWT-001: tempo de troca ≤ 8 minutos                                | **Sim**     |
| Compatibilidade com OpenAI SDK confirmada                  | TC-API-008: PASS                                                      | **Sim**     |
| Compatibilidade com Ouroboros Runtime confirmada           | TC-OUR-001: PASS                                                      | **Sim**     |
| Soak test de 2h com uptime ≥ 95%                           | TC-STR-002: PASS                                                      | **Sim**     |
| Taxa de aprovação de testes P2                             | ≥ 70% dos testes P2 com status PASS                                   | Não         |

### 11.2 Critérios de Bloqueio Absoluto (impede release em qualquer hipótese)

| Critério de Bloqueio                                                                           |
|-----------------------------------------------------------------------------------------------|
| Falha em qualquer teste P0                                                                     |
| `checkpoint.json` corrompido ou com perda de dados (TC-CKP-003 ou TC-CKP-005 falhando)        |
| Troca de conta com duração > 15 minutos (dobro do MTTR alvo)                                  |
| Vazamento de memória detectado no soak test (TC-STR-002 — crescimento RSS > 50 MB)            |
| Incompatibilidade com OpenAI SDK (TC-API-008 falhando)                                        |
| Ouroboros Runtime não consegue completar uma Wave (TC-OUR-001 falhando)                       |

---

## 12. Métricas e Relatórios

### 12.1 Métricas Coletadas Durante os Testes

| Métrica                              | Ferramenta                    | Limiar Aceitável              |
|--------------------------------------|-------------------------------|-------------------------------|
| Latência P50 (50 tokens)             | `pytest-benchmark` / locust   | < 5s                          |
| Latência P95 (50 tokens)             | `pytest-benchmark` / locust   | < 15s                         |
| Taxa de erro em carga                | locust                        | < 5%                          |
| Tempo médio de troca de conta (MTTR) | Logs do orquestrador          | < 8 minutos                   |
| Uptime em soak test 2h               | Logs do orquestrador          | ≥ 95%                         |
| Tamanho médio do checkpoint          | `stat` do arquivo             | < 1 MB                        |
| Cobertura de testes unitários        | `pytest-cov`                  | ≥ 80%                         |
| Crescimento de RSS em soak test      | `psutil`                      | < 50 MB em 2h                 |

### 12.2 Execução da Suite Completa

```bash
# Testes unitários (rápido, < 2 minutos)
pytest tests/unit/ -v --tb=short

# Testes de integração com mocks (médio, < 10 minutos)
pytest tests/integration/ -v --tb=short

# Gerar relatório de cobertura
pytest tests/unit/ tests/integration/ \
  --cov=colab_infinity \
  --cov-report=html:reports/coverage \
  --cov-report=term-missing

# Testes E2E com sistema real (lento, exige Colab ativo, ~30 minutos)
pytest tests/e2e/ -v --slow --real-colab --tb=short

# Soak test (muito lento, ~2 horas)
locust -f tests/stress/locustfile.py \
  --headless \
  --users 2 \
  --spawn-rate 1 \
  --run-time 2h \
  --html reports/soak_test_$(date +%Y%m%d).html
```

### 12.3 Formato do Relatório de Teste

Após cada ciclo de testes, gerar relatório com:

```bash
pytest tests/ \
  --cov=colab_infinity \
  --cov-report=html:reports/coverage \
  --html=reports/test_report_$(date +%Y%m%d).html \
  --tb=short \
  -v
```

O relatório deve ser arquivado em `reports/` e associado ao número de versão/commit.

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*