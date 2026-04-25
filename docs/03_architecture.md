# Documento de Arquitetura de Software (SAD)

**Projeto:** Colab Infinity
**Versão:** 1.0.0
**Status:** Aprovado
**Referências:** `01_project_charter.md`, `02_srs.md`

---

## 1. Visão Arquitetural

A arquitetura do **Colab Infinity** adota um modelo híbrido de **Proxy/Orquestrador Local** operando em conjunto com um **Serviço de Nuvem Efêmero (Google Colab)**, interligados assincronamente por meio do **Google Drive (Armazenamento de Estado)** e dinamicamente mapeados através de **Ngrok Tunnels (Rede)**.

O objetivo do *design* arquitetural é blindar completamente os clientes (Agentes Autônomos) da instabilidade inerente do Google Colab Gratuito. O ecossistema de clientes apenas conversa com uma interface de rede local (um proxy HTTP), que por sua vez roteia e resolve toda a complexidade de instanciamento, rotação e recuperação de falhas no *backend*.

## 2. Diagrama de Contexto (Nível 1 - C4 Model)

```text
[Agentes Autônomos]
   (ex: Ouroboros)
          |
          | HTTP POST /v1/chat/completions
          | (Target: 127.0.0.1:11434)
          V
+-----------------------+                    +-------------------------+
|                       |  Mapeia URL/Status |                         |
|   Orquestrador Local  |<==================>|      Google Drive       |
|    (Proxy + Automação)|                    |  (Checkpoint ngrok_url) |
|                       |                    |                         |
+-----------------------+                    +-------------------------+
          |
          | Tráfego HTTP(s) Roteado dinamicamente
          V
   [ Ngrok Cloud ]
   (https://<hash>.ngrok-free.app)
          |
          | Túnel Reverso
          V
+-----------------------+
|  Google Colab VM (T4) |
|                       |
|   +---------------+   |
|   | FastAPI /v1   |   |
|   +---------------+   |
|   | Transformers  |   |
|   | 4-bit Quantiz.|   |
+-----------------------+
```

## 3. Componentes Principais (Nível 2)

### 3.1 Orquestrador Local (Daemon Python/Node)
O componente central em execução no servidor do usuário. Desempenha duas funções primárias que operam em paralelo:
- **Proxy HTTP Transparente:** Um servidor web reverso (`localhost:11434`) que escuta as requisições compatíveis com a API da OpenAI. Ele lê o arquivo do *Google Drive* local para saber qual a URL Ngrok ativa e repassa a requisição HTTP. Implementa *retries* automáticos se houver um timeout.
- **Gerenciador de Ciclo de Vida (Lifecycle Manager):** Um robô em *background* construído usando **Playwright/Selenium Headless**. Ele possui as credenciais das N contas. Fica rodando loops de *heartbeat*. Ao detectar falha no Colab, o robô injeta cookies, loga em uma nova conta no Google, abre a URL do *notebook* Colab base, roda o script e aguarda até o novo arquivo Ngrok aparecer no Drive.

### 3.2 Backend Efêmero: Colab + FastAPI
Código em formato de *Jupyter Notebook* (`.ipynb`) responsável pela interface com o hardware:
- Inicializa as dependências via `pip`.
- Sobe um servidor `FastAPI` na porta 8000 usando `uvicorn`.
- Instancia o pacote `pyngrok` ligando um túnel HTTPS seguro que mapeia diretamente para a porta 8000 do FastAPI.
- Carrega o modelo de ML na VRAM através de bibliotecas como `transformers` e `bitsandbytes`.

### 3.3 State Manager: Google Drive (Conta Armazém)
Devido ao isolamento da infraestrutura do Colab, o mecanismo mais confiável e não-bloqueante para comunicação de metadados com o mundo exterior é o Drive.
- O *notebook* monta o Drive nativamente (`google.colab.drive.mount`).
- Grava um JSON contendo `{"ngrok_url": "https://xyz.ngrok-free.app", "timestamp": ...}`.
- O Orquestrador Local tem este Drive sincronizado (ou consulta sua API) para ler a URL.

## 4. Decisões Arquiteturais e Trade-offs

- **Túneis: Ngrok vs Cloudflare Tunnels vs LocalTunnel:**
  *Decisão:* Adotar **Ngrok** gerido via `pyngrok` diretamente do Python.
  *Motivo:* Menor atrito de instalação em ambientes não-root do Colab, alta estabilidade, e possibilidade nativa de controle programático em Python pelo *notebook*. Remove a necessidade de binários separados que o Cloudflare precisava, embora introduza a restrição estrita de 1 túnel por *token* no plano grátis (resolvido por mapear um token por conta do pool).

- **Sincronização de Estado: Google Drive vs Banco de Dados Externo:**
  *Decisão:* Usar o **Google Drive** montado na VM.
  *Motivo:* Não requer configuração de banco de dados externo (Redis, Firebase, Supabase). Não consome limites de API externa. É nativo, gratuito e possui alta taxa de tolerância em integrações da própria Google, reduzindo dependências do projeto.

- **Automação do Browser: Playwright vs Módulos de Requisição:**
  *Decisão:* **Playwright Headless**.
  *Motivo:* As telas de autenticação e verificação do Google (Google OAuth/Colab) fazem uso massivo de detecção de bots e carregamento dinâmico via JS. Requisições POST convencionais iriam falhar. O Playwright simula de fato um usuário real e permite injeção nativa de *Cookies* de sessão gravados no *filesystem*, evadindo bloqueios.

## 5. Fluxo de Execução Crítico (Rotação e Failover)

1. **Falha Detectada:** O Orquestrador faz um ping em `<ngrok_url>/v1/status` e obtém falha (Timeout ou 502 Bad Gateway).
2. **Locking do Proxy:** O Proxy Local suspende a entrada de novas requisições dos agentes, botando as requisições pendentes na *Queue* (fila de espera).
3. **Instanciamento de Nova Sessão:** O *Lifecycle Manager* acorda, faz *logout* do perfil corrompido/expirado, seleciona a próxima Conta do Google válida na lista do JSON e loga via Playwright injetando cookies armazenados.
4. **Boot do Colab:** O *notebook* da nova conta é executado. O pip baixa `pyngrok`, o modelo sobe e o novo túnel Ngrok é ativado.
5. **Atualização de Estado:** O Colab escreve a nova URL no *Google Drive* compartilhado.
6. **Desbloqueio:** O Orquestrador detecta a alteração no arquivo do Drive, atualiza a sua variável de rota do Proxy para a nova URL Ngrok e destrava a Fila, processando as requisições dos Agentes que estavam aguardando (garantindo transição *seamless* para os agentes).

---
