# Documento de Arquitetura (SAD)

## 1. Visão Geral da Arquitetura

O Colab Infinity opera dividindo responsabilidades entre uma máquina local (onde o Ouroboros e o Orquestrador residem) e a nuvem do Google Colab (onde o modelo LLM é processado e servido). O Cloudflare Tunnel liga os dois mundos.

### 1.1 Diagrama de Componentes (ASCII)

```text
    +-------------------------------------------------------------+
    |                        AMBIENTE LOCAL                       |
    |                                                             |
    |  +--------------------+         +-----------------------+   |
    |  | Ouroboros Runtime  |         | Orquestrador (Python) |   |
    |  | (Daemon / TS)      |         | + Playwright          |   |
    |  +---------+----------+         +-----------+-----------+   |
    |            |                                |               |
    |   [API Request / RPC]              [Automação de Browser]   |
    |            |                                |               |
    +------------|--------------------------------|---------------+
                 |                                |
                 v                                v
       +-------------------+         [Troca conta e inicia Nbk]
       | Cloudflare Tunnel |<---------------------+
       | (URL Dinâmica)    |                      |
       +---------+---------+                      |
                 |                                |
    +------------|--------------------------------|---------------+
    |            |         GOOGLE COLAB           |               |
    |            v                                v               |
    |  +-------------------------------------------------------+  |
    |  |                      Notebook (.ipynb)                |  |
    |  |                                                       |  |
    |  |  +----------------+  +-----------------+  +--------+  |  |
    |  |  | FastAPI Server |  | Checkpoint Mgr  |  | Monitor|  |  |
    |  |  +-------+--------+  +--------+--------+  +--------+  |  |
    |  |          |                    |                       |  |
    |  |  +-------v--------+           |                       |  |
    |  |  | LLM 4-bit Quant|           |                       |  |
    |  |  | (Mistral-7B)   |           |                       |  |
    |  |  +----------------+           |                       |  |
    |  +-------------------------------+-----------------------+  |
    |                                  |                          |
    |                                  v                          |
    |  +-------------------------------------------------------+  |
    |  |            GOOGLE DRIVE (Conta Armazém)               |  |
    |  |               (Arquivos de Checkpoint)                |  |
    |  +-------------------------------------------------------+  |
    +-------------------------------------------------------------+
```

## 2. Descrição dos Componentes

1. **Ouroboros Runtime:** Agente autônomo baseado em Bun/TypeScript. Faz requisições ao endpoint remoto que simula a OpenAI em suas tomadas de decisão.
2. **Orquestrador:** Script local em Python que monitora a URL do túnel e, em caso de erro 502/down, lança uma automação do Playwright, limpa a sessão atual, faz login em uma conta Google de fallback do pool e roda o notebook do Colab novamente.
3. **Notebook Colab:** Epicentro remoto do sistema. Executa três subprocessos vitais: a API com FastAPI; o gerenciamento de quantização (bitsandbytes) para rodar os pesos Mistral-7B; o monitor de cotas que interage com o GDrive.
4. **Cloudflare Tunnel (`cloudflared`):** Roteador seguro de rede para exportar o processo `localhost:8000` (FastAPI) da máquina do Google para a internet sem portas diretas.
5. **Google Drive Armazém:** Uma conta Google dedicada apenas para armazenar artefatos de estado. Todas as contas de processamento (Worker Accounts) devem ter permissões de edição na pasta compartilhada deste Drive.

## 3. Fluxos e Modelos de Dados

### 3.1 Fluxo Normal de Operação
- Ouroboros (ou Hermes) cria um prompt complexo e converte para JSON de OpenAI `/v1/chat/completions`.
- O payload viaja até o túnel da Cloudflare e alcança o FastAPI no Colab.
- FastAPI transforma mensagens OpenAI em um prompt padronizado (Chat Template) via tokenizador do Hugging Face.
- O modelo Mistral-7B processa em GPU (4-bits) os tokens e gera a resposta via streaming ou estático.
- A resposta volta por todo o caminho até o Ouroboros.

### 3.2 Fluxo de Fallback (Troca de Conta)
```text
  Orquestrador             Colab Worker 1            Colab Worker 2
       |                         |                          |
       |-- Ping Tunnel Alive? ---|                          |
       |<-- Timeout / 502 -------|                          |
       |                         |                          |
       |-- Detecta Cota Limite --|                          |
       |                         |                          |
       |-- Inicia Playwright ----|                          |
       |-- Logout Worker 1 ------x                          |
       |                                                    |
       |-- Login Worker 2 --------------------------------->|
       |-- Abre Notebook & Executa Tudo ------------------->|
       |                                                    |
       |<-- Novo Cloudflare Tunnel URL Pronta --------------|
       |                                                    |
       |-- Atualiza Ouroboros Env/Daemon c/ URL Nova ------|
       |                                                    |
```

## 4. Decisões Arquiteturais

1. **Por que Quantização 4-bits (bitsandbytes):** Uma GPU T4 de colab gratuito dispõe de apenas 15GB/16GB de VRAM. Modelos 7B brutos (em fp16) consomem mais de 14GB e causam Out-Of-Memory na geração de contexto (KV cache). O 4-bits reduz a pegada de VRAM para ~5-6GB.
2. **Por que Cloudflare em vez de Ngrok:** Os túneis gratuitos da Cloudflare não deslogam proativamente conexões nem exigem reinícios como os limites estritos da cota atual do ngrok grátis, sendo muito mais adaptado ao uso headless prolongado.
3. **Por que Conta Armazém separada:** Como a conta Worker é trocada no fluxo de fallback, seu Google Drive será desmontado da VM. Usar um repositório GDrive compartilhado central de outra conta isolada ("Armazém") garante que o progresso sempre seja acessível globalmente a qualquer Worker que o acesse.
