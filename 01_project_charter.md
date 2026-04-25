# Colab Infinity — Documento de Visão e Escopo (Project Charter)

**Versão:** 1.0.0
**Data:** 2025-07-14
**Status:** Aprovado
**Responsável Técnico:** Equipe Colab Infinity
**Referências:** `02_srs.md`, `03_sad.md`, `09_integration_guide.md`

---

## Índice

1. [Resumo Executivo](#1-resumo-executivo)
2. [Problema](#2-problema)
3. [Objetivos SMART](#3-objetivos-smart)
4. [Público-Alvo e Stakeholders](#4-público-alvo-e-stakeholders)
5. [Escopo do Projeto](#5-escopo-do-projeto)
6. [Métricas de Sucesso](#6-métricas-de-sucesso)
7. [Premissas e Restrições](#7-premissas-e-restrições)
8. [Riscos de Alto Nível](#8-riscos-de-alto-nível)
9. [Glossário](#9-glossário)

---

## 1. Resumo Executivo

**Colab Infinity** é um módulo de infraestrutura de LLM (Large Language Model) projetado para
fornecer capacidade de inferência contínua, resiliente e de custo zero ao ecossistema de agentes
autônomos que compõem o **Ouroboros Runtime** e seus agentes satélite (Hermes Agent, OpenClaw e
qualquer cliente compatível com a API OpenAI).

O sistema transforma o **Google Colab gratuito** — normalmente limitado a sessões de até 12 horas
por conta — em um servidor de LLM de disponibilidade contínua, mediante o uso de um pool de
múltiplas contas Google com rotação automática, checkpointing de estado no Google Drive e exposição
de um endpoint HTTP compatível com a API `POST /v1/chat/completions` da OpenAI.

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
│   Qualquer cliente         :8080                                     │
│   OpenAI-compat.    ──┘                                              │
│                                      ▲                               │
│                              Rotação Automática                      │
│                              de Contas Google                        │
│                              + Checkpoint Drive                      │
└──────────────────────────────────────────────────────────────────────┘
```

O Colab Infinity **não é um agente**, não toma decisões de negócio e não gerencia prompts. Ele é
exclusivamente uma camada de infraestrutura — análoga a um servidor de inferência como Ollama ou
LM Studio — porém operando sobre hardware gratuito em nuvem com tolerância automática a falhas.

---

## 2. Problema

### 2.1 Contexto

O **Ouroboros Runtime** (https://github.com/RenyEnnos/ouroboros-runtime) é um sistema de
orquestração multiagente sofisticado, com daemon persistente em Bun/TypeScript, memória SQLite com
WAL mode, sandbox Python isolado e um Conselho de agentes especializados (Ouroboros Core, Vision,
Architect, Guardian, Kinetic). Esses agentes dependem de modelos de linguagem para raciocínio,
geração de código e tomada de decisão.

O **problema central** é o custo e a fricção operacional do acesso a LLMs de qualidade:

| Alternativa Atual         | Custo Mensal Estimado | Limitações                                           |
|---------------------------|-----------------------|------------------------------------------------------|
| OpenAI API (GPT-4o)       | USD 50–500+           | Custo proporcional ao volume; vendor lock-in          |
| Anthropic API (Claude)    | USD 30–300+           | Custo proporcional; rate limits restritivos           |
| Ollama local              | USD 0                 | Exige GPU dedicada (hardware caro ou indisponível)   |
| Google Colab (sessão única)| USD 0                | Limite de 12h por sessão; desconecta por inatividade |
| Colab Pro/Pro+            | USD 10–50/mês         | Ainda limitado; não adequado para daemon contínuo    |

### 2.2 Limitações Específicas do Google Colab Gratuito

| Limitação                        | Impacto no Ouroboros Runtime                                  |
|----------------------------------|---------------------------------------------------------------|
| Sessão máxima de ~12 horas       | Daemon do Ouroboros precisa de LLM disponível 24/7            |
| Desconexão por inatividade (~90 min) | Agentes idle desconectam o servidor de inferência         |
| 1 GPU por conta simultaneamente  | Impossibilita alta disponibilidade com conta única            |
| Cota diária de GPU limitada      | Contas com uso intenso ficam sem GPU até o dia seguinte       |
| Sem API programática de controle | Não há como reiniciar sessão automaticamente via CLI oficial  |
| GPU não garantida (T4 ou nenhuma)| Disponibilidade de GPU varia por horário e região             |

### 2.3 Por que este Problema Precisa ser Resolvido Agora

O Ouroboros Runtime está em fase de desenvolvimento ativo com o **Protocolo Anti-Vibe** — um
pipeline rigoroso de qualidade que exige: especificação → validação → revisão humana → testes →
promoção de código. Cada etapa consome chamadas de LLM. Pagar por API comercial durante o
desenvolvimento é proibitivo; usar modelo local exige hardware dedicado. O Colab Infinity
elimina ambas as barreiras.

---

## 3. Objetivos SMART

| ID     | Objetivo                                                                                     | Prazo    |
|--------|----------------------------------------------------------------------------------------------|----------|
| OBJ-01 | **Disponibilidade ≥ 95%**: o endpoint `/v1/chat/completions` deve estar disponível ≥ 95% do tempo medido em janelas de 24h | 30 dias após deploy |
| OBJ-02 | **MTTR ≤ 8 minutos**: o tempo médio de recuperação após falha (expiração de conta, queda de ngrok) deve ser inferior a 8 minutos | 30 dias após deploy |
| OBJ-03 | **Latência P95 ≤ 15s**: para requisições com `max_tokens ≤ 512`, o percentil 95 de latência deve ser inferior a 15 segundos | 30 dias após deploy |
| OBJ-04 | **Zero perda de contexto**: em trocas de conta, o estado do checkpoint deve ser restaurado sem perda de dados (0 checkpoints corrompidos em 30 dias) | Contínuo |
| OBJ-05 | **Compatibilidade total com Ouroboros Runtime**: todas as waves do Ouroboros que realizam chamadas de LLM devem funcionar sem modificação no código do agente | Antes do primeiro deploy |
| OBJ-06 | **Pool mínimo de 3 contas**: o sistema deve suportar operação contínua com pool de 3 contas Google, cobrindo ao menos 36 horas de GPU antes de necessitar reciclagem | 15 dias após deploy |

---

## 4. Público-Alvo e Stakeholders

### 4.1 Usuários Primários

| Ator                        | Perfil                                                              | Necessidade Principal                           |
|-----------------------------|---------------------------------------------------------------------|-------------------------------------------------|
| **Desenvolvedor do Ouroboros Runtime** | Engenheiro de software desenvolvendo agentes autônomos com Bun/TS | LLM disponível 24/7 sem custo para o daemon    |
| **Usuário do Hermes Agent** | Desenvolvedor ou pesquisador usando o agente conversacional Hermes | Endpoint local de LLM compatível com OpenAI API |
| **Usuário do OpenClaw**     | Desenvolvedor usando agente de automação OpenClaw                  | Provider de LLM configurável e de baixo custo  |
| **Operador do Sistema**     | Pessoa responsável por manter o pool de contas e o orquestrador    | Ferramentas de monitoramento e recuperação      |

### 4.2 Stakeholders

| Stakeholder                   | Interesse                                      | Influência |
|-------------------------------|------------------------------------------------|------------|
| Ouroboros Runtime (projeto)   | Infraestrutura de LLM para agentes             | Alta       |
| Hermes Agent (projeto)        | Provider de LLM alternativo ao OpenAI          | Alta       |
| OpenClaw (projeto)            | Integração com LLM local/remoto                | Média      |
| Google (externo)              | Titular dos recursos Colab e Drive usados      | Alta       |
| ngrok (externo)               | Provedor do serviço de túnel HTTP              | Média      |
| HuggingFace (externo)         | Host dos modelos LLM quantizados               | Média      |

### 4.3 Sistemas Consumidores

```
┌─────────────────────────────────────────────────────────┐
│              ECOSSISTEMA OUROBOROS                       │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Ouroboros Runtime (Daemon — Bun/TypeScript)       │   │
│  │  ├── Ouroboros Core Agent                         │   │
│  │  ├── Vision Agent                                 │   │
│  │  ├── Architect Agent                              │   │
│  │  ├── Guardian Agent                               │   │
│  │  └── Kinetic Agent                                │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │ OpenAI-compatible API calls        │
│  ┌──────────────────┴───────────────────────────────┐   │
│  │ Hermes Agent           OpenClaw Agent             │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │                                    │
│                     ▼                                    │
│            COLAB INFINITY (this project)                 │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Escopo do Projeto

### 5.1 O que o Colab Infinity FAZ

| # | Funcionalidade                                                                                         |
|---|--------------------------------------------------------------------------------------------------------|
| 1 | Carrega modelo LLM quantizado (4-bit via bitsandbytes) em GPU T4 do Google Colab gratuito             |
| 2 | Expõe endpoint `POST /v1/chat/completions` compatível com a API OpenAI                                |
| 3 | Cria e gerencia túnel público via ngrok para expor a API à rede local do operador                     |
| 4 | Monitora o tempo de sessão do Colab e detecta proximidade do limite de ~12 horas                      |
| 5 | Salva checkpoints de estado em JSON no Google Drive (conta armazém) a intervalos configuráveis        |
| 6 | Alterna automaticamente para a próxima conta Google do pool quando a cota se esgota                   |
| 7 | Mantém um proxy local (`127.0.0.1:8080`) que redireciona requisições para a URL ngrok ativa           |
| 8 | Restaura estado a partir do último checkpoint válido após reinicialização ou troca de conta           |
| 9 | Expõe endpoint `GET /health` para monitoramento de vivacidade pelo orquestrador                       |
| 10 | Registra logs estruturados (JSON Lines) com métricas de uso, latência e eventos de troca de conta    |
| 11 | Mantém a sessão Colab ativa via script de keep-alive (JavaScript no browser) para evitar desconexão  |
| 12 | Suporta configuração de pool com mínimo de 3 contas (1 armazém + 2 operacionais) e máximo ilimitado  |
| 13 | É compatível com qualquer cliente que implemente a API OpenAI Chat Completions (`openai` SDK, etc.)  |
| 14 | Fornece CLI local (`colab-infinity`) para operações de gerenciamento do pool e diagnóstico           |

### 5.2 O que o Colab Infinity NÃO FAZ

| # | Fora do Escopo                                                                                          |
|---|---------------------------------------------------------------------------------------------------------|
| 1 | **Não é um agente autônomo**: não toma decisões, não planeja tarefas, não acessa ferramentas           |
| 2 | **Não substitui o Ouroboros Runtime**: não orquestra agentes, não gerencia memória ou sessões de agentes |
| 3 | **Não gerencia prompts ou contexto**: não faz engenharia de prompt, não mantém histórico de conversas  |
| 4 | **Não suporta fine-tuning**: apenas inferência; treinamento de modelos está fora do escopo             |
| 5 | **Não suporta multimodalidade**: sem visão computacional, sem geração de imagens ou áudio              |
| 6 | **Não garante qualidade das respostas do LLM**: qualidade depende do modelo escolhido                  |
| 7 | **Não cria contas Google automaticamente**: o operador deve criar e configurar as contas manualmente   |
| 8 | **Não gerencia segredos corporativos**: não é um vault; credenciais são responsabilidade do operador   |
| 9 | **Não funciona sem intervenção humana para abrir o notebook**: a abertura da sessão Colab é manual     |
| 10 | **Não oferece SLA comercial**: é software open-source sem garantias de disponibilidade contratual     |

---

## 6. Métricas de Sucesso

### 6.1 KPIs Primários

| KPI                              | Meta           | Método de Medição                                     |
|----------------------------------|----------------|-------------------------------------------------------|
| Disponibilidade do endpoint      | ≥ 95%          | Health checks a cada 30s; janela de 24h               |
| MTTR (Mean Time to Recovery)     | ≤ 8 minutos    | Timestamp de falha vs. timestamp de recuperação nos logs |
| Latência P95 (512 tokens)        | ≤ 15 segundos  | Distribuição de latência no arquivo `metrics.jsonl`   |
| Taxa de sucesso de checkpoint     | ≥ 99%          | Checkpoints salvos / checkpoints tentados nos logs    |
| Taxa de troca de conta bem-sucedida | ≥ 98%       | Trocas bem-sucedidas / trocas tentadas nos logs       |

### 6.2 KPIs Secundários

| KPI                              | Meta           | Método de Medição                                     |
|----------------------------------|----------------|-------------------------------------------------------|
| Requisições servidas por sessão  | ≥ 100          | Campo `requests_served` no `pool_state.json`          |
| Latência P50 (512 tokens)        | ≤ 5 segundos   | Distribuição de latência no `metrics.jsonl`           |
| Consumo de VRAM                  | ≤ 85%          | Monitoramento via `torch.cuda.memory_allocated()`     |
| Tempo de startup do servidor     | ≤ 5 minutos    | Delta entre abertura do notebook e primeiro `/health` bem-sucedido |
| Uptime acumulado em 7 dias       | ≥ 95%          | Soma dos períodos de disponibilidade / 168h           |

### 6.3 Critérios de Falha do Projeto

O projeto é considerado **não apto para produção** se qualquer um dos seguintes ocorrer:

- Disponibilidade cair abaixo de 90% em qualquer janela de 24 horas por mais de 3 dias consecutivos
- MTTR médio superar 20 minutos em um período de 7 dias
- Checkpoint corrompido causando perda de estado de pool (conta banida não detectada)
- Incompatibilidade com o Ouroboros Runtime que exija modificação no código do agente

---

## 7. Premissas e Restrições

### 7.1 Premissas

| # | Premissa                                                                                             |
|---|------------------------------------------------------------------------------------------------------|
| 1 | O operador tem acesso a pelo menos 3 contas Google válidas com verificação por telefone             |
| 2 | O Google Colab gratuito mantém disponibilidade de GPU T4 por pelo menos 8 horas consecutivas        |
| 3 | O plano gratuito do ngrok permite 1 túnel ativo simultâneo com requisições ilimitadas               |
| 4 | Os modelos Mistral-7B-Instruct e Llama-3-8B-Instruct cabem em GPU T4 (15 GB VRAM) com quantização 4-bit |
| 5 | O Ouroboros Runtime suporta configuração de endpoint LLM customizado via variável de ambiente       |
| 6 | O Google Drive oferece pelo menos 15 GB gratuitos suficientes para checkpoints e logs               |
| 7 | A máquina local do operador tem acesso ininterrupto à internet para manter o orquestrador ativo     |
| 8 | O ngrok não altera o corpo das requisições HTTP (apenas o roteamento de rede)                       |

### 7.2 Restrições

| # | Restrição                                              | Tipo       | Impacto                                              |
|---|--------------------------------------------------------|------------|------------------------------------------------------|
| 1 | Sessão Colab gratuita limitada a ~12 horas             | Técnica    | Exige rotação de contas a cada 10-11 horas           |
| 2 | Política de Uso Aceitável do Google Colab              | Legal      | Uso excessivo pode levar a banimento das contas      |
| 3 | plano free do ngrok: 1 túnel, URLs aleatórias          | Técnica    | URL muda a cada sessão; orquestrador deve atualizar  |
| 4 | Sem API oficial do Colab para controle programático    | Técnica    | Abertura de sessão requer intervenção humana         |
| 5 | Modelos open-source podem ter licenças restritivas     | Legal      | Verificar licença do modelo antes de usar em produção|
| 6 | GPU T4 não garantida no plano gratuito                 | Operacional| Sessão pode iniciar sem GPU; orquestrador deve detectar |
| 7 | Tokens ngrok gratuito: sem autenticação de IP          | Segurança  | Endpoint público; proteção via API key recomendada   |
| 8 | Latência adicional de 50-200ms pelo túnel ngrok        | Desempenho | Latência total da API ligeiramente superior ao direto|

---

## 8. Riscos de Alto Nível

| ID    | Risco                                      | Probabilidade | Impacto | Mitigação Resumida                              |
|-------|--------------------------------------------|---------------|---------|-------------------------------------------------|
| R-01  | Google banir conta por uso excessivo       | Média         | Alto    | Pool de contas, rotação conservadora, cooldown  |
| R-02  | GPU T4 indisponível no horário de pico     | Alta          | Médio   | Detectar na inicialização; tentar outra conta   |
| R-03  | ngrok descontinuar plano gratuito          | Baixa         | Alto    | Avaliar Cloudflare Tunnel como alternativa      |
| R-04  | Checkpoint corrompido                      | Baixa         | Médio   | Escrita atômica; validação de schema ao carregar|
| R-05  | Mudança na API do Google Colab             | Média         | Alto    | Monitorar releases; abstrair dependências do Colab |
| R-06  | Modelo LLM gera respostas com formato inválido | Baixa     | Baixo   | Validação de schema na resposta pelo proxy      |

*Análise completa em `08_risk_analysis.md`.*

---

## 9. Glossário

| Termo                     | Definição                                                                                       |
|---------------------------|-------------------------------------------------------------------------------------------------|
| **Colab Infinity**        | Este projeto; módulo de infraestrutura LLM para agentes autônomos                              |
| **Ouroboros Runtime**     | Sistema de orquestração multiagente (Bun/TypeScript) que consome o Colab Infinity              |
| **Hermes Agent**          | Agente conversacional do ecossistema Ouroboros que usa LLM via API compatível com OpenAI       |
| **OpenClaw**              | Agente de automação do ecossistema que consome a mesma API                                     |
| **Wave**                  | Unidade de trabalho do Ouroboros Runtime; sequência de chamadas de agentes para resolver tarefa|
| **Protocolo Anti-Vibe**   | Pipeline de qualidade do Ouroboros: especificação → validação → revisão → testes → promoção    |
| **Pool de Contas**        | Conjunto de contas Google rotativas usadas para prover sessões Colab                           |
| **Conta Armazém**         | Conta Google dedicada a armazenar estado (checkpoints, pool_state.json) no Google Drive        |
| **Checkpoint**            | Snapshot do estado operacional salvo em JSON no Drive para recuperação após falha              |
| **ngrok**                 | Serviço de túnel HTTP que expõe a porta local do Colab para a internet                         |
| **Proxy Local**           | Processo local que recebe requisições dos agentes e as redireciona para a URL ngrok ativa      |
| **Orquestrador**          | Script Python local que gerencia o ciclo de vida do pool, monitora saúde e executa trocas      |
| **Quantização 4-bit**     | Técnica de compressão de modelo (bitsandbytes NF4) que reduz uso de VRAM de ~28 GB para ~5 GB |
| **MTTR**                  | Mean Time to Recovery; tempo médio para restaurar o serviço após uma falha                     |
| **Keep-alive**            | Script JavaScript injetado no browser para prevenir desconexão por inatividade do Colab        |
| **pool_state.json**       | Arquivo no Drive com o estado atual do pool (contas disponíveis, ativas, exauridas)            |
| **colab_server.ipynb**    | Notebook Colab que executa o servidor FastAPI + modelo LLM                                     |
| **VRAM**                  | Video RAM; memória da GPU T4 (15 GB no Colab) usada para alocar o modelo LLM                  |

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*