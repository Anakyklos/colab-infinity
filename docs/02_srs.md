# Especificação de Requisitos de Software (SRS)

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `01_project_charter.md`, `03_architecture.md`

---

## 1. Introdução e Propósito

Este documento de Especificação de Requisitos de Software (SRS) delineia os Requisitos Funcionais, Não Funcionais, Regras de Negócio e restrições tecnológicas da plataforma **Colab Infinity**. Ele serve como fonte única da verdade para a equipe de desenvolvimento de infraestrutura, garantindo o suporte estável, escalável e de "custo zero" para agentes como o Ouroboros Runtime, suportado por um orquestrador local e túneis estritos através do **Ngrok**.

---

## 2. Requisitos Funcionais (RF)

| ID | Nome do Requisito | Criticidade | Descrição | Critério de Aceitação (DoD) |
|---|---|---|---|---|
| **RF01** | Carga de Modelo Quantizado | Alta | O sistema no Colab deve carregar um LLM pré-treinado (priorizando modelos focados em instrução) suportando explicitamente quantização 4-bits (`bitsandbytes`) para caber na GPU T4. | Modelo como `Mistral-7B-Instruct-v0.2` carregado na VRAM utilizando < 10GB. |
| **RF02** | Endpoint Compatível OpenAI | Crítico | O servidor local FastAPI rodando no Colab deve expor a rota `POST /v1/chat/completions` interpretando *payloads* no padrão exato da OpenAI (suportando `messages`, `temperature`, `stream`). | Respostas geradas e validadas por bibliotecas cliente oficias (ex: `openai-python`, `openai-node`). |
| **RF03** | **Exposição via Ngrok Tunnels** | Crítico | O serviço FastAPI deve ser acessível externamente utilizando um túnel reverso criado pela biblioteca `pyngrok`, gerando uma URL pública HTTPs estável para a sessão vigente. | URL pública (`https://<hash>.ngrok-free.app`) deve rotear para o FastAPI interno com latência adicionada < 200ms. |
| **RF04** | Endpoint de Status da Sessão | Média | O servidor deve expor `GET /v1/status` com informações da saúde do hardware da GPU, uso de VRAM e metadados da sessão e túnel ativo. | Resposta JSON com campos `session_id`, `model`, `vram_usage_mb` e `ngrok_url`. |
| **RF05** | Publicação do Estado no Drive | Alta | O *notebook* Colab deve, ao estabelecer o túnel Ngrok, escrever/atualizar um arquivo de *checkpoint* (ex: `ngrok_url.json`) no Google Drive centralizado. | Arquivo JSON sincronizado no Drive dentro de 10s após a criação bem-sucedida do túnel Ngrok. |
| **RF06** | Automação e Rotação via Browser | Crítico | O Orquestrador Local deve usar Playwright para realizar *login*, abrir o *notebook*, e clicar em "Run All" automaticamente. Caso detecte limite de cota excedida, deve trocar para a próxima conta do `accounts.json`. | Script navega sem erros de "Captcha" ou *timeout* e inicia as células no Colab dentro de 60s. |
| **RF07** | Proxy Local Transparente | Alta | O Orquestrador Local deve expor um proxy HTTP na porta `11434` (ou `8081`) que escuta as chamadas dos Agentes e as roteia automaticamente para a URL atual do Ngrok lida do Google Drive. | Requisição a `127.0.0.1:11434/v1/chat/completions` retorna a inferência vinda do Colab sem que o Agente conheça a URL do Ngrok. |
| **RF08** | Configuração Centralizada | Alta | Todo o mapeamento de contas, credenciais do Ngrok e configurações do *pool* deve ser lido de um arquivo de segurança estrito `~/.colab_infinity/accounts.json`. | Orquestrador inicializa corretamente após ler o JSON validando o *schema* e *tokens*. |

---

## 3. Requisitos Não Funcionais (RNF)

### 3.1 Desempenho e Capacidade (RNF-P)
- **RNF-P01:** O tempo total para inicialização a frio (*Cold Start* - login, boot Colab, instalação pip, carregamento dos pesos, túnel Ngrok) não deve exceder **8 minutos**.
- **RNF-P02:** O sistema deve manter o *Throughput* mínimo de inferência de **10 tokens por segundo** em modelos 7B (quantização 4-bits) utilizando hardware T4.

### 3.2 Resiliência e Disponibilidade (RNF-R)
- **RNF-R01:** O Orquestrador deve monitorar o túnel Ngrok com requisições de *heartbeat* (via `/v1/status`) a cada 30 segundos; em caso de 3 falhas consecutivas, deve disparar a rotação de conta.
- **RNF-R02:** O sistema deve tolerar desconexões efêmeras do Google Drive lidando com re-tentativas exponenciais (*Exponential Backoff*).

### 3.3 Segurança e Privacidade (RNF-S)
- **RNF-S01:** Credenciais de *tokens* Ngrok e *cookies* do Google devem estar armazenados com restrição de sistema de arquivos (permissão `chmod 600`).
- **RNF-S02:** Variáveis de ambiente sensíveis devem ser injetadas de forma dinâmica no ambiente do Colab via *scripting* do Playwright e nunca gravadas (hardcoded) nas células visíveis do *notebook*.
- **RNF-S03:** Toda comunicação via proxy do orquestrador ao túnel Ngrok é inerentemente coberta por criptografia HTTPS provida pelo certificado gerado do próprio Ngrok.

---

## 4. Requisitos de Hardware (RH)

- **Máquina do Orquestrador (Local/Servidor):**
  - Mínimo de 1 CPU Core e 1 GB RAM (Node/Python + Playwright Headless é leve).
  - Acesso contínuo à Internet, com portas abertas para HTTP de saída e Porta `11434` para tráfego local (`localhost`).
- **Google Colab (Remoto):**
  - Ambiente *Free Tier*: 1 GPU T4 (16 GB VRAM), 12.7 GB RAM de sistema.

---

## 5. Regras de Negócio e Restrições de Design

- **Restrição de Túneis:** Apenas 1 (um) túnel Ngrok simultâneo por *token* de autenticação, em respeito à camada gratuita (*Free Tier*) do Ngrok. Cada conta do Google cadastrada no sistema deve possuir seu próprio *token* exclusivo do Ngrok, evitando *Rate Limit Errors*.
- **Arquitetura *Drop-in*:** A aplicação não pode exigir que o ecossistema consumidor (ex: Ouroboros) implemente código específico para lidar com transições. Toda a mágica do *failover* reside no proxy Orquestrador.
- **Acesso Contínuo:** Como medida de *anti-idle*, o Orquestrador injeta eventos de atividade intermitentes no DOM do navegador Playwright aberto na página do Colab, prevenindo a interrupção prematura da máquina virtual.

---
