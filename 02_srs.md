# Colab Infinity — Especificação de Requisitos de Software (SRS)

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Referências:** `01_project_charter.md`, `03_sad.md`

---

## Índice

1. [Introdução e Propósito](#1-introdução-e-propósito)
2. [Requisitos Funcionais](#2-requisitos-funcionais)
3. [Requisitos Não Funcionais](#3-requisitos-não-funcionais)
4. [Requisitos de Hardware](#4-requisitos-de-hardware)
5. [Requisitos de Software](#5-requisitos-de-software)
6. [Interfaces Externas](#6-interfaces-externas)
7. [Restrições e Dependências Externas](#7-restrições-e-dependências-externas)
8. [Modelos de Dados](#8-modelos-de-dados)
9. [Casos de Uso Principais](#9-casos-de-uso-principais)

---

## 1. Introdução e Propósito

### 1.1 Propósito do Documento

Este documento especifica todos os requisitos funcionais e não funcionais do sistema **Colab Infinity**,
um módulo de infraestrutura de LLM projetado para fornecer capacidade de inferência gratuita e contínua
ao ecossistema de agentes autônomos Ouroboros Runtime, Hermes Agent, OpenClaw e quaisquer outros
clientes compatíveis com a OpenAI Chat Completions API.

### 1.2 Escopo do Sistema

O Colab Infinity opera em duas camadas:

- **Camada de Computação (Google Colab):** Notebook que carrega o modelo LLM, expõe a API de inferência
  e gerencia o ciclo de vida da sessão Colab gratuita.
- **Camada de Orquestração (Local):** Script local que monitora a saúde do servidor, gerencia o pool
  de contas Google e executa a rotação automática quando necessário.

### 1.3 Definições e Abreviações

| Termo               | Definição                                                                              |
|---------------------|----------------------------------------------------------------------------------------|
| **Conta Armazém**   | Conta Google dedicada ao armazenamento de estado persistente no Google Drive           |
| **Pool de Contas**  | Conjunto de contas Google rotativas usadas para manter sessões Colab ativas            |
| **Checkpoint**      | Arquivo JSON salvo no Drive com o estado atual do sistema para recuperação de falhas   |
| **Orquestrador**    | Script local (`orchestrator.py`) que gerencia o ciclo de vida das sessões Colab        |
| **Proxy Local**     | Componente do orquestrador que roteia requisições dos agentes para o Colab ativo       |
| **Quota Colab**     | Limite de tempo de GPU gratuita por conta Google (aproximadamente 12h por sessão)      |
| **MTTR**            | Mean Time To Recovery — tempo médio de recuperação após uma falha                      |
| **MTBF**            | Mean Time Between Failures — tempo médio entre falhas                                  |
| **SSE**             | Server-Sent Events — protocolo para streaming de respostas                             |
| **Ouroboros**       | Sistema de orquestração multiagente (https://github.com/RenyEnnos/ouroboros-runtime)   |
| **Anti-Vibe**       | Protocolo de qualidade do Ouroboros com gates de especificação, validação e revisão    |

---

## 2. Requisitos Funcionais

A tabela abaixo apresenta todos os requisitos funcionais. Cada requisito possui um identificador único,
prioridade e critério de aceite verificável.

### 2.1 Grupo: Servidor de Inferência (Notebook Colab)

| ID    | Nome                              | Prioridade | Descrição                                                                                                   | Critério de Aceite                                                                                         |
|-------|-----------------------------------|------------|-------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| RF01  | Carregamento de modelo LLM        | Alta       | O notebook deve carregar um modelo LLM de linguagem causal (Mistral-7B, Llama-3-8B ou similar) na GPU T4 do Colab usando quantização de 4 bits via `bitsandbytes`. | Modelo responde a um prompt de teste em menos de 30s após o carregamento, com uso de VRAM < 10 GB.        |
| RF02  | Endpoint de inferência OpenAI     | Alta       | O servidor deve expor o endpoint `POST /v1/chat/completions` com formato de requisição e resposta 100% compatível com a OpenAI Chat Completions API v1. | O `openai` Python SDK (≥ 1.0.0) conectado ao servidor consegue completar um chat sem exceções de parsing. |
| RF03  | Endpoint de health check          | Alta       | O servidor deve expor `GET /health` retornando status, modelo carregado, uso de VRAM, uptime e session ID. | Resposta JSON com `"status": "ok"` e todos os campos obrigatórios em menos de 500ms.                      |
| RF04  | Endpoint de status da sessão      | Média      | O servidor deve expor `GET /v1/status` com informações detalhadas da sessão, conta ativa e estado do pool. | Resposta JSON completa com campos `session`, `model`, `account`, `pool` e `ngrok`.                        |
| RF05  | Suporte a streaming (SSE)         | Alta       | O endpoint `POST /v1/chat/completions` deve suportar o parâmetro `"stream": true`, retornando a resposta como Server-Sent Events em tempo real. | Stream retorna chunks `data: {...}` incrementais e finaliza com `data: [DONE]`.                            |
| RF06  | Túnel público via ngrok           | Alta       | O notebook deve criar um túnel HTTP público via ngrok assim que o servidor FastAPI estiver pronto, tornando a API acessível externamente. | URL pública `https://<hash>.ngrok-free.app` acessível de fora da rede local com latência adicional < 200ms. |
| RF07  | Publicação da URL no Drive        | Alta       | Imediatamente após criar o túnel ngrok, o notebook deve salvar a URL pública em `hermes_infinito/pool_state/ngrok_url.json` na conta armazém (Drive montado). | Arquivo `ngrok_url.json` presente e atualizado no Drive dentro de 10s após criação do túnel.               |
| RF08  | Keepalive da sessão Colab         | Alta       | O notebook deve executar um mecanismo de keepalive (clique periódico via JavaScript ou loop de polling) para evitar a desconexão automática por inatividade do Colab. | Sessão permanece ativa por pelo menos 2h sem interação manual do usuário.                                  |
| RF09  | Monitoramento de quota de GPU     | Alta       | O notebook deve monitorar continuamente o tempo de GPU restante e disparar o processo de checkpoint + sinalização de troca quando restar menos de `QUOTA_WARNING_MINUTES` minutos (padrão: 30min). | Evento `quota_warning` registrado nos logs e `pool_state.json` atualizado com flag `switch_requested: true` quando threshold é atingido. |
| RF10  | Validação de entrada da API       | Média      | O servidor deve validar todos os campos da requisição e retornar `400 Bad Request` com mensagem descritiva para entradas inválidas (array `messages` vazio, `temperature` fora do range, `max_tokens` acima do limite). | Requisições inválidas retornam HTTP 400 com `error.code` e `error.message` preenchidos corretamente.      |
| RF11  | Autenticação opcional por API Key | Média      | O servidor deve suportar autenticação via header `Authorization: Bearer <token>` quando `require_auth: true` estiver configurado. Sem autenticação, todas as requisições são aceitas. | Com `require_auth: true`, requisições sem token válido retornam HTTP 401. Com `require_auth: false`, todas as requisições são aceitas. |

### 2.2 Grupo: Checkpoint e Persistência

| ID    | Nome                              | Prioridade | Descrição                                                                                                   | Critério de Aceite                                                                                         |
|-------|-----------------------------------|------------|-------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| RF12  | Checkpoint automático periódico   | Alta       | O orquestrador deve salvar o estado completo do sistema em `checkpoint_YYYYMMDD_HHMMSS.json` no Drive a cada `CHECKPOINT_INTERVAL_SECONDS` (padrão: 300s). | Arquivo de checkpoint criado e validado como JSON íntegro a cada 5 minutos de operação.                   |
| RF13  | Checkpoint forçado via API        | Média      | O endpoint `POST /v1/checkpoint` deve permitir forçar o salvamento imediato do estado, retornando confirmação com ID e caminho do arquivo criado. | Resposta HTTP 200 com `status: "saved"` e arquivo confirmado no Drive dentro de 5s da requisição.         |
| RF14  | Restauração a partir de checkpoint| Alta       | O orquestrador deve detectar um checkpoint válido no Drive ao iniciar e restaurar o estado (conta ativa, pool status, contadores) antes de iniciar uma nova sessão. | Após restart com checkpoint existente, `active_account_index` e `pool_state` refletem os valores salvos. |
| RF15  | Limpeza de checkpoints antigos    | Baixa      | O orquestrador deve manter apenas os `CHECKPOINT_MAX_FILES` (padrão: 10) checkpoints mais recentes, excluindo os mais antigos automaticamente. | Após 11 checkpoints criados, o mais antigo é removido, mantendo exatamente 10 arquivos.                   |
| RF16  | Escrita atômica de checkpoint     | Alta       | O checkpoint deve ser escrito em arquivo temporário (`checkpoint.tmp`) e renomeado atomicamente para o nome final, evitando leitura de arquivo parcialmente escrito. | Interrupção durante a escrita não corrompe o checkpoint anterior; o arquivo `.tmp` não persiste após falha. |

### 2.3 Grupo: Orquestrador e Pool de Contas

| ID    | Nome                              | Prioridade | Descrição                                                                                                   | Critério de Aceite                                                                                         |
|-------|-----------------------------------|------------|-------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| RF17  | Proxy local transparente          | Alta       | O orquestrador deve manter um proxy HTTP local em `127.0.0.1:8081` que roteia requisições para a URL ngrok ativa, atualizando o destino de forma transparente após cada troca de conta. | Requisições ao proxy local chegam ao servidor Colab correto sem alteração do endpoint pelos agentes consumidores. |
| RF18  | Monitoramento de saúde contínuo   | Alta       | O orquestrador deve realizar polling de saúde no endpoint `/health` a cada `HEALTH_CHECK_INTERVAL_SECONDS` (padrão: 30s) e registrar falhas consecutivas. | Log estruturado registra resultado de cada health check; falhas são contabilizadas por sessão.             |
| RF19  | Troca automática de conta         | Alta       | Após `HEALTH_CHECK_FAIL_THRESHOLD` (padrão: 3) falhas consecutivas de health check, o orquestrador deve iniciar o processo de troca de conta automaticamente. | Estado transita ACTIVE → SWITCHING → ACTIVE; nova conta operacional em até 8 minutos.                     |
| RF20  | Seleção da próxima conta          | Alta       | O orquestrador deve selecionar a próxima conta disponível do pool em ordem de índice, ignorando contas com status `exhausted`, `banned` ou em período de cooldown. | Conta com status diferente de `available` é pulada; conta `available` mais próxima é selecionada.         |
| RF21  | Gerenciamento de cooldown         | Média      | Contas marcadas como `exhausted` devem entrar em período de cooldown (`COOLDOWN_HOURS`, padrão: 24h) antes de ficarem disponíveis novamente. | Conta com timestamp `exhausted_at` não é selecionada até que `now() - exhausted_at >= COOLDOWN_HOURS`.    |
| RF22  | Configuração do pool via arquivo  | Alta       | O pool de contas deve ser configurável via `~/.colab_infinity/accounts.json`, suportando de 3 a 10 contas com campos: `index`, `email`, `ngrok_token`, `role` e `notes`. | Adição de nova conta ao `accounts.json` seguida de `--reload-pool` reflete a mudança sem reiniciar o orquestrador. |
| RF23  | Troca manual de conta via CLI     | Média      | O orquestrador deve suportar o comando `python -m colab_infinity.cli switch-account --target-index N` para forçar troca manual de conta. | Troca iniciada imediatamente; nova conta ativa em até 8 minutos; log registra `manual_switch_triggered`.  |
| RF24  | Notificação de eventos críticos   | Baixa      | Quando `notifications.enabled: true`, o orquestrador deve enviar um payload HTTP POST para `webhook_url` nos eventos: `account_switched`, `pool_exhausted`, `checkpoint_saved`, `server_error`. | Webhook recebe payload JSON com `event`, `timestamp` e dados do evento em menos de 5s após o evento.      |
| RF25  | Estado do pool no Drive           | Alta       | O orquestrador deve manter `pool_state.json` atualizado na conta armazém após qualquer mudança de estado de conta ou troca de sessão. | Após troca de conta, `pool_state.json` no Drive reflete o novo `active_account_index` em menos de 30s.   |
| RF26  | Compatibilidade com Ouroboros Runtime | Alta  | O servidor deve ser consumível pelo Ouroboros Runtime via sua camada de comunicação com LLMs, sem modificações no código do Ouroboros. | Uma Wave simples do Ouroboros com `provider: openai_compatible` e `base_url: http://127.0.0.1:8081/v1` completa com sucesso. |
| RF27  | Compatibilidade com Hermes Agent  | Alta       | O servidor deve ser consumível pelo Hermes Agent configurando `base_url` para o proxy local, sem modificações no código do agente. | Hermes Agent completa uma conversa de 5 turnos sem erros de parsing ou timeout.                           |
| RF28  | Compatibilidade com OpenClaw      | Alta       | O servidor deve aceitar requisições de qualquer agente compatível com a OpenAI API, incluindo o OpenClaw, sem adaptadores específicos. | OpenClaw conectado ao proxy local retorna respostas válidas para um prompt de teste.                      |

---

## 3. Requisitos Não Funcionais

### 3.1 Desempenho

| ID       | Métrica                         | Alvo                  | Medição                                          |
|----------|---------------------------------|-----------------------|--------------------------------------------------|
| RNF-P01  | Latência P50 (50 tokens)        | ≤ 5 segundos          | `pytest-benchmark` com modelo Mistral-7B 4bit    |
| RNF-P02  | Latência P95 (50 tokens)        | ≤ 15 segundos         | `pytest-benchmark` em 100 amostras               |
| RNF-P03  | Latência P99 (50 tokens)        | ≤ 30 segundos         | `pytest-benchmark` em 100 amostras               |
| RNF-P04  | Tempo de resposta do health check | ≤ 500ms             | Polling do orquestrador                          |
| RNF-P05  | Throughput máximo                | ≥ 2 req/min           | Locust com 2 usuários concorrentes               |
| RNF-P06  | Tempo de carregamento do modelo  | ≤ 5 minutos           | Medido do início da Célula 4 até `model_loaded: true` |
| RNF-P07  | MTTR (troca de conta)           | ≤ 8 minutos           | Medido do primeiro health check falho até `ACTIVE` |
| RNF-P08  | Overhead do proxy local          | ≤ 50ms adicionais     | Comparação curl direto vs. via proxy             |
| RNF-P09  | Uso de memória do orquestrador   | ≤ 256 MB RSS          | `psutil` durante soak test de 2h                 |
| RNF-P10  | Tempo de escrita do checkpoint   | ≤ 3 segundos          | Medido em `POST /v1/checkpoint`                  |

### 3.2 Segurança

| ID       | Requisito                                                                                          | Nível    |
|----------|----------------------------------------------------------------------------------------------------|----------|
| RNF-S01  | Tokens ngrok e credenciais OAuth do Drive devem ser armazenados exclusivamente em `~/.colab_infinity/` com permissões `chmod 600`. | Crítico  |
| RNF-S02  | O arquivo `accounts.json` nunca deve ser versionado em repositórios Git (entrada obrigatória em `.gitignore`). | Crítico  |
| RNF-S03  | O proxy local deve escutar exclusivamente em `127.0.0.1` (loopback), nunca em `0.0.0.0`.          | Alto     |
| RNF-S04  | Logs estruturados não devem conter tokens ngrok, API keys, e-mails completos ou qualquer credencial sensível. | Alto     |
| RNF-S05  | A API Key do servidor (quando `require_auth: true`) deve ter comprimento mínimo de 32 caracteres e gerada com `secrets.token_hex(32)`. | Médio    |
| RNF-S06  | Variáveis de ambiente sensíveis no notebook Colab (`NGROK_TOKEN`, `API_KEY`) devem ser injetadas via `os.environ`, nunca hardcoded nas células. | Alto     |
| RNF-S07  | Credenciais OAuth do Drive devem ser criptografadas em repouso quando possível; o token `drive_token.json` deve ter permissões `chmod 600`. | Médio    |
| RNF-S08  | O servidor Colab deve rejeitar requisições com payloads acima de `MAX_REQUEST_SIZE_MB` (padrão: 1 MB) para prevenir ataques de exaustão de memória. | Médio    |

### 3.3 Confiabilidade

| ID       | Requisito                                                                                          | Alvo     |
|----------|----------------------------------------------------------------------------------------------------|----------|
| RNF-R01  | Uptime do serviço (disponibilidade da API para os agentes)                                         | ≥ 95%    |
| RNF-R02  | MTBF (tempo médio entre falhas que exigem intervenção humana)                                      | ≥ 10 horas |
| RNF-R03  | Taxa de sucesso de checkpoints (checkpoints bem-sucedidos / tentativas totais)                    | ≥ 99%    |
| RNF-R04  | Taxa de sucesso de troca de conta (trocas bem-sucedidas / trocas iniciadas)                       | ≥ 98%    |
| RNF-R05  | Nenhuma perda de estado entre trocas de conta quando checkpoint está disponível                   | 100%     |
| RNF-R06  | O orquestrador deve reiniciar automaticamente após crash via supervisor (systemd, supervisord ou PM2) | Obrigatório |
| RNF-R07  | Em caso de pool exausto, o sistema deve aguardar cooldown e retomar automaticamente, sem intervenção humana, quando uma conta ficar disponível novamente | Obrigatório |

### 3.4 Manutenibilidade

| ID       | Requisito                                                                                          |
|----------|----------------------------------------------------------------------------------------------------|
| RNF-M01  | Todos os logs do orquestrador devem ser no formato JSON Lines (structlog) com campos: `timestamp`, `level`, `event`, `component`, e dados contextuais do evento. |
| RNF-M02  | O código do orquestrador deve atingir cobertura de testes ≥ 80% medida pelo `pytest-cov`.        |
| RNF-M03  | O código deve seguir o estilo `black` (formatação) e `isort` (ordenação de imports).             |
| RNF-M04  | O schema do `checkpoint.json` deve ser versionado (`schema_version`) para permitir migração entre versões sem perda de dados. |
| RNF-M05  | Logs de rotação automática: arquivo de log não deve exceder 100 MB; manter últimos 5 arquivos de backup. |
| RNF-M06  | O projeto deve incluir um `Makefile` com targets: `test`, `lint`, `format`, `coverage`, `run-orchestrator`. |

---

## 4. Requisitos de Hardware

### 4.1 Servidor de Inferência (Google Colab)

| Componente       | Mínimo             | Preferido          | Observação                                          |
|------------------|--------------------|--------------------|-----------------------------------------------------|
| GPU              | NVIDIA Tesla T4    | NVIDIA A100 40GB   | T4 disponível no plano Free; A100 no Colab Pro+     |
| VRAM             | 15 GB (T4)         | 40 GB (A100)       | Mistral-7B 4bit usa ~5 GB; Llama-3-70B 4bit usa ~38 GB |
| RAM do Colab     | 12 GB              | 25 GB (Colab Pro)  | Necessária para tokenizer e buffers de inferência   |
| Disco (Colab)    | Ephemeral (50 GB)  | —                  | Modelo é baixado do HuggingFace a cada sessão       |
| Google Drive     | 15 GB free         | 100 GB (Google One)| Armazena checkpoints, pool_state, logs              |

### 4.2 Modelos Suportados por Tier de GPU

| Modelo                               | Parâmetros | Quantização | VRAM Necessária | GPU Mínima |
|--------------------------------------|------------|-------------|-----------------|------------|
| `mistralai/Mistral-7B-Instruct-v0.3` | 7B         | 4bit        | ~5 GB           | T4 (Free)  |
| `meta-llama/Meta-Llama-3-8B-Instruct`| 8B         | 4bit        | ~6 GB           | T4 (Free)  |
| `microsoft/Phi-3-mini-4k-instruct`   | 3.8B       | 4bit        | ~3 GB           | T4 (Free)  |
| `mistralai/Mixtral-8x7B-Instruct-v0.1`| 47B (MoE) | 4bit        | ~24 GB          | A100 (Pro+)|
| `meta-llama/Meta-Llama-3-70B-Instruct`| 70B       | 4bit        | ~38 GB          | A100 (Pro+)|

### 4.3 Máquina Local (Orquestrador)

| Componente  | Mínimo            | Recomendado       |
|-------------|-------------------|-------------------|
| SO          | Linux / macOS     | Ubuntu 22.04 LTS  |
| CPU         | 2 núcleos         | 4+ núcleos        |
| RAM         | 4 GB              | 8 GB              |
| Disco       | 2 GB livres       | 10 GB livres      |
| Rede        | 10 Mbps           | 50+ Mbps          |

---

## 5. Requisitos de Software

### 5.1 Orquestrador Local (Python)

| Pacote                     | Versão Mínima | Propósito                                    |
|----------------------------|---------------|----------------------------------------------|
| Python                     | 3.10          | Runtime do orquestrador                      |
| `requests`                 | 2.31.0        | HTTP client para health checks e proxy       |
| `httpx`                    | 0.27.0        | HTTP async client para proxy                 |
| `google-auth`              | 2.29.0        | Autenticação OAuth com Google Drive          |
| `google-auth-oauthlib`     | 1.2.0         | Fluxo OAuth interativo                       |
| `google-api-python-client` | 2.130.0       | Drive API client                             |
| `pydantic`                 | 2.7.0         | Validação de schemas (checkpoint, config)    |
| `pyyaml`                   | 6.0.1         | Leitura de `colab_infinity_config.yaml`      |
| `structlog`                | 24.2.0        | Logging estruturado JSON Lines               |
| `tenacity`                 | 8.3.0         | Retry com backoff exponencial                |
| `schedule`                 | 1.2.1         | Agendamento do checkpoint periódico          |
| `click`                    | 8.1.7         | Interface de linha de comando (CLI)          |
| `fastapi`                  | 0.111.0       | Servidor do proxy local                      |
| `uvicorn[standard]`        | 0.30.1        | ASGI server para o proxy                     |

### 5.2 Servidor de Inferência (Notebook Colab)

| Pacote                     | Versão Mínima | Propósito                                    |
|----------------------------|---------------|----------------------------------------------|
| `transformers`             | 4.42.0        | Carregamento e inferência de modelos HF      |
| `accelerate`               | 0.31.0        | Distribuição automática do modelo na GPU     |
| `bitsandbytes`             | 0.43.1        | Quantização 4bit/8bit                        |
| `fastapi`                  | 0.111.0       | Framework da API de inferência               |
| `uvicorn[standard]`        | 0.30.1        | ASGI server                                  |
| `pyngrok`                  | 7.1.6         | Criação do túnel ngrok via Python            |
| `pydantic`                 | 2.7.3         | Validação de request/response                |
| `torch`                    | 2.3.0         | Backend de deep learning (GPU)               |
| `sentencepiece`            | 0.2.0         | Tokenização (Mistral, LLaMA)                 |
| `protobuf`                 | 4.25.3        | Suporte a alguns tokenizers                  |

### 5.3 Ouroboros Runtime (TypeScript / Bun)

| Componente   | Versão     | Relevância para Colab Infinity                          |
|--------------|------------|----------------------------------------------------------|
| Bun          | ≥ 1.1.0    | Runtime do Ouroboros Daemon                              |
| TypeScript   | ≥ 5.4.0    | Linguagem do Ouroboros                                   |
| `openai` npm | ≥ 4.47.0   | SDK usado pelo Ouroboros para chamar o endpoint          |
| SQLite (WAL) | —          | Memória persistente do Ouroboros (não afetada pelo Colab Infinity) |

### 5.4 Ferramentas de Desenvolvimento

| Ferramenta     | Versão     | Propósito                                    |
|----------------|------------|----------------------------------------------|
| `pytest`       | 8.2.2      | Framework de testes                          |
| `pytest-asyncio`| 0.23.7   | Testes assíncronos                           |
| `pytest-cov`   | 5.0.0      | Cobertura de testes                          |
| `black`        | 24.4.2     | Formatação de código Python                  |
| `mypy`         | 1.10.0     | Verificação estática de tipos                |
| `locust`       | 2.29.0     | Testes de carga/estresse                     |

---

## 6. Interfaces Externas

### 6.1 Interface com Agentes Consumidores (OpenAI-Compatible API)

O Colab Infinity expõe uma API REST no proxy local (`http://127.0.0.1:8081`) que qualquer agente
compatível com OpenAI pode consumir sem modificações:

```
Ouroboros Runtime  ──┐
Hermes Agent       ──┼──► http://127.0.0.1:8081/v1/chat/completions
OpenClaw           ──┘
(qualquer OpenAI-compatible client)
```

**Endpoints disponíveis:**

| Método | Endpoint               | Descrição                          |
|--------|------------------------|------------------------------------|
| GET    | `/health`              | Health check da sessão atual       |
| POST   | `/v1/chat/completions` | Inferência principal               |
| GET    | `/v1/status`           | Status detalhado da sessão e pool  |
| POST   | `/v1/checkpoint`       | Forçar checkpoint imediato         |

### 6.2 Interface com Google Drive (Conta Armazém)

O orquestrador usa a Google Drive API v3 para leitura e escrita dos arquivos de estado:

| Operação           | Arquivo                                      | Frequência            |
|--------------------|----------------------------------------------|-----------------------|
| Leitura/Escrita    | `hermes_infinito/pool_state/ngrok_url.json`  | A cada troca de conta |
| Escrita            | `hermes_infinito/checkpoints/checkpoint_*.json` | A cada 5 minutos    |
| Leitura            | `hermes_infinito/checkpoints/checkpoint_*.json` | Na inicialização     |
| Escrita            | `hermes_infinito/pool_state/pool_state.json` | A cada mudança de estado |
| Escrita (append)   | `hermes_infinito/logs/metrics.jsonl`         | A cada hora          |

### 6.3 Interface com ngrok

O notebook Colab usa a biblioteca `pyngrok` para:
1. Autenticar com `ngrok_lib.set_auth_token(NGROK_TOKEN)`
2. Criar túnel: `tunnel = ngrok_lib.connect(8000, "http")`
3. Obter URL pública: `public_url = tunnel.public_url`
4. Publicar URL no Drive para o orquestrador

**Limitações do ngrok Free:**
- 1 túnel ativo simultâneo por conta
- URL aleatória muda a cada nova sessão
- Sem domínios customizados
- Sem autenticação nativa de requests (deve ser implementada na aplicação)

### 6.4 Interface com o Ouroboros Runtime

O Ouroboros Runtime se conecta ao Colab Infinity via SDK OpenAI configurado para apontar ao proxy local.
A integração é feita via variável de ambiente ou arquivo de configuração do Ouroboros:

```typescript
// Exemplo ilustrativo — configuração no Ouroboros Runtime
const llmClient = new OpenAI({
  baseURL: process.env.COLAB_INFINITY_BASE_URL ?? "http://127.0.0.1:8081/v1",
  apiKey: process.env.COLAB_INFINITY_API_KEY ?? "no-key",
});
```

---

## 7. Restrições e Dependências Externas

### 7.1 Termos de Serviço do Google Colab

| Restrição                                    | Impacto                                              | Mitigação                                         |
|----------------------------------------------|------------------------------------------------------|---------------------------------------------------|
| Uso comercial não permitido no plano Free    | Colab Infinity deve ser usado apenas para fins não comerciais e de pesquisa | Documentar uso como projeto de pesquisa/estudo   |
| Limite de tempo de GPU (~12h por sessão)     | Sessões precisam ser rotacionadas                   | Mecanismo de troca automática de conta             |
| Proibição de usar para mineração ou computação em lote não interativa | O uso como servidor de API pode violar os ToS | Ver Análise de Riscos (`08_risk_analysis.md`)    |
| Limite de disco ephemeral (~50 GB por sessão)| Modelos grandes precisam de download a cada sessão  | Cache no Drive (se dentro do limite de 15 GB)     |

### 7.2 Limites do ngrok Free

| Limite                          | Valor          | Impacto                                        |
|---------------------------------|----------------|------------------------------------------------|
| Túneis simultâneos              | 1              | Uma sessão Colab ativa por vez por conta ngrok |
| Conexões por minuto             | 40             | Suficiente para uso pelo Ouroboros Runtime     |
| Transferência de dados          | 1 GB/mês       | Adequado para texto (tokens LLM)               |
| URLs de domínio customizado     | Não disponível | URL muda a cada sessão; proxy local abstrai isso |
| TLS automático                  | Sim            | HTTPS disponível automaticamente               |

### 7.3 Limites do Google Drive (Free)

| Limite                          | Valor          | Impacto                                        |
|---------------------------------|----------------|------------------------------------------------|
| Armazenamento total             | 15 GB          | Checkpoints JSON são pequenos (~5 KB cada)     |
| Uploads por dia                 | 750 GB         | Não é um limitante para checkpoints            |
| Requisições de API por segundo  | 10             | Health checks e checkpoints respeitam esse limite |
| Tamanho máximo por arquivo      | 5 TB           | Sem impacto para os arquivos do Colab Infinity |

---

## 8. Modelos de Dados

### 8.1 Schema: `checkpoint.json`

```json
{
  "schema_version": "1.1",
  "saved_at": "2025-07-14T08:00:00Z",
  "save_reason": "periodic",
  "checkpoint_id": "ckpt_20250714_080000_periodic",
  "session": {
    "session_id": "sess_colab_20250714_031500",
    "started_at": "2025-07-14T03:15:00Z",
    "requests_served": 847,
    "tokens_generated": 284312,
    "uptime_seconds": 17700
  },
  "active_account_index": 1,
  "ngrok_url": "https://a1b2c3d4.ngrok-free.app",
  "model": {
    "model_id": "mistralai/Mistral-7B-Instruct-v0.3",
    "quantization": "4bit",
    "context_length": 4096
  },
  "pool_state": {
    "total_accounts": 4,
    "accounts": [
      {"index": 0, "status": "exhausted", "exhausted_at": "2025-07-13T15:20:00Z"},
      {"index": 1, "status": "active",    "activated_at": "2025-07-14T03:15:00Z"},
      {"index": 2, "status": "available", "last_used_at": null},
      {"index": 3, "status": "available", "last_used_at": null}
    ]
  },
  "metrics": {
    "total_requests_all_sessions": 2341,
    "total_tokens_all_sessions": 887234,
    "total_account_switches": 3,
    "total_checkpoints_saved": 59
  }
}
```

**Campos do Schema:**

| Campo                          | Tipo    | Descrição                                                              |
|--------------------------------|---------|------------------------------------------------------------------------|
| `schema_version`               | string  | Versão do schema para migração. Atual: `"1.1"`                         |
| `saved_at`                     | string  | ISO 8601 UTC do momento do salvamento                                  |
| `save_reason`                  | string  | `"periodic"`, `"manual"`, `"quota_warning"`, `"pre_switch"`           |
| `checkpoint_id`                | string  | Identificador único do checkpoint                                      |
| `session.session_id`           | string  | ID único da sessão Colab                                               |
| `session.requests_served`      | integer | Total de requisições atendidas na sessão atual                         |
| `session.tokens_generated`     | integer | Total de tokens gerados na sessão atual                                |
| `active_account_index`         | integer | Índice (0-based) da conta atualmente ativa no pool                    |
| `ngrok_url`                    | string  | URL ngrok da sessão ativa no momento do checkpoint                     |
| `pool_state.accounts[].status` | string  | `"available"`, `"active"`, `"exhausted"`, `"banned"`, `"cooldown"`   |
| `metrics`                      | object  | Contadores acumulados de todas as sessões históricas                   |

### 8.2 Schema: `pool_state.json`

```json
{
  "schema_version": "1.0",
  "last_updated": "2025-07-14T08:00:05Z",
  "active_account_index": 1,
  "switch_requested": false,
  "switch_reason": null,
  "ngrok_url": "https://a1b2c3d4.ngrok-free.app",
  "session_id": "sess_colab_20250714_031500",
  "accounts": [
    {"index": 0, "status": "exhausted", "exhausted_at": "2025-07-13T15:20:00Z", "email_masked": "pr***@gmail.com"},
    {"index": 1, "status": "active",    "activated_at": "2025-07-14T03:15:00Z", "email_masked": "se***@gmail.com"},
    {"index": 2, "status": "available", "last_used_at": null,                   "email_masked": "te***@gmail.com"},
    {"index": 3, "status": "available", "last_used_at": null,                   "email_masked": "qu***@gmail.com"}
  ]
}
```

### 8.3 Schema: `ngrok_url.json`

```json
{
  "url": "https://a1b2c3d4.ngrok-free.app",
  "session_id": "sess_colab_20250714_031500",
  "account_index": 1,
  "updated_at": "2025-07-14T03:15:42Z",
  "model": "mistralai/Mistral-7B-Instruct-v0.3"
}
```

### 8.4 Schema: `colab_infinity_config.yaml`

```yaml
# Arquivo principal de configuração — ~/.colab_infinity/colab_infinity_config.yaml

project:
  name: "colab-infinity"
  version: "1.0.0"
  environment: "production"

pool:
  accounts_file: "~/.colab_infinity/accounts.json"
  min_available_accounts: 2
  switch_threshold_hours: 10
  cooldown_hours: 24

colab:
  notebook_id: "1aBcDeFgHiJkLmNoPqRsTuVwXyZ"
  startup_timeout_seconds: 300
  health_check_interval_seconds: 30
  health_check_fail_threshold: 3
  quota_warning_minutes: 30
  model_id: "mistralai/Mistral-7B-Instruct-v0.3"
  quantization: "4bit"
  max_tokens_limit: 4096

checkpoint:
  interval_seconds: 300
  max_files: 10
  drive_folder: "hermes_infinito/checkpoints"

proxy:
  host: "127.0.0.1"
  port: 8081
  request_timeout_seconds: 120
  max_retries: 3
  retry_backoff_seconds: 5

server:
  api_key: null
  require_auth: false
  max_request_size_mb: 1
  rate_limit_per_second: 2
  rate_limit_per_minute: 60

drive:
  credentials_file: "~/.colab_infinity/drive_credentials.json"
  token_file: "~/.colab_infinity/drive_token.json"
  warehouse_folder_id: "1aBcDeFgHiJkLmNoPqRsTuVwXyZ"

logging:
  level: "INFO"
  format: "json"
  file: "~/.colab_infinity/logs/orchestrator.log"
  max_size_mb: 100
  backup_count: 5

notifications:
  enabled: false
  webhook_url: null
  events:
    - "account_switched"
    - "pool_exhausted"
    - "checkpoint_saved"
    - "server_error"
```

---

## 9. Casos de Uso Principais

### 9.1 UC-01: Operação Normal — Ouroboros Runtime envia prompt ao LLM

**Ator Principal:** Ouroboros Runtime (Daemon)
**Pré-condição:** Proxy local ativo em `127.0.0.1:8081`; sessão Colab ativa com modelo carregado.

**Fluxo Principal:**
1. O Ouroboros Daemon constrói um payload `chat/completions` para um de seus agentes (Vision, Architect, etc.)
2. O payload é enviado via HTTP para `http://127.0.0.1:8081/v1/chat/completions`
3. O proxy local do orquestrador roteia a requisição para `https://<hash>.ngrok-free.app/v1/chat/completions`
4. O ngrok encaminha para o FastAPI rodando no Colab
5. O FastAPI formata o prompt com o template do modelo e executa `model.generate()`
6. A resposta é retornada pelo mesmo caminho
7. O Ouroboros Daemon recebe a `ChatCompletion` e passa para o agente correspondente

**Fluxo Alternativo — Modelo ainda carregando:**
- Passo 4 retorna `503 MODEL_LOADING`
- Proxy local aguarda 20s e retenta (até `max_retries` vezes)

### 9.2 UC-02: Rotação Automática por Quota Esgotada

**Ator Principal:** Orquestrador (automático)
**Pré-condição:** Conta ativa com menos de 30 minutos de quota restante.

**Fluxo:**
1. Monitor de quota no notebook detecta `remaining_minutes < QUOTA_WARNING_MINUTES`
2. Notebook salva checkpoint com `save_reason: "quota_warning"` e atualiza `pool_state.json` com `switch_requested: true`
3. Orquestrador local detecta `switch_requested: true` no Drive
4. Orquestrador inicia nova sessão Colab com a próxima conta disponível do pool
5. Nova sessão carrega o modelo e cria novo túnel ngrok
6. Nova URL ngrok é publicada no Drive
7. Orquestrador atualiza o proxy local com a nova URL de destino
8. Conta anterior é marcada como `exhausted` com timestamp

### 9.3 UC-03: Restart com Restauração de Checkpoint

**Ator Principal:** Operador (restart manual ou crash recovery)
**Pré-condição:** Checkpoint válido disponível no Drive.

**Fluxo:**
1. Orquestrador inicia e detecta arquivo `checkpoint_*.json` mais recente no Drive
2. Carrega estado: `active_account_index`, `pool_state`, contadores acumulados
3. Inicia nova sessão Colab com a conta indicada no checkpoint (se disponível)
4. Restaura contadores de métricas a partir dos valores salvos
5. Proxy local é iniciado; agentes retomam operação normalmente

### 9.4 UC-04: Pool Exausto — Aguardar Cooldown

**Ator Principal:** Orquestrador (automático)
**Pré-condição:** Todas as contas do pool com status `exhausted` ou `banned`.

**Fluxo:**
1. Orquestrador verifica pool e não encontra conta com status `available`
2. Proxy local começa a retornar `503 SESSION_SWITCHING` para todas as requisições
3. Evento `pool_exhausted` é disparado (webhook, se configurado)
4. Orquestrador entra em loop de polling a cada 5 minutos verificando se alguma conta passou pelo cooldown de 24h
5. Quando uma conta fica disponível, orquestrador inicia nova sessão automaticamente
6. Sistema retorna ao estado `ACTIVE` sem intervenção humana

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*