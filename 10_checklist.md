# Colab Infinity — Checklist de Deploy

**Versão:** 1.0.0  
**Data:** 2025-07-14  
**Status:** Aprovado  
**Referências:** `05_setup_guide.md`, `06_runbook.md`, `09_integration_guide.md`

---

> **Como usar este checklist:**  
> Execute cada item em ordem antes de iniciar a operação contínua do Colab Infinity.  
> Marque `[x]` quando o item estiver concluído e validado.  
> Itens marcados com ⚠️ são **bloqueadores** — o sistema não deve ser colocado em operação com qualquer deles pendente.  
> Itens marcados com ℹ️ são recomendados mas não bloqueadores.

---

## Seção 1 — Contas Google e Pool

### 1.1 Contas do Pool Operacional

- [ ] ⚠️ Mínimo de **3 contas Google** criadas com verificação por número de telefone
- [ ] ⚠️ Cada conta do pool foi acessada manualmente no [Google Colab](https://colab.research.google.com) e aceitou os Termos de Serviço
- [ ] ⚠️ Cada conta do pool tem GPU T4 disponível (verificado via `nvidia-smi` em célula de teste)
- [ ] ⚠️ Contas foram criadas com intervalo de **pelo menos 24 horas** entre cada criação
- [ ] ℹ️ Contas foram criadas a partir de IPs/dispositivos diferentes (reduz risco de correlação pelo Google)
- [ ] ⚠️ Nenhuma conta do pool é a mesma que a conta armazém
- [ ] ℹ️ Todas as contas do pool têm nomes de usuário não sequenciais (ex.: não usar `conta1`, `conta2`)

### 1.2 Conta Armazém (Google Drive)

- [ ] ⚠️ Conta armazém criada e configurada como **dedicada exclusivamente** ao armazenamento de estado
- [ ] ⚠️ Pasta raiz `colab_infinity/` criada no Google Drive da conta armazém
- [ ] ⚠️ Subpastas criadas manualmente: `checkpoints/`, `pool_state/`, `notebooks/`, `config/`, `logs/`
- [ ] ⚠️ ID da pasta `colab_infinity/` anotado (copiado da URL do Drive)
- [ ] ⚠️ Projeto `colab-infinity-storage` criado no [Google Cloud Console](https://console.cloud.google.com) com a conta armazém
- [ ] ⚠️ **Google Drive API** habilitada no projeto do Cloud Console
- [ ] ⚠️ Credenciais OAuth2 ("App para computador") criadas e arquivo JSON baixado
- [ ] ⚠️ Arquivo de credenciais salvo em `~/.colab_infinity/drive_credentials.json` com permissão `chmod 600`
- [ ] ⚠️ Fluxo de autorização OAuth2 concluído (token gerado em `~/.colab_infinity/drive_token.json`)
- [ ] ⚠️ Acesso ao Drive verificado com sucesso:
  ```
  python3 -m colab_infinity.cli drive health
  # Resultado esperado: Drive API: OK
  ```

---

## Seção 2 — ngrok

- [ ] ⚠️ **1 conta ngrok** criada para cada conta do pool operacional (não para a conta armazém)
- [ ] ⚠️ Token de autenticação de cada conta ngrok copiado do [dashboard.ngrok.com](https://dashboard.ngrok.com)
- [ ] ⚠️ Cada token ngrok adicionado ao campo `ngrok_token` correspondente em `~/.colab_infinity/accounts.json`
- [ ] ⚠️ Pelo menos 1 token ngrok testado localmente com sucesso:
  ```
  # Teste: tunnel cria URL pública e responde a request externo
  ```
- [ ] ℹ️ Tokens ngrok de contas diferentes estão em linhas separadas no `accounts.json` (não confundir tokens entre contas)

---

## Seção 3 — Arquivos de Configuração Local

- [ ] ⚠️ Diretório `~/.colab_infinity/` criado com permissão `chmod 700`
- [ ] ⚠️ Arquivo `~/.colab_infinity/accounts.json` criado com a estrutura correta:
  - Campo `warehouse` preenchido com e-mail e `drive_folder_name`
  - Array `pool` com todas as contas operacionais, índices únicos e tokens ngrok
- [ ] ⚠️ Arquivo `~/.colab_infinity/accounts.json` com permissão `chmod 600`
- [ ] ⚠️ Arquivo `~/.colab_infinity/colab_infinity_config.yaml` criado a partir do exemplo
- [ ] ⚠️ Campo `colab.notebook_id` preenchido com o ID do `colab_server.ipynb` no Drive
- [ ] ⚠️ Campo `drive.warehouse_folder_id` preenchido com o ID da pasta `colab_infinity/` no Drive
- [ ] ⚠️ Campo `proxy.port` definido como `11434` (porta padrão)
- [ ] ⚠️ Arquivo `~/.colab_infinity/colab_infinity_config.yaml` com permissão `chmod 600`
- [ ] ⚠️ Arquivo `.gitignore` do repositório inclui `~/.colab_infinity/` e `*.json` sensíveis
- [ ] ⚠️ Nenhum arquivo com credenciais está rastreado pelo git:
  ```bash
  git status | grep -E "accounts.json|drive_token|drive_credentials"
  # Resultado esperado: nenhuma saída (arquivos não rastreados)
  ```

---

## Seção 4 — Notebook Colab (`colab_server.ipynb`)

- [ ] ⚠️ Notebook `colab_server.ipynb` criado com todas as 7 células (ver Setup Guide Passo 4)
- [ ] ⚠️ **Célula 3** editada com os valores corretos:
  - `MODEL_ID` correto para o modelo desejado
  - `NGROK_TOKEN` substituído pelo token real da conta primária (índice 0)
  - `WAREHOUSE_FOLDER_ID` preenchido com o ID da pasta armazém
- [ ] ⚠️ Notebook testado manualmente em execução completa (Células 1 a 6) na conta primária
- [ ] ⚠️ Célula 6 imprimiu com sucesso: `"Colab Infinity ativo!"` com URL ngrok
- [ ] ⚠️ `ngrok_url.json` apareceu na pasta `colab_infinity/pool_state/` do Drive após execução
- [ ] ⚠️ Notebook salvo no Drive da conta armazém em `colab_infinity/notebooks/colab_server.ipynb`
- [ ] ⚠️ ID do notebook anotado (URL do Colab) e inserido em `colab_infinity_config.yaml`
- [ ] ℹ️ Notebook testado em pelo menos mais 1 conta do pool (conta de índice 1)
- [ ] ℹ️ Checkpoint automático verificado no Drive após 5 minutos de execução do notebook

---

## Seção 5 — Instalação Local

- [ ] ⚠️ Python 3.10+ instalado na máquina local: `python3 --version`
- [ ] ⚠️ Repositório clonado: `git clone https://github.com/RenyEnnos/colab-infinity.git`
- [ ] ⚠️ Ambiente virtual criado: `python3 -m venv .venv`
- [ ] ⚠️ Dependências instaladas: `pip install -r requirements.txt` (sem erros)
- [ ] ⚠️ Bun instalado (para o Ouroboros Runtime): `bun --version`
- [ ] ⚠️ `curl` disponível e funcional: `curl --version`

---

## Seção 6 — Orquestrador

- [ ] ⚠️ Configuração validada sem erros:
  ```bash
  python3 -m colab_infinity.cli validate-config
  # Resultado esperado: ✓ Configuração validada com sucesso.
  ```
- [ ] ⚠️ Pool inicializado no Drive:
  ```bash
  python3 -m colab_infinity.cli init-pool
  # Resultado esperado: Pool inicializado com N contas.
  ```
- [ ] ⚠️ `pool_state.json` visível na pasta `colab_infinity/pool_state/` do Drive
- [ ] ⚠️ Orquestrador iniciado sem erros:
  ```bash
  python3 -m colab_infinity.orchestrator --config ~/.colab_infinity/colab_infinity_config.yaml
  ```
- [ ] ⚠️ Proxy local respondendo em `http://127.0.0.1:11434`:
  ```bash
  curl -s http://127.0.0.1:11434/health | python3 -c \
    "import sys,json; d=json.load(sys.stdin); print(d['status'])"
  # Resultado esperado: ok
  ```
- [ ] ⚠️ Modelo carregado (`model.loaded: true`) verificado no health check
- [ ] ⚠️ GPU ativa (`runtime.device: "cuda"`) verificada no health check
- [ ] ℹ️ Orquestrador configurado como serviço systemd ou supervisord para reinicialização automática

---

## Seção 7 — Testes de Inferência

- [ ] ⚠️ Teste de inferência síncrono bem-sucedido:
  ```bash
  curl -s http://127.0.0.1:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"messages":[{"role":"user","content":"Responda: FUNCIONAL"}],"max_tokens":10}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['choices'][0]['message']['content'])"
  ```
- [ ] ⚠️ Teste de streaming funcionando (chunks SSE recebidos corretamente)
- [ ] ⚠️ Endpoint `/v1/status` retorna estrutura JSON completa com todos os campos
- [ ] ⚠️ Endpoint `POST /v1/checkpoint` retorna `"status": "saved"` e arquivo aparece no Drive
- [ ] ⚠️ Compatibilidade com `openai` Python SDK verificada:
  ```python
  from openai import OpenAI
  client = OpenAI(base_url="http://127.0.0.1:11434/v1", api_key="dummy")
  resp = client.chat.completions.create(
      model="mistralai/Mistral-7B-Instruct-v0.2",
      messages=[{"role": "user", "content": "ping"}],
      max_tokens=5,
  )
  assert resp.choices[0].message.content  # não vazio
  print("SDK: OK")
  ```
- [ ] ℹ️ Smoke test completo executado e todos os itens passaram:
  ```bash
  bash tests/smoke_test.sh
  # Resultado esperado: N passou, 0 falhou
  ```

---

## Seção 8 — Integração com Agentes Consumidores

### 8.1 Ouroboros Runtime

- [ ] ⚠️ Arquivo `.env` do Ouroboros Runtime contém as variáveis corretas:
  - `LLM_BASE_URL=http://127.0.0.1:11434/v1`
  - `LLM_API_KEY=dummy` (ou key real se `require_auth: true`)
  - `LLM_MODEL=mistralai/Mistral-7B-Instruct-v0.2`
  - `LLM_TIMEOUT_MS=120000`
  - `LLM_MAX_RETRIES=3`
- [ ] ⚠️ Ouroboros Runtime inicia sem erros de conexão ao LLM:
  ```bash
  bun run start
  # Verificar logs: sem "ECONNREFUSED" ou "LLM unavailable"
  ```
- [ ] ⚠️ Uma Wave simples completa com sucesso passando pelo Gate 1 do Protocolo Anti-Vibe
- [ ] ℹ️ Temperatura configurada por agente do Conselho (Architect: 0.4, Guardian: 0.2, Kinetic: 0.7)
- [ ] ℹ️ Monitoramento de `quota_remaining_minutes` implementado no Daemon para alertas preventivos

### 8.2 Hermes Agent

- [ ] ⚠️ Hermes Agent configurado com `base_url: "http://127.0.0.1:11434/v1"` em `~/.config/hermes-agent/config.yaml`
- [ ] ⚠️ Teste de conexão bem-sucedido: `hermes-agent test-connection`
- [ ] ℹ️ Conversa de 3+ turnos testada sem erros de timeout ou parsing

### 8.3 OpenClaw

- [ ] ℹ️ Variáveis de ambiente configuradas: `OPENAI_API_BASE=http://127.0.0.1:11434/v1`
- [ ] ℹ️ Teste de chamada simples confirmado sem erros

---

## Seção 9 — Segurança e Boas Práticas

- [ ] ⚠️ Nenhuma credencial sensível (token ngrok, API key, refresh token) está hardcoded no notebook Colab
- [ ] ⚠️ Nenhum arquivo sensível rastreado pelo git (verificar `git status` e `git log --all`)
- [ ] ⚠️ Proxy local escuta apenas em `127.0.0.1:11434` (nunca em `0.0.0.0`):
  ```bash
  ss -tlnp | grep 11434 | grep -v "127.0.0.1"
  # Resultado esperado: nenhuma saída (bind apenas em loopback)
  ```
- [ ] ⚠️ Arquivo `~/.colab_infinity/accounts.json` com permissão `600`:
  ```bash
  ls -la ~/.colab_infinity/accounts.json | awk '{print $1}'
  # Resultado esperado: -rw-------
  ```
- [ ] ⚠️ Arquivo `~/.colab_infinity/drive_token.json` com permissão `600`
- [ ] ℹ️ API Key habilitada (`require_auth: true`) se o ambiente tiver acesso compartilhado
- [ ] ℹ️ Logs verificados para ausência de tokens ou credenciais em texto plano:
  ```bash
  grep -Ei "ngrok_token|refresh_token|client_secret|api_key=" ~/.colab_infinity/logs/orchestrator.log
  # Resultado esperado: nenhuma saída
  ```

---

## Seção 10 — Tolerância a Falhas e Recuperação

- [ ] ⚠️ Pelo menos 1 checkpoint salvo e validado no Drive:
  ```bash
  python3 -m colab_infinity.cli checkpoint list --limit 3
  # Resultado esperado: ao menos 1 arquivo ci_ckpt_*.json listado
  ```
- [ ] ⚠️ Restauração de checkpoint testada com sucesso:
  ```bash
  python3 -m colab_infinity.cli checkpoint inspect --latest
  # Resultado esperado: JSON válido com schema_version, session, pool
  ```
- [ ] ⚠️ Troca manual de conta testada pelo menos 1 vez:
  ```bash
  python3 -m colab_infinity.cli switch-account --target-index 1
  # Resultado esperado: troca concluída em < 8 minutos
  ```
- [ ] ⚠️ Estado do pool verificado após a troca:
  ```bash
  python3 -m colab_infinity.cli pool list
  # Resultado esperado: conta 0 com status "exhausted", conta 1 com "active"
  ```
- [ ] ℹ️ Teste de falha simulada realizado (mock de 3 health checks falhando → troca automática)
- [ ] ℹ️ Procedimento de recuperação de desastre documentado e testado ao menos 1 vez

---

## Seção 11 — Monitoramento e Observabilidade

- [ ] ⚠️ Logs do orquestrador sendo escritos corretamente:
  ```bash
  tail -5 ~/.colab_infinity/logs/orchestrator.log | python3 -m json.tool
  # Resultado esperado: JSON Lines válidos com timestamp, level, event
  ```
- [ ] ⚠️ Métricas disponíveis em `GET /v1/status` (uptime, requests_served, tokens_generated)
- [ ] ℹ️ Webhook de notificação configurado (Slack/Discord) se o ambiente exigir alertas automáticos
- [ ] ℹ️ Script de smoke test (`bash tests/smoke_test.sh`) disponível e executável
- [ ] ℹ️ Alarme manual configurado para verificar `quota_remaining_minutes` diariamente

---

## Seção 12 — Checklist Final Pré-Operação

Execute esta verificação final imediatamente antes de colocar o sistema em operação contínua:

- [ ] ⚠️ Orquestrador está em estado `ACTIVE` (não `STARTING`, `HALTED` ou `IDLE`)
- [ ] ⚠️ `GET /health` retorna `"status": "ok"` com `"model.loaded": true` e `"runtime.device": "cuda"`
- [ ] ⚠️ Pelo menos **2 contas disponíveis** no pool (pool com apenas 1 conta não tem redundância):
  ```bash
  python3 -m colab_infinity.cli pool list
  # Verificar coluna "Status": pelo menos 2 com "available" ou 1 "active" + 1 "available"
  ```
- [ ] ⚠️ Drive armazém acessível: `python3 -m colab_infinity.cli drive health`
- [ ] ⚠️ Último checkpoint salvo há menos de 10 minutos
- [ ] ⚠️ Nenhum erro de nível `CRITICAL` ou `ERROR` nos logs das últimas 2 horas:
  ```bash
  grep '"level":"critical"\|"level":"error"' ~/.colab_infinity/logs/orchestrator.log \
    | tail -5
  # Resultado ideal: nenhuma saída recente
  ```
- [ ] ⚠️ Ouroboros Runtime (ou agente consumidor) conectado e respondendo ao health check
- [ ] ⚠️ Sessão Colab ativa com pelo menos 60 minutos de quota restante:
  ```bash
  curl -s http://127.0.0.1:11434/health | python3 -c \
    "import sys,json; d=json.load(sys.stdin); \
     m=d['session']['estimated_quota_remaining_minutes']; \
     print(f'Quota restante: {m} min', '✓' if m >= 60 else '⚠️  BAIXO')"
  ```

---

## Resumo do Status do Deploy

Preencha após concluir todas as verificações:

| Seção                                   | Itens Obrigatórios | Concluídos | Status    |
|-----------------------------------------|-------------------|------------|-----------|
| 1 — Contas Google e Pool                | 8                 | ___        | ___       |
| 2 — ngrok                               | 4                 | ___        | ___       |
| 3 — Arquivos de Configuração Local      | 9                 | ___        | ___       |
| 4 — Notebook Colab                      | 7                 | ___        | ___       |
| 5 — Instalação Local                    | 6                 | ___        | ___       |
| 6 — Orquestrador                        | 7                 | ___        | ___       |
| 7 — Testes de Inferência                | 5                 | ___        | ___       |
| 8 — Integração com Agentes              | 4                 | ___        | ___       |
| 9 — Segurança                           | 5                 | ___        | ___       |
| 10 — Tolerância a Falhas                | 4                 | ___        | ___       |
| 11 — Monitoramento                      | 2                 | ___        | ___       |
| 12 — Checklist Final                    | 8                 | ___        | ___       |
| **TOTAL**                               | **69**            | **___**    | **___**   |

**Data de conclusão do deploy:** `____-__-__`  
**Responsável técnico:** `______________________`  
**Versão do Colab Infinity:** `1.0.0`  
**Assinatura:** `______________________`

---

> ✅ **GO** para operação contínua: todos os 69 itens obrigatórios (⚠️) marcados como concluídos.  
> ❌ **NO-GO**: qualquer item obrigatório pendente. Não iniciar operação contínua até resolver.

---

*Documento gerado para o projeto Colab Infinity. Versão 1.0.0 — Julho 2025.*